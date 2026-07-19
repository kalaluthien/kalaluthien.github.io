---
share: true
title: Android image decoding
categories:
  - Wiki
author: claude
type: concept
created: 2026-07-17
updated: 2026-07-17
sources:
  - "[[2026-07-17-how-android-apps-load-gallery-photos]]"
aliases:
  - BitmapFactory
  - ImageDecoder
  - Bitmap memory
---

# Android image decoding

Turning encoded bytes into pixels that fit a frame budget. The decode half of
"load the user's photos"; enumeration is [Android MediaStore](/wiki/android-mediastore/).

## One codec, two API surfaces

`BitmapFactory` (API 1) and `ImageDecoder` (API 28) are **backed by the same Skia
decoder**. BitmapFactory's four native entry points funnel into one `doDecode()` that
builds `SkCodec::MakeFromStream` → `SkAndroidCodec::MakeFromCodec`; ImageDecoder's JNI
builds the same `SkAndroidCodec`. Same libjpeg-turbo, libpng, libwebp. The differences
are elsewhere:

| | `BitmapFactory` | `ImageDecoder` |
|---|---|---|
| Input | stream/fd/byte[]/asset | `Source` (fd, file, ByteBuffer) |
| Scaling | `inSampleSize` + `inDensity`/`inTargetDensity` trick | `setTargetSize(w,h)` — arbitrary, fused |
| EXIF orientation | **ignored** | **applied natively** (except raw) |
| Default allocator | software ARGB_8888 | **hardware where possible** |
| Bitmap reuse | `inBitmap` | **none — every decode allocates** |
| Corrupt input | returns `null` | throws `DecodeException` |

Measured (sm7250, Android 14, 10 runs): ImageDecoder is **5–15% faster**, and the gain is
IO, not decode — it takes an fd directly, while `decodeStream(InputStream)` calls back into
Java for every buffer fill. ~1 MP JPEG full decode: 53 ms vs 49 ms; sampled to 640×480:
37 ms vs 30 ms; 4 MP PNG: 253 ms vs 233 ms.

**No `inBitmap` on ImageDecoder** is why pooling libraries kept BitmapFactory.

## Why `inSampleSize` is nearly free

libjpeg-turbo supports scaling by M/8 (1/8, 1/4, 3/8, 1/2, 5/8, 3/4, 7/8, 1/1) implemented
**inside the IDCT** — at 1/8 it computes only the DC coefficient per 8×8 block; 1/2 and 1/4
use reduced-size IDCTs. Full-resolution pixels are never reconstructed. Skia exposes this via
`SkAndroidCodec::getSampledDimensions`; `inSampleSize` maps straight through to
`codecOptions.fSampleSize`.

**Anything the codec can't do natively costs a second full pass**: BitmapFactory redraws
through an `SkCanvas` with bilinear filtering into a second allocation. ImageDecoder fuses
subsampled decode + one bilinear scale into a single call, which is its real ergonomic win.

Corollary: **decode cost scales with output pixels, not file size** — as long as the codec
can subsample. HEIC can't (see below). PNG can't fractionally either.

## The `inDensity`/`inTargetDensity` trick

Power-of-two sampling gets within 2× of the target. Both Glide and Coil close the gap by
abusing density scaling, which exists for drawable resources and accepts any ratio:
`scale = (float) targetDensity / density`, then `round(dim * scale)`.

Coil's use is direct: `inDensity = Int.MAX_VALUE; inTargetDensity = (Int.MAX_VALUE * scale).roundToInt()`
— largest denominator, finest precision. Glide additionally *measures and cancels* the
float-division error (`adjustTargetDensityForError`): compute the naive target density,
measure the resulting error, pre-distort to compensate. Both must repair the bitmap's density
metadata afterwards (`setDensity(displayMetrics.densityDpi)`).

Glide also **reimplements the framework's per-format rounding** to predict output size (JPEG:
libjpeg ceil up to 8 then Skia floor; PNG floor; WebP round on N+, floor before) — determined
empirically, "tested on emulators for Android versions 15-26". For unknown formats it runs a
*third* bounds decode with `inSampleSize` set. That is the tax for an API that never told you.

## `inBitmap` and the end of pooling

Contract: KitKat+ allows any mutable bitmap whose allocated byte count ≥ the result's. Failures
are indistinguishable from corrupt input — **any** error with `inBitmap != null` throws
`IllegalArgumentException`, including a too-small pooled bitmap; an immutable one is silently
ignored with a logcat warning. Glide's pattern: catch IAE, return the bitmap to the pool, null
`inBitmap`, retry without reuse.

The era ended when pixels moved off the Java heap:

- API ≤ 10: native memory. **API 11–25: the Dalvik/ART heap** (a decade of OOM-driven pooling).
  **API 26+: the native heap.**
- Glide: `BITMAP_POOL_TARGET_SCREENS = SDK < O ? 4 : 1`, and **0** on O+ low-RAM. Its comment:
  "On Android O+, we use HARDWARE for all reasonably sized images ... the Bitmap pool is much
  less important on O".
- Coil **deleted its pool in 2.0**; `inMutable = false` always.
- Fresco still pools.

## Hardware bitmaps

`Config.HARDWARE` (API 26) stores pixels only in graphics memory (gralloc), always immutable,
"optimal for cases, when the only operation with the bitmap is to draw it on a screen". Glide:
"½ the memory of other Bitmap configurations" (one copy instead of heap + GPU texture) and no
texture-upload jank. They appear under Graphics/gralloc in `dumpsys meminfo`, not heap counts.

The decode itself is not hardware: BitmapFactory decodes to a heap allocation, *then* uploads
("extra swizzling & upload to gralloc step"). Peak memory is unchanged; steady state improves.

**Sharp edges, all runtime throws:**

- `getPixel`/`getPixels`/`getColor`/`copyPixelsTo|FromBuffer` → `IllegalStateException`.
- Drawing into a **software Canvas** → `IllegalStateException` at draw time — this breaks
  view-hierarchy screenshots; use `PixelCopy`.
- mutable + HARDWARE → `IllegalArgumentException`.
- **`ImageDecoder.decodeBitmap` returns one by default** (`ALLOCATOR_DEFAULT`), so the first
  `getPixel` in a codebase discovers it.
- **EXIF rotation forces a software decode** — a hardware bitmap can't be redrawn. Glide vetoes
  the hardware config; Coil calls `config.toSoftware()`. Camera photos are exactly the affected set.

**File descriptors** are the systemic hazard: ~2 per hardware bitmap against a 1024 rlimit on
O/O_MR1 (32768 on 28+), causing "graphics errors, Binder errors, and a variety of crashes".
Both libraries count `/proc/self/fd`:

- **Glide**: P+ only (`HARDWARE_BITMAPS_SUPPORTED = SDK >= P`); check every 50 decodes (each costs
  1–2 ms); cap 20,000, reduced to **500** on named OnePlus P devices (b/139097735); blocked
  entirely pre-Q until the GL context exists (b/126573603#comment12), so cold-start images are
  software by design.
- **Coil**: FD counting on **26/27 only** (cap 800, every 30 decodes or 30 s); always allowed on
  28+. Denylist blocks **every Samsung `SM-` device on API 26** plus ~60 named devices on 26/27.

Glide's history rhymes: v4.9.0 allowed O with a 700-FD cap; v4.11.0 added a Samsung blocklist
disallowing hardware *exactly on API 26*; master dropped O entirely. Two libraries, two evidence
bases, one conclusion: **Android O's hardware bitmaps were not shippable.**

## EXIF orientation: no uniform rule

This is the trap, and it is an asymmetry rather than a bug:

- BitmapFactory ignores EXIF entirely — no orientation code in `doDecode`.
- ImageDecoder applies it natively (invisible from Java), via `SkCodec::getOrigin` folded into
  the output matrix — costing an extra full-size temp allocation and a draw.
- **Except for raw**, where it doesn't. Immich shipped a fix for portrait DNGs displayed
  sideways: "the jpeg/heic decoders rotate on their own, raw doesn't". So an app that uniformly
  rotates after ImageDecoder double-rotates JPEGs; one that never rotates shows raw sideways.
- **HEIF orientation lives in `irot`/`imir` container properties**, not EXIF. MIAF (ISO/IEC
  23000-22:2024) §7.3.10.1: EXIF transformations "shall be ignored by a MIAF renderer". Apps that
  read EXIF *and* let the platform rotate double-rotate HEICs whose EXIF disagrees with irot.

Library approaches: **Glide** hand-parses JPEG APP1/TIFF IFDs pre-decode (and reads *only* JPEG
and raw TIFF — not HEIF/PNG/WebP, which is a filed bug; fix is androidx `exifinterface` ≥ 1.2.0).
**Coil** uses androidx `ExifInterface` over a peeked stream that lies about `available()` (returns
1 GB so ExifInterface doesn't stop early); default strategy `RESPECT_PERFORMANCE` honors EXIF only
for jpeg/webp/heic/heif, because `RESPECT_ALL` buffers formats like PNG entirely into memory.
**Telegram** doesn't rotate at all — it ships orientation alongside the bitmap and applies it at
**draw time**, avoiding the second full-size allocation entirely. It also projects MediaStore's
`ORIENTATION` **column**, so its grid never parses EXIF per file.

Rotation elsewhere is `Matrix` + `Bitmap.createBitmap(...)` — a second allocation. Glide used to
draw into a pooled bitmap and deliberately stopped: "BitmapPool doesn't preserve gainmaps and
color space". Ultra HDR broke the optimization.

## HEIC is a different machine

JPEG/PNG/WebP decode in-process via Skia. **HEIC goes through the media framework**:
`HeifDecoderImpl` constructs a `MediaMetadataRetriever`, sets the data source to a `BnDataSource`
binder object proxying the app's stream (64 KB transfer buffer, 4→64 MB rolling cache), and calls
`getImageAtIndex`. Results arrive as a `VideoFrame` in binder shared memory.

Every HEIC decode = binder round-trips + a MediaCodec HEVC session + YUV→RGB conversion:

1. A **fixed latency floor** per image (session setup dominates small images).
2. **Contention** — hardware codec instances are a limited system resource.
3. **Sample size doesn't help.** HEVC tiles must be fully decoded before scaling; there is no
   scaled-IDCT analogue. Glide's bug report: "It doesn't matter if I have default settings or
   `.override(50, 50)` ... the load speed stays the same for 2-3 seconds per image."

Estimate: **~100–500 ms/image vs ~20–60 ms for an equivalently downsampled JPEG** on midrange
hardware. And memory doesn't improve — a 12 MP HEIC still yields the same 46.5 MB bitmap.

**If you control capture, write JPEG.** The storage saving is repaid in latency on every view.

## Bitmap memory arithmetic

`width × height × bytes-per-pixel`: 1 (ALPHA_8), 2 (RGB_565), 4 (ARGB_8888, RGBA_1010102),
8 (RGBA_F16).

| Source | ARGB_8888 |
|---|---|
| 12 MP 4:3 (4032×3024) | **46.5 MB** |
| 50 MP (8160×6120) | 190.5 MB |
| 200 MP (16320×12240) | **762 MB** |
| FHD+ screenshot (1080×2340) | 9.6 MB |
| Full-screen decode target (1080×810) | 3.3 MB |
| Grid thumb (512×384) | 768 KB |

`inSampleSize` ladder for 12 MP: 1 → 46.5 MB, 2 → 11.6 MB, 4 → 2.9 MB, 8 → 744 KB, 16 → 186 KB.

**One decoded full-resolution photo costs more than an entire 40-cell thumbnail grid** (10 MB at
256², 30 MB at 512×384). That single fact is the whole argument for downsampling.

`getByteCount()` "can no longer be used to determine memory usage" as of KitKat — use
`getAllocationByteCount()`, which is larger when a bitmap was reused for a smaller decode.

## Size ceilings

- **Canvas byte limit**: pre-P `MAX_BITMAP_SIZE = 100 MB`; Android 13 made it
  `max(ro.hwui.max_texture_allocation_size, 100 MB)`; current main raises the default to **150 MB
  and exempts HARDWARE bitmaps**. At 100 MB the ceiling is ~26.2 MP ARGB_8888 — a 64 MP shot is
  256.6 MB and always threw `Canvas: trying to draw too large bitmap`.
- **GPU texture limit** is separate and lower: commonly 4096/8192/16384 px per dimension. A
  software bitmap over it logs "Bitmap too large to be uploaded into a texture" and simply doesn't
  render. Panoramas hit this long before any byte limit. 50 MP (8160 px) exceeds 4096; 200 MP
  (16320 px) exceeds 8192.

## Region decoding and tiling

`BitmapRegionDecoder` + a tile pyramid is the answer for images that can't be one texture;
`subsampling-scale-image-view` is the reference implementation (last commit 2020 — the problem
stopped changing).

- **The platform serializes it**: `private final Object mNativeLock` — "only one decode can occur
  at a time". One decoder instance = serialized tiles. SSIV ships an experimental *pool* of
  decoders for real parallelism (hard cap 4).
- SSIV's read-write lock exists for an older reason: "Before SDK 21, BitmapRegionDecoder was not
  synchronized internally. Any attempt to decode regions from multiple threads with one decoder
  instance causes a segfault."
- Its sizing inverts Glide's: take the **smaller** ratio "to guarantee a final image with both
  dimensions larger than or equal to the requested". The base layer is decoded one level *sharper*
  than needed and **never discarded** on zoom-out, so there is always something to draw.
- **Format support moved**: JPEG and PNG only through Android 11; **WebP and HEIF from Android 12
  / API 31**. Tiling a HEIC on 11 and below is not an option.

## Concurrency

- **Glide has no global decode lock on modern devices.** `BITMAP_DRAWABLE_LOCK` looks like one but
  is a real `ReentrantLock` *only* for a hardcoded list of 2014-era Moto models; elsewhere it's a
  `NoLock` stub. Decodes run fully concurrently, bounded by thread pools.
- **Coil caps parallelism at 4** (`DEFAULT_MAX_PARALLELISM`), with the *same semaphore shared*
  between `BitmapFactoryDecoder` and `StaticImageDecoder`.

## Thumbnails

- **`ContentResolver.loadThumbnail` (Q+)** is a cache-warming API, not a lookup. Cache hit: open
  `<primary>/Pictures/.thumbnails/<id>.jpg`. **Cache miss: full source decode + JPEG q90 encode +
  rename, before you get the fd — then you still decode the result.** Concurrent racers each
  regenerate independently (no coalescing). Cold, it is the *most* expensive call available —
  strictly more work than decoding the original yourself at `inSampleSize = 8`.
- **The size is not yours**: `thumbSize = min(screenWidth, screenHeight) / 2` — ~540 px on a 1080p
  phone, JPEG q90, regardless of what you requested.
- **Invalidation is whole-directory**: a `.thumbnails/.database_uuid` mismatch deletes every
  thumbnail. A MediaProvider DB rebuild silently converts the entire gallery to cache-miss.
- **The legacy API is the modern API**: `Images.Thumbnails.getThumbnail(..., MINI_KIND, ...)`
  converts kind→size and calls `loadThumbnail` on Q+. Constants: `MICRO_KIND` 96×96, `MINI_KIND`
  512×384, `FULL_SCREEN_KIND` 1024×**786** — a typo frozen into public API forever.
- **`ThumbnailUtils.createImageThumbnail`** (what MediaProvider calls; public API) tries in order:
  HEIF embedded thumbnail via MMR → **`exif.getThumbnailBytes()`**, decoded directly, never
  touching the main image → full-file `ImageDecoder.decodeBitmap`. The EXIF fast path is the
  difference between a grid that scrolls and one that doesn't.
- **Libraries mostly ignore all of it.** Glide and Coil point at the content URI and let
  `inSampleSize` do the work — a 1/8 JPEG decode beats a binder round-trip plus a cold thumbnail
  generation, and doesn't depend on a warm cache.

## Formats in the wild

- **Binning means the 200 MP phone ships 12 MP files by default.** Galaxy Ultra's 16320×12240 is
  opt-in; Pixel 8's 50 MP sensor bins 4-to-1 to 4080×3072 with no full-res option.
- **Samsung camera defaults to JPEG** (HEIF is an opt-in toggle that updates sometimes flip on);
  iPhone imports are HEIC; Pixel is JPEG.
- **"Screenshots are PNG" is only true on Pixels.** Samsung One UI defaults to **JPG**. This
  matters — PNG has no scaled-decode path and costs ~4× a JPEG of equal pixels.
- **Ultra HDR costs a second decode**: the gainmap is extracted automatically via a second codec
  pass at the same sample size. Default-on for Pixel 8+/Android 14. Still `.jpg`.
- **Motion photos are JPEG + appended MP4** (Google formalized the format; Samsung uses a
  `MotionPhoto_Data` marker). Decoding works — decoders stop at the JPEG EOI — but files are
  video-dominated (~8–25 MB), so any heuristic reasoning from file length is wrong. Telegram
  budgets XMP parsing to the first 15 photos.
- Telegram's own send pipeline (verified from source): 1280 px @ JPEG q87 standard, 2560 px @ q99
  high-quality. WhatsApp re-encodes to ~1600 px.

## Practical shape for a read-only own-photos gallery

Use **Coil with defaults**: on API 29+ that is `StaticImageDecoder` → `ImageDecoder` (fused
scaling, native EXIF, hardware bitmaps where allowed); below 29 it falls back to BitmapFactory with
the density trick and `ExifUtils`. The correct amount of custom decode code is **zero** — every
hand-rolled pipeline (Telegram's, Fresco's) exists to serve a constraint a simple gallery doesn't
have.

Don't call `loadThumbnail`. Don't region-decode unless full-res capture ships. Capture JPEG, not
HEIC. Read EXIF for a detail panel via `ExifInterface` over `openInputStream` — and don't let that
leak into the decode path, or you'll re-apply a rotation the platform already applied.
