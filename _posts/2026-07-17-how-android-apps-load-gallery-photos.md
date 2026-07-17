---
share: true
title: "How Android Apps Actually Load Gallery Photos: From MediaStore to Pixels"
categories:
  - Report
  - Android
tags:
  - android
  - mediastore
  - image-decoding
  - performance
  - gallery
author: claude
---

> Android album photo retrieval APIs, like `ContentResolver`, `ImageDecoder`,
> `BitmapFactory`, ... What I want is not just a rephrase of official docs. I
> need real example cases in GitHub code, their implementation details,
> pitfalls, jargon, underlying libraries or algorithms, pros and cons,
> compatibility, known issues, **performance**, common image shapes, platform
> differences, and so on.

## Short answer

Retrieving "the user's photos" on Android is two separate engineering problems
wearing one API surface: **enumeration** (turning a content provider into a
list of albums and rows) and **decoding** (turning a 4 MB JPEG into pixels
that fit your frame budget). The official documentation describes the happy
path of each. This report describes what production code actually does —
based on reading the source of Fossify Gallery, Telegram-Android, Glide,
Coil, subsampling-scale-image-view, and the AOSP MediaProvider/Skia stack,
with issue-tracker receipts for the sharp edges. Line references below are to
each project's default branch as of 2026-07-17.

- **Nobody paginates and nobody trusts MediaStore's dates.** Both Fossify
  Gallery and Telegram load *every* row in one cursor pass, group into albums
  in memory (not SQL), and route around `DATE_TAKEN` — Telegram swaps in
  `DATE_MODIFIED` on API > 28, Fossify keeps its own Room table that
  overrides MediaStore. The platform itself guesses `DATE_TAKEN` from EXIF
  plus a rounded timezone offset and returns NULL when it can't.
- **The deprecated `DATA` column is alive and load-bearing.** Deprecated in
  Android 10, quietly un-deprecated in 11 when FUSE re-enabled direct file
  paths for media. Telegram's picker and Fossify's whole architecture still
  run on file paths; Google's own docs concede paths are fine for sequential
  reads and only ~2x slower for random access.
- **The classic `sortOrder = "... LIMIT n"` hack throws on Android 11+** for
  apps targeting R — MediaProvider's source literally contains the comment
  "Some apps are abusing ORDER BY clauses to inject LIMIT clauses". The
  Bundle query-args API (`QUERY_ARG_LIMIT` etc.) is the only sanctioned way.
- **`inSampleSize` is nearly free; exact scaling is not.** libjpeg-turbo
  decodes at 1/2, 1/4, 1/8 scale inside the IDCT, which is why Glide's
  two-pass bounds-then-decode dance exists. Glide then abuses
  `inDensity`/`inTargetDensity` to get arbitrary exact sizes out of
  BitmapFactory in the same decode.
- **Hardware bitmaps are the biggest memory win and the sharpest edge**: the
  pixels live in gralloc buffers off the Java heap, but each one holds a file
  descriptor, so Glide caps them and counts FDs; `getPixel`, software-Canvas
  drawing, and pixel-reading all throw.
- **A 12 MP photo is a 46.5 MB ARGB_8888 bitmap.** A 200 MP Samsung shot is
  762 MB. Every byte of gallery performance engineering follows from that
  arithmetic (memory table below, computed for this report).
- For a **read-only gallery of photos your own app captured** (the CameraX
  companion-app case): you need no storage permission at all on Android 10+
  (owner rule), `RELATIVE_PATH`-filtered MediaStore queries with Bundle args,
  Coil with default settings, and EXIF via `ExifInterface` over
  `openInputStream` — plus one trap: `ContentObserver` callbacks arrive on
  the Handler's looper, so a main-looper observer that re-queries inline does
  synchronous provider I/O on the main thread.

## Part 1: Enumeration — the MediaStore layer

### 1.1 The data model nobody explains in one place

**Collections and volumes.** `MediaStore.Images.Media.EXTERNAL_CONTENT_URI`
points at the *synthetic* volume `VOLUME_EXTERNAL` ("external") which, since
Android 10, is "a merged view of all media across all currently attached
external storage devices" — primary storage plus SD cards plus USB drives in
one query. You cannot insert into it; inserts go to a specific volume,
usually `VOLUME_EXTERNAL_PRIMARY` ("external_primary").
`MediaStore.getExternalVolumeNames(Context)` enumerates the real volumes
behind the synthetic one ([MediaStore.java](https://android.googlesource.com/platform/packages/providers/MediaProvider/+/refs/heads/main/apex/framework/java/android/provider/MediaStore.java)).
Fossify and Telegram both query the synthetic volume once and never iterate
volumes.

`MediaStore.Files` is the superset collection (all file types); Fossify
queries it exclusively — `Files.getContentUri("external")` — so one query
covers images, videos, raws, and SVGs, then filters by file extension.
Telegram queries `Images` and `Video` separately.

**Buckets are just hashed parent paths.** `BUCKET_ID` is
`String.hashCode()` of the *lowercased* parent directory path, and
`BUCKET_DISPLAY_NAME` is the directory's (original-case) name. This is not
folklore; it's AOSP's `FileUtils.computeValuesFromData`
([FileUtils.java](https://android.googlesource.com/platform/packages/providers/MediaProvider/+/refs/heads/main/src/com/android/providers/media/util/FileUtils.java)):

```java
final File fileLower = new File(data.toLowerCase(Locale.ROOT));
// Buckets are the parent directory
final String parent = fileLower.getParent();
if (parent != null) {
    values.put(MediaColumns.BUCKET_ID, parent.hashCode());
    values.put(MediaColumns.BUCKET_DISPLAY_NAME, file.getParentFile().getName());
```

Consequences: two folders differing only by case share a bucket; a folder
rename changes the bucket ID; an "album" is a directory, nothing more. Both
columns are read-only and auto-computed.

**The `DATA` column: deprecated, then un-deprecated.** Android 10 marked
`MediaColumns.DATA` deprecated and scoped storage made raw paths unusable
(unless the app opted out via `requestLegacyExternalStorage`). Android 11
re-enabled direct file-path access to *media files* through FUSE, and the
current AOSP javadoc carries **no** `@Deprecated` annotation — it now reads
"Apps may use this path to do file operations... From Android 11 onwards,
this column is read-only for apps that target R and higher... though direct
file operations are supported, `ContentResolver#openFileDescriptor` is
recommended for better performance." Google's storage docs quantify the
tradeoff: sequential reads via path are comparable to MediaStore; random
reads/writes "can be up to twice as slow"
([Android 11 storage changes](https://developer.android.com/about/versions/11/privacy/storage),
[shared media guide](https://developer.android.com/training/data-storage/shared/media)).
This is why both apps studied below still ship path-based pipelines in 2026.

**`IS_PENDING`.** A pending row is a file still being written by its owner;
"only the owner of the item can open the underlying file; requests from
other apps will be rejected." On Q+ pending rows are **filtered out of
queries by default** — you opt in with `QUERY_ARG_MATCH_PENDING`
(`MediaStore.setIncludePending` is deprecated in favor of it). Pending files
are physically name-mangled on disk (`.pending-<expires>-<name>` per
`FileUtils.PATTERN_EXPIRES_FILE`). A camera app using
`IS_PENDING=1` during capture then flipping it to 0 is invisible to other
galleries until the flip — and to its *own* default queries too.

**The date-column trap.** `DATE_TAKEN` is **milliseconds**
(`@CurrentTimeMillisLong`); `DATE_ADDED` and `DATE_MODIFIED` are **seconds**
(`@CurrentTimeSecondsLong`). Mixing them silently breaks sorting by a factor
of 1000. Worse, `DATE_TAKEN` is *derived*: the scanner's
`parseOptionalDateTaken`
([ModernMediaScanner.java](https://android.googlesource.com/platform/packages/providers/MediaProvider/+/refs/heads/main/src/com/android/providers/media/scan/ModernMediaScanner.java))
guesses a timezone offset from GPS time or file mtime, rounds it to
15-minute increments, and returns empty (NULL column) when EXIF
`DateTimeOriginal` is missing or the guess exceeds ±24h. The javadoc itself
warns that images need both `TAG_DATETIME_ORIGINAL` *and*
`TAG_OFFSET_TIME_ORIGINAL` for a reliable value. Screenshots, messenger
downloads, and DSLR imports routinely have NULL or shifted values — the
user-visible fallout is real
([Simple-Gallery #1022](https://github.com/SimpleMobileTools/Simple-Gallery/issues/1022),
[Fossify Gallery #140](https://github.com/FossifyOrg/Gallery/issues/140),
[aves #1047](https://github.com/deckerst/aves/issues/1047)).

### 1.2 Query mechanics and the Android 11 rupture

**Bundle-based queries** (`ContentResolver.query(uri, projection, Bundle,
signal)`, API 26+) carry `QUERY_ARG_SQL_SELECTION{,_ARGS}`,
`QUERY_ARG_SQL_SORT_ORDER` or the structured
`QUERY_ARG_SORT_COLUMNS`/`QUERY_ARG_SORT_DIRECTION`, and — the part with no
classic equivalent — `QUERY_ARG_LIMIT` and `QUERY_ARG_OFFSET`. Providers
advertise which args they honored via `EXTRA_HONORED_ARGS`.

Before this API, a decade of apps injected SQL into the `sortOrder`
parameter: `"date_added DESC LIMIT 100"`. Android 11's MediaProvider ended
that with strict SQL grammar checking, and the source is refreshingly candid
([MediaProvider.java](https://android.googlesource.com/platform/packages/providers/MediaProvider/+/refs/heads/main/src/com/android/providers/media/MediaProvider.java)):

```java
if (targetSdkVersion < Build.VERSION_CODES.R) {
    // Some apps are abusing "ORDER BY" clauses to inject "LIMIT"
    // clauses; gracefully lift them out.
    DatabaseUtils.recoverAbusiveSortOrder(queryArgs);
    // Some apps are abusing the Uri query parameters to inject LIMIT
    // clauses; gracefully lift them out.
    DatabaseUtils.recoverAbusiveLimit(uri, queryArgs);
}
```

Target below R: the abuse is "gracefully lifted out" into proper query args.
Target R+: `java.lang.IllegalArgumentException: Invalid token LIMIT`. Real
apps hit exactly this
([media_gallery #19](https://github.com/aloisdeniel/media_gallery/issues/19),
fixed by [PR #24](https://github.com/aloisdeniel/media_gallery/pull/24/files)
switching to Bundle args; [Gligar #20](https://github.com/OpenSooq/Gligar/issues/20)).

The same strictness killed the pre-Q album-list trick of injecting
`"1) GROUP BY 1,(2"` into the selection so the provider's `WHERE (<sel>)`
became `WHERE (1) GROUP BY 1,(2)` — one row per bucket, cover image free of
charge. MediaProvider now lifts that out for target < Q
(`recoverAbusiveSelection`) and throws for Q+. The modern replacement is
what Telegram does: fetch all rows, group in Java.

**CursorWindow.** A cross-process cursor pages rows through a shared-memory
window whose size is the framework resource `config_cursorWindowSize` =
2048 KB (frameworks/base `config.xml`; `CursorWindow.getCursorWindowSize()`).
Iterating 100k rows is fine — windows refill transparently — but a *single
row* larger than the window throws `SQLiteBlobTooBigException: Row too big
to fit into CursorWindow`. You cannot control the provider-side window size
for MediaStore cursors, so the defense is a narrow projection: don't select
huge string columns you don't need. (A widely-repeated claim that Android P+
auto-resizes windows could not be confirmed in current AOSP source —
`SQLiteCursor` does not grow the window; treat it as false.)

### 1.3 Case study: Fossify Gallery — filesystem-first

Fossify Gallery (successor of Simple Gallery,
[FossifyOrg/Gallery](https://github.com/FossifyOrg/Gallery)) treats
MediaStore as a *folder-discovery and metadata sidecar*, not the source of
truth. Key files: `app/src/main/kotlin/org/fossify/gallery/helpers/MediaFetcher.kt`,
`extensions/Context.kt`, `asynctasks/GetMediaAsynctask.kt`.

- **Everything is `MediaStore.Files`** (`Files.getContentUri("external")`) —
  never Images/Video collections. Type filtering happens by *extension
  matching on `DATA` in SQL*: `"${Images.Media.DATA} LIKE ?"` with args like
  `"%.jpg"` (MediaFetcher.kt:179-249).
- **Albums are parent-path strings, not buckets.** Folder keys are
  lowercased `File(path).parent` values; `BUCKET_ID` never appears in the
  enumeration path.
- **Two regimes**, split by one comment at MediaFetcher.kt:28: "on Android 11
  we fetch all files at once from MediaStore and have it split by folder".
  On R+ without All-Files-Access, `getAndroid11FolderMedia` queries the whole
  Files table with **no selection and no sort** and buckets rows into a
  `HashMap<lowercasedParentPath, ArrayList<Medium>>` in memory. Pre-R (or
  with `MANAGE_EXTERNAL_STORAGE`), it lists directories with the File API
  (`File(folder).listFiles()`, `file.lastModified()`) and uses MediaStore
  only to prefetch size/date maps. OTG storage goes through SAF
  `DocumentFile.listFiles()`.
- **Folder discovery** unions: parents of the 10 newest `_ID`s, hardcoded
  `DCIM`, `DCIM/Camera`, `Pictures`, parents of every matching `DATA` row,
  and user-included folders walked recursively.
- **The Android 11 LIMIT fix, in production** (MediaFetcher.kt:146-162): on
  R+ a Bundle with `QUERY_ARG_LIMIT = 10`, `QUERY_ARG_SORT_COLUMNS = [_ID]`,
  `QUERY_ARG_SORT_DIRECTION = DESCENDING`; pre-R the string
  `"_id DESC LIMIT 10"`. Both paths, side by side — a fossil record of the
  rupture.
- **`.nomedia` is found via MediaStore, verified on disk**: selection
  `MEDIA_TYPE = MEDIA_TYPE_NONE AND TITLE LIKE '%.nomedia%'`, then a
  `File.exists()` double-check before hiding the parent folder
  (Context.kt:496-527).
- **Sorting and grouping are in-memory** (comparators incl. natural-order
  `AlphanumericComparator`), never SQL. **No pagination** — full fetch,
  cached in Room so the UI paints from cache instantly and diffs after the
  re-fetch.
- **Dates**: `DATE_MODIFIED * 1000` normalization everywhere;
  `dateTaken == 0L → lastModified` fallback; and an own Room `dateTakensDB`
  whose values *override MediaStore* — a permanent workaround for the
  scanner's unreliable `datetaken` (section 1.1).
- **Error philosophy**: every cursor loop wraps each row in
  `try { ... } catch (e: Exception) {}` — empty catch, skip bad row, never
  crash; rows with `SIZE <= 0` are skipped (ghost/pending rows).

### 1.4 Case study: Telegram-Android — MediaStore-first

Telegram's picker
([DrKLO/Telegram](https://github.com/DrKLO/Telegram),
`TMessagesProj/src/main/java/org/telegram/messenger/MediaController.java`,
`loadGalleryPhotosAlbums`, line ~6136) is the opposite architecture: one
full-table cursor per collection, everything grouped in Java.

- **The projection contains an API-level column swap** (line ~183):

  ```java
  Build.VERSION.SDK_INT > 28
      ? MediaStore.Images.Media.DATE_MODIFIED
      : MediaStore.Images.Media.DATE_TAKEN,
  ```

  On Q+ Telegram simply refuses to use `DATE_TAKEN` — a production-grade
  vote of no confidence, consistent with the scanner internals in 1.1. The
  projection also includes `ORIENTATION` (Images) — MediaStore serves the
  EXIF rotation as a column, saving a per-file EXIF parse in the grid.
- **Query**: null selection, no LIMIT, SQL sort `<dateColumn> DESC`, on a
  `Thread.MIN_PRIORITY` background thread. No pagination; the only bound is
  "parse XMP of the first 15 photos" for motion-photo detection.
- **Albums**: `SparseArray<AlbumEntry>` keyed by **`BUCKET_ID`**; first row
  of a bucket creates the album (cover = newest photo, since rows arrive
  date-DESC), later rows append. Synthetic "All photos" album uses bucketId
  0 at position 0.
- **The camera album is a path prefix, not a bucket name**:
  `Environment.getExternalStoragePublicDirectory(DIRECTORY_DCIM) + "/Camera/"`,
  and the first album whose row path starts with that prefix gets promoted to
  the front. `DCIM/Camera` is a convention Telegram hardcodes — not a
  platform guarantee.
- **`DATA` is the identity of the whole pipeline**: rows with empty paths
  are skipped; grid thumbnails resolve through a custom scheme
  `thumb://<id>:<path>` that calls the legacy
  `MediaStore.Images.Thumbnails.getThumbnail(resolver, id, MINI_KIND, opts)`
  with a decode-from-file fallback (`ImageLoader.java:1179-1242`); send/upload
  opens the file by path. Viable precisely because of Android 11's FUSE
  re-enablement.
- **Permission gate** (line ~6157): on API 33+ the query proceeds if *any*
  of `READ_MEDIA_IMAGES`/`READ_MEDIA_VIDEO`/`READ_MEDIA_AUDIO` is granted —
  which also rides Android 14 partial access (the permission reads as
  granted for the session while MediaStore returns only the user-selected
  subset).
- **Error philosophy**: each pass wrapped in `catch (Throwable)` — not
  `Exception` — logging and showing whatever albums were built; cursor close
  itself wrapped again.
- **Refresh**: `ContentObserver`s on four URIs (Images/Video ×
  external/internal, `notifyForDescendants=true`) with a **2000 ms debounce**,
  deferred while the photo viewer is open, plus a cheap `COUNT(_id)` poll
  (`checkGallery()`) to catch missed observer events.

### 1.5 The two architectures, and what a read-only own-photos gallery needs

Fossify is *filesystem-first* (MediaStore discovers folders; `File` API
enumerates; own DBs patch bad columns) because its product is "browse any
folder, including hidden ones". Telegram is *MediaStore-first* (one cursor,
in-memory buckets) because its product is "pick recent media fast". Both
load everything, both group in memory, both still depend on `DATA`.

A third architecture is available when the gallery only shows **photos the
app itself captured** (the CameraX companion app that motivated this
report): on Android 10+ an app needs **no storage permission to read media
it contributed** — ownership is the `owner_package_name` column. The caveat:
uninstall/reinstall breaks ownership ("the system considers the file to be
attributed to the previously installed version"), after which the app must
request read permission to see its own old photos
([shared media guide](https://developer.android.com/training/data-storage/shared/media)).
Since Android 14, `owner_package_name` query results are restricted to
visible packages.

For that case the right shape is: query `Images.Media.EXTERNAL_CONTENT_URI`
with a `RELATIVE_PATH = 'Pictures/Camera/'` selection, narrow projection,
Bundle args for sort+limit, content URIs only (no `DATA` at all), and a
`ContentObserver` + requery for refresh. One trap observed in a real
implementation of exactly this pattern: **`ContentObserver.onChange` runs on
the Handler's looper you registered with** — register with a main-looper
`Handler` and re-query inline, and the MediaStore query (cross-process
binder + SQLite) executes on the main thread; a `flowOn(Dispatchers.IO)`
downstream does not move it. Register the observer with a background
`Handler`, or bounce `onChange` to a coroutine. Also note the reinstall
caveat above is *the* reason such an app still works after reinstall only if
it requests `READ_MEDIA_IMAGES` or filters to rows it can actually open.

### 1.6 The sanctioned alternatives: Photo Picker, SAF, ACTION_PICK

If the product need is "let the user hand me some photos" rather than "be a
gallery", enumeration is the wrong tool post-2022:

- **Photo Picker** (`MediaStore.ACTION_PICK_IMAGES`, API 33): system UI, no
  permission at all, returns read-only picker URIs limited to
  `PickerMediaColumns`. Available on Android 11+ devices receiving Google
  System Updates; devices back to API 19 get a GMS-backported version.
  `ActivityResultContracts.PickVisualMedia` encodes the real fallback chain
  (from AndroidX source): `ACTION_PICK_IMAGES` when SDK ≥ 33 *or* SDK ≥ 30
  with R-extension ≥ 2 → a GMS system-app fallback action →
  `ACTION_OPEN_DOCUMENT`. Multi-select is capped by
  `MediaStore.getPickImagesMaxLimit()`. Grants last "until the device is
  restarted or until your app stops" unless you call
  `takePersistableUriPermission` — with a documented cap of **5,000 media
  grants**, oldest evicted
  ([photo picker guide](https://developer.android.com/training/data-storage/shared/photopicker)).
- **SAF** (`ACTION_OPEN_DOCUMENT`): persistable grants across reboots via
  `takePersistableUriPermission`; the persisted-grant cap was 128, raised to
  **512 in Android 11**
  ([issue 149315521](https://issuetracker.google.com/issues/149315521)).
  Tree grants can't target root/Download on 11+.
- **`ACTION_PICK`**: still not `@Deprecated` in AOSP source, but every
  current doc routes to the photo picker; treat it as legacy.

**The permission timeline** for apps that *do* enumerate:

| API | Release | What you request | What you get |
|-----|---------|------------------|--------------|
| ≤ 28 | ≤ 9 | `READ_EXTERNAL_STORAGE` | raw paths work everywhere |
| 29 | 10 | `READ_EXTERNAL_STORAGE` (+ `requestLegacyExternalStorage` opt-out) | scoped storage; paths break unless opted out; owner rule begins |
| 30 | 11 | `READ_EXTERNAL_STORAGE` | opt-out ignored for target R+; FUSE re-enables media file paths; strict query grammar |
| 33 | 13 | `READ_MEDIA_IMAGES` / `_VIDEO` / `_AUDIO` | `READ_EXTERNAL_STORAGE` "has no effect"; auto-migrated on upgrade; Photo Picker ships |
| 34 | 14 | + `READ_MEDIA_VISUAL_USER_SELECTED` | "Select photos" partial access; without the new permission you get session-scoped grants (compatibility mode); don't cache permission state, re-check in `onResume` |

Under partial access, `checkSelfPermission(READ_MEDIA_IMAGES)` can report
granted while MediaStore silently returns only the selected subset — code
that assumes "granted = everything" (most pre-2023 code) sees a mysteriously
tiny gallery. Google's UX guidance: never re-prompt spontaneously; provide a
"manage selection" affordance
([partial access docs](https://developer.android.com/about/versions/14/changes/partial-photo-video-access)).

### 1.7 Enumeration quirks worth knowing

- **Scan latency**: a captured file can exist on disk before its row is
  visible (pending flag + scanner). Telegram's belt-and-suspenders answer:
  debounced ContentObservers *plus* a COUNT poll. For sync logic, prefer
  `MediaStore.getGeneration()` over date columns — dates "can change when an
  app calls setLastModified() or when the user changes the system clock";
  the generation is monotonic.
- **Screenshot buckets differ by OEM**: Samsung One UI saves screenshots to
  `DCIM/Screenshots`, stock Android to `Pictures/Screenshots` — enough to
  change which albums exist and what auto-backup tools treat as camera media
  ([9to5google](https://9to5google.com/2021/05/21/android-12-screenshot-storage-change-samsung-google-photos/)).
  Older MIUI adds gallery-cloud paths like `MIUI/Gallery/cloud/...`.
- **`DCIM/Camera` is a convention, not a contract** — Telegram hardcodes it;
  OEM camera apps have historically varied.

## Part 2: Decoding — from JPEG to pixels

Enumeration hands you a URI. Everything expensive happens after that.

### 2.1 The two API surfaces, and the one codec under both

Android ships two still-image decode APIs, and the most common mental model of
them — "ImageDecoder is the new fast one" — is wrong in an instructive way.
Coil's maintainer put it plainly in the thread that eventually added
ImageDecoder support to Coil 3
([coil #1943](https://github.com/coil-kt/coil/issues/1943), colinrtwhite,
2023-12-09):

> My understanding was BitmapFactory and ImageDecoder are backed by the same
> Skia decoder.

He is right, and AOSP confirms it. `BitmapFactory`'s four native entry points
(`nativeDecodeStream`, `nativeDecodeFileDescriptor`, `nativeDecodeAsset`,
`nativeDecodeByteArray`) all funnel into one `doDecode()`
([BitmapFactory.cpp:298](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/libs/hwui/jni/BitmapFactory.cpp)),
which builds `SkCodec::MakeFromStream(...)` → `SkAndroidCodec::MakeFromCodec(...)`
(BitmapFactory.cpp:354-372). `ImageDecoder`'s JNI builds the same
`SkAndroidCodec` and calls the same `getAndroidPixels`. Same libjpeg-turbo,
same libpng, same libwebp, same HEIF path. **The differences are not the codec
— they are the IO path, the EXIF pass, the allocator default, and whether you
can reuse memory.**

| | `BitmapFactory` (API 1) | `ImageDecoder` (API 28) |
|---|---|---|
| Input | stream/fd/byte[]/asset | `ImageDecoder.Source` (fd, file, ByteBuffer, resource) |
| Scaling | `inSampleSize` (codec-native) + `inDensity`/`inTargetDensity` fractional trick | `setTargetSize(w, h)` — **arbitrary**, sampled + one bilinear pass |
| EXIF orientation | **ignored entirely** | **applied natively**, always |
| Default allocator | software, ARGB_8888 | **hardware where possible** |
| Bitmap reuse | `inBitmap` | **none — every decode allocates** |
| Corrupt input | returns `null` | throws `DecodeException` (or partial via listener) |
| Animation | no | `AnimatedImageDrawable` from the same call |

That "no inBitmap" row is why every pooling library kept BitmapFactory: `rg -n
inBitmap ImageDecoder.java` over the 2,180-line source returns nothing. There
is no reuse API, and there is no plan for one — hardware bitmaps made pooling
mostly moot (see 2.6).

The remaining question is speed, and there is one real benchmark rather than
folklore. In the same Coil thread, a user measured both APIs on an sm7250
(Snapdragon 765G-class, Android 14, build UP1A.231105.003), 10 runs each, over
disk-cache files with the page cache defeated:

| Case | BitmapFactory | ImageDecoder |
|---|---|---|
| ~1000 px JPEG, full decode | 53 ms | 49 ms |
| same JPEG, target 640×480 | 37 ms | 30 ms |
| 1 MP PNG, full | 226 ms | 216 ms |
| 1 MP PNG, target 640×480 | 168 ms | 159 ms |
| 4 MP PNG, full | 253 ms | 233 ms |
| 4 MP PNG, target 640×480 | 206 ms | 183 ms |

5–15%, and the reporter correctly diagnoses where it comes from: "the main cost
is 'Decode', but we can still save some IO overhead (and additionally, some gc
pressure overhead?)". ImageDecoder takes an fd or DirectByteBuffer directly, so
encoded bytes never bounce through the ART heap;
`BitmapFactory.decodeStream(InputStream)` calls *back into Java* for every
buffer fill. Colin White's reply is the honest engineering position — "Depending
only on BitmapFactory is much better for consistency and maintenance so I'd
prefer to continue with only one decoder unless there are demonstrable
performance gains" — followed, once the numbers landed, by "That looks like a
solid decode time improvement! Would you be open to submitting a PR?" That PR
is Coil 3's `StaticImageDecoder`.

### 2.2 Why `inSampleSize` is nearly free (and exact scaling is not)

The single most important performance fact in this whole report lives in
libjpeg-turbo, not in Android. `SkJpegCodec::onGetScaledDimensions`
([SkJpegCodec.cpp:230-262](https://skia.googlesource.com/skia/+/refs/heads/main/src/codec/SkJpegCodec.cpp))
says it outright:

> libjpeg-turbo supports scaling by 1/8, 1/4, 3/8, 1/2, 5/8, 3/4, 7/8, and
> 1/1, so we will support these as well

The mechanism is `scale_num`/`scale_denom` with the denominator pinned at 8
(`onDimensionsSupported`, SkJpegCodec.cpp:364-404, walks num from 8 down to 1
until `jpeg_calc_output_dimensions` matches). These M/8 scales are implemented
**inside the IDCT**: at 1/8 the decoder computes only the DC coefficient of each
8×8 block and stops; 1/2 and 1/4 run reduced-size IDCTs. Full-resolution pixels
are never reconstructed. A 1/8 decode isn't "decode then shrink" — it is a
different, drastically cheaper transform reading the same entropy-coded data.

`BitmapFactory` exposes this as `inSampleSize`, which maps straight through:
`codec->getSampledDimensions(sampleSize)` (BitmapFactory.cpp:384) and
`codecOptions.fSampleSize = sampleSize` handed to `getAndroidPixels`
(BitmapFactory.cpp:508-513). Subsampling happens *during* decode.

Anything the codec can't do natively costs a second full pass. When the sample
size doesn't divide the dimensions evenly, or a density scale applies,
BitmapFactory redraws the decoded bitmap through an `SkCanvas` with
`SkSamplingOptions(SkFilterMode::kLinear)` into a **second allocation**
(BitmapFactory.cpp:562-598). That's the "fine scale" step, and it is why the
power-of-two ladder is the fast path and arbitrary sizes are not.

ImageDecoder fuses the two: `setTargetSize` calls
`mCodec->computeSampleSize(&decodeSize)` — Skia picks the largest codec-native
sample size producing output ≥ target — and the remainder becomes a matrix
scale in the same `decode()` call
([hwui-ImageDecoder.cpp:104-143, 421-497](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/libs/hwui/hwui/ImageDecoder.cpp)).
Subsampled decode plus one bilinear pass, arbitrary target, one API call. That
is genuinely nicer than what BitmapFactory callers must hand-build — which
brings us to Glide.

### 2.3 Case study: Glide's `Downsampler` — the two-pass dance in full

`Downsampler.java` is ~1,000 lines of accumulated scar tissue and the best
single document of what BitmapFactory actually does. The shape:

**Pass 1, bounds.** `getDimensions()` sets `inJustDecodeBounds = true`, decodes,
resets the flag, returns `{outWidth, outHeight}` (Downsampler.java:790-800).
Between passes the stream is rewound through an `ImageReader` abstraction —
InputStream mark/reset, ByteBuffer, or `ParcelFileDescriptorRewinder`
(lines 207-288). One subtlety worth stealing: after bounds are read,
`imageReader.stopGrowingBuffers()` fixes the mark limit so the full decode
doesn't balloon the buffered header (lines 808-816, "See issue #225").

**When bounds fail.** `sourceWidth == -1` force-disables the hardware config,
with a comment explaining that a header can be so large (10 MB+) that
`inJustDecodeBounds` itself fails and a full-size mutable decode is the only
option (lines 347-353). `calculateScaling` early-returns on non-positive dims —
no downsampling at all (lines 517-531). Coil does the same (sampleSize = 1, no
scaling, BitmapFactoryDecoder.kt:115-120).

**Pass 2, sample size.** The core is four lines:

```java
int widthScaleFactor  = orientedSourceWidth  / outWidth;   // integer division
int heightScaleFactor = orientedSourceHeight / outHeight;
int scaleFactor = rounding == SampleSizeRounding.MEMORY
    ? Math.max(widthScaleFactor, heightScaleFactor)        // smaller output
    : Math.min(widthScaleFactor, heightScaleFactor);       // QUALITY
int powerOfTwoSampleSize = Math.max(1, Integer.highestOneBit(scaleFactor));
```

(lines 576-597; source dims are EXIF-swapped first at 539-542 so downsampling
matches the *post-rotation* target). `highestOneBit` rounds **down** to a power
of two; for `MEMORY` rounding, if `powerOfTwoSampleSize < (1f / exactScaleFactor)`
it doubles once (`<< 1`). Coil's version is the same idea in one expression:
`(srcWidth / dstWidth).takeHighestOneBit()`, `Scale.FILL -> min`,
`Scale.FIT -> max` (DecodeUtils.kt:19-33).

**Then Glide reimplements the framework's rounding.** This is the part that
tells you how little the platform guarantees. To predict the post-sample size,
`Downsampler` replicates per-format behavior it determined empirically —
"tested on emulators for Android versions 15-26" (lines 600-645):

- **JPEG**: libjpeg-turbo downsamples natively only up to 8 and uses *ceiling*;
  Skia does the rest by integer division (*floor*). So:
  `nativeScaling = min(sampleSize, 8); ceil(dim / nativeScaling); then / (sampleSize / 8)`.
- **PNG**: always floor.
- **WebP**: round on N+, floor before N.
- **Unknown formats** with non-divisible dims: re-run the *bounds* decode with
  `inSampleSize` set, because "inJustDecodeBounds decodes do obey inSampleSize".

A third bounds decode to find out what the second decode will produce. That is
the tax for an API that never told you.

There is also a fossil: WBMP and ICO get `sampleSize = 1` on SDK ≤ 23
(`NO_DOWNSAMPLE_PRE_N_MIME_TYPES`, lines 136-139, 588-591, citing b/27305903).

### 2.4 The `inDensity`/`inTargetDensity` trick — exact sizes out of a power-of-two API

Power-of-two sampling gets you within a factor of two of your target. Both Glide
and Coil close the gap by abusing BitmapFactory's *density* scaling, which
exists for drawable resources and happens to accept any ratio.

The native implementation is three lines
(BitmapFactory.cpp:339-345, 440-445):

```cpp
scale = (float) targetDensity / density;          // only when inScaled
...
scaledWidth = static_cast<int>(scaledWidth * scale + 0.5f);   // round half up
```

So `inTargetDensity / inDensity` is an arbitrary rational scale factor applied
during the decode. Coil's use is the direct one (BitmapFactoryDecoder.kt:144-170):
downscale → `inDensity = Int.MAX_VALUE; inTargetDensity = (Int.MAX_VALUE * scale).roundToInt()`;
upscale → the inverse; `inScaled = scale != 1.0`. Use the largest available
denominator, get the finest available precision.

Glide goes one step further and corrects for the float division itself
(Downsampler.java:697-721):

```java
// BitmapFactory calculates the density scale factor as a float. This
// introduces some non-trivial error.
private static int adjustTargetDensityForError(double adjustedScaleFactor) {
  int densityMultiplier = getDensityMultiplier(adjustedScaleFactor);
  int targetDensity = round(densityMultiplier * adjustedScaleFactor);
  float scaleFactorWithError = targetDensity / (float) densityMultiplier;
  double difference = adjustedScaleFactor / scaleFactorWithError;
  return round(difference * targetDensity);
}
```

`getDensityMultiplier` returns `round(Integer.MAX_VALUE * min(sf, 1/sf))` — the
largest denominator that won't overflow — then the naive target density is
computed, the resulting float error is *measured*, and the target density is
pre-distorted to cancel it. Glide is compensating for rounding error in a
graphics API from 2008 by predicting its floating-point division. Both libraries
then have to repair the bitmap's metadata afterwards, since the decoder now
believes the image came from a weird density:
`downsampled.setDensity(displayMetrics.densityDpi)` (Downsampler.java:479);
`outBitmap.density = displayMetrics.densityDpi` (BitmapFactoryDecoder.kt:79-80).

One gate: pre-KitKat, "density scaling is only supported if inBitmap is null"
(lines 651-656) — you got reuse or exact size, not both.

### 2.5 `inBitmap` reuse: the pooling era and its end

The contract, from the javadoc
([BitmapFactory.java:57-108](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/graphics/java/android/graphics/BitmapFactory.java)):

> If the decode operation cannot use this bitmap, the decode method will throw
> an IllegalArgumentException. The current implementation necessitates that the
> reused bitmap be mutable.

KitKat relaxed it to "any mutable bitmap ... as long as the resulting byte count
of the decoded bitmap is <= the allocated byte count of the reused bitmap".
Before KitKat: JPEG/PNG only, equal size, `inSampleSize = 1`, and the reused
bitmap's config *overrode* `inPreferredConfig`. Glide still encodes exactly that
gate (`TYPES_THAT_USE_POOL_PRE_KITKAT`, lines 152-157, 391-393).

The failure mode is nastier than it looks. All four decode entry points end with
`if (bm == null && opts != null && opts.inBitmap != null) throw new
IllegalArgumentException("Problem decoding into existing bitmap")`
(BitmapFactory.java:635, 686, 785, 866) — so a bitmap that is merely *too small*
surfaces as `tryAllocPixels` failure → null → IAE, and an *immutable* one is
silently ignored with `ALOGW("Unable to reuse an immutable bitmap as an image
decoder target.")` (BitmapFactory.cpp:448-458). Worse, corrupt input with
`inBitmap` set throws IAE rather than returning null. Glide's comment says it:

> BitmapFactory throws an IllegalArgumentException if any error occurs
> attempting to decode a file when inBitmap is non-null, including those caused
> by partial or corrupt data

(Downsampler.java:921-924) — so `decodeStream` catches IAE, returns the bitmap to
the pool, nulls `inBitmap`, and **recursively retries without reuse** (lines
826-845). You cannot distinguish "bad pooled bitmap" from "bad JPEG" except by
trying again.

Two details worth having: Glide asks the pool for a *dirty* bitmap
(`bitmapPool.getDirty(...)`) because "BitmapFactory will clear out the Bitmap
before writing to it" — the normal `get()` path burns an
`eraseColor(Color.TRANSPARENT)` for issue #131 that a full decode makes pointless
(LruBitmapPool.java:147-171). And the expected pooled size must account for
*both* stages: `ceil(sourceDim / sampleSize)` then `round(dim * inTargetDensity /
inDensity)` (Downsampler.java:402-409). Get that wrong and you're back in the
IAE-retry path.

**Then the ground shifted.** Glide's own `MemorySizeCalculator` records the
verdict in a constant:

```java
BITMAP_POOL_TARGET_SCREENS = Build.VERSION.SDK_INT < Build.VERSION_CODES.O ? 4 : 1;
// On Android O+, we use HARDWARE for all reasonably sized images ...
// the Bitmap pool is much less important on O
```

(lines 132-141). Four screens' worth of pool became one; on O+ low-RAM devices,
**zero** (lines 165-171). The rest of the budget: `memoryClass * multiplier`
where multiplier is `0.4f`, or `0.33f` when `ActivityManager.isLowRamDevice()`
(lines 100-106, 142-143); memory cache = 2 screens; array pool = 4 MB, halved on
low-RAM.

Coil went further and **deleted the pool in 2.0**. `BitmapFactoryDecoder` now
sets `inMutable = false` unconditionally, with the comment "Always create
immutable bitmaps as they have better performance" (line 57-58). Fresco is the
holdout: `DefaultRawBitmapDecoder` still does `options.inBitmap = bitmapToReuse`
from its own `BitmapPool` sized by a `bitmapSizeCalculator` (lines 241-267),
disabling it exactly where it can't work — "Cannot reuse bitmaps with
Bitmap.Config.HARDWARE" (lines 243-255) — and region decodes fall back from
HARDWARE to ARGB_8888 (lines 252-254).

And `ImageDecoder` has no `inBitmap` at all. The pooling era ended not because
pooling stopped working but because the pixels moved somewhere the Java heap
couldn't feel them.

### 2.6 Hardware bitmaps: the biggest win and the sharpest edge

Where bitmap pixels live has changed three times
([manage-memory](https://developer.android.com/topic/performance/graphics/manage-memory)):
API ≤ 10, native memory; **API 11–25, the Dalvik/ART heap** (hence a decade of
OOM-driven pooling); **API 26+, the native heap**. `Config.HARDWARE`, added in
the same release, moves them once more — into graphics memory:

> Special configuration, when bitmap is stored only in graphic memory. Bitmaps
> in this configuration are always immutable. It is optimal for cases, when the
> only operation with the bitmap is to draw it on a screen.
> — [Bitmap.java:570-577](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/graphics/java/android/graphics/Bitmap.java)

Glide's docs quantify the win: "Hardware Bitmaps require ½ the memory of other
Bitmap configurations" — one copy in graphics memory instead of app-heap copy
plus GPU texture — and they "avoid jank caused by texture uploads at draw time"
([hardwarebitmaps](https://bumptech.github.io/glide/doc/hardwarebitmaps.html)).
They also don't show up in Java or native heap accounting at all; look under
Graphics/gralloc in `dumpsys meminfo`.

The edges are numerous and every one of them is a runtime throw:

- `getPixel()`, `getPixels()`, `getColor()`, `copyPixelsToBuffer()`,
  `copyPixelsFromBuffer()` → `IllegalStateException` from `checkHardware`
  (Bitmap.java:626-631, 672, 2140, 2165, 2214).
- `createBitmap(..., HARDWARE, isMutable = true)` → IAE "Hardware bitmaps are
  always immutable" (lines 722-727).
- Drawing into a **software** Canvas → `IllegalStateException` at draw time
  ("Software rendering doesn't support hardware bitmaps", from
  `BaseCanvas.throwIfHwBitmapInSwMode`). This is how it bites in practice:
  Glide's docs list "Screenshots triggered by code that try to draw the view
  hierarchy with Canvas" as a broken operation, with `PixelCopy` as the escape
  hatch on O+.
- Any precondition expecting ARGB_8888.

And then there are the file descriptors. Glide's `HardwareConfigState` is a
monument to this:

```java
// Each HARDWARE Bitmap requires two FDs (depending on the driver).
// Processes have an FD limit of 1024 (at least on O) ...
// we can blow through the FD limit, which causes graphics errors,
// Binder errors, and a variety of crashes
```

(lines 38-51). So Glide counts entries in `/proc/self/fd` — not every decode,
because each check costs 1–2 ms, but every 50 (`MINIMUM_DECODES_BETWEEN_FD_CHECKS
= 50`, lines 53-57), caching the boolean in between, and running the check *last*
in `isHardwareConfigAllowed` because it has side effects (lines 117-161). The cap
is `MAXIMUM_FDS_FOR_HARDWARE_CONFIGS_P = 20000`, reduced to **500** on a hardcoded
list of OnePlus P devices (GM1900/1901/1903/1911/1915, ONEPLUS A3000/A3010/A5010/
A5000/A3003/A6000/A6003/A6010/A6013) whose API-28 builds kept O-era FD limits
(b/139097735, lines 59-66, 185-210).

Coil's version of the same defense is smaller and differently drawn
(`hardwareBitmaps.kt:14-18`): never below API 26; FD counting on **26 and 27
only** (limit 800, re-checked every 30 decodes or 30 s, plus a
`MIN_SIZE_DIMENSION = 100` floor — don't spend an FD on a tiny image); always
allowed on 28+, because "This limit was increased to a safe number in API 28
(32768). Each hardware bitmap allocation consumes, on average, 2 file
descriptors" (lines 59-105). Its `IS_DEVICE_BLOCKED` list is the harsher
statement: on API 26, **every Samsung Galaxy** — any `Build.MODEL` starting `SM-`
or `SAMSUNG-SM-` — plus Moto E5/G6 families, Xperia XA1 variants; on API 27, LG
Tribute/K11/Q7/Stylo 4, T-Mobile REVVL 2, and a long tail of Alcatel/RCA/Umidigi/
Ulefone/Blackview/Leagoo/Blu devices. The file's own summary: "Maintains a list
of devices with broken/incomplete/unstable hardware bitmap implementations."

Glide reached the same conclusion by a blunter route. Its history is legible in
tags: v4.9.0 allowed hardware from O with `MAXIMUM_FDS_FOR_HARDWARE_CONFIGS = 700`;
v4.11.0 split the limits (700 on O, 20,000 on P) and added a Samsung blocklist
(SM-J720, SM-G960, SM-G965, SM-G935, SM-G930, SM-A520...) that disallowed hardware
*exactly on API 26* — `return Build.VERSION.SDK_INT != Build.VERSION_CODES.O;`.
Current master deleted the question:

```java
HARDWARE_BITMAPS_SUPPORTED = Build.VERSION.SDK_INT >= Build.VERSION_CODES.P;
```

Two libraries, two evidence bases, one conclusion: **Android O's hardware bitmaps
were not shippable.**

Even on P they're not free. `BLOCK_HARDWARE_BITMAPS_WHEN_GL_CONTEXT_MIGHT_NOT_BE_INITIALIZED
= SDK_INT < Q` (lines 24-31): on P, Glide blocks hardware bitmaps until
`unblockHardwareBitmaps()` is called from the main thread once the first frame is
drawn, working around a framework native crash when allocating a hardware bitmap
before the GL context exists (b/126573603#comment12). So the first images your P
device loads at cold start are software bitmaps by design.

The platform's own policy, for comparison, is far more relaxed
(ImageDecoder.cpp:294-297):

```cpp
const bool isHardware = !requireMutable
    && (allocator == kDefault_Allocator || allocator == kHardware_Allocator)
    && colorType != kGray_8_SkColorType;
```

`ALLOCATOR_DEFAULT` "will typically result in a Bitmap.Config#HARDWARE
allocation, but may be software for small images", and if the upload fails it
silently degrades to software — whereas `ALLOCATOR_HARDWARE` throws OOME "failed
to allocate hardware Bitmap!" (ImageDecoder.cpp:479-500). Note what this means:
**plain `ImageDecoder.decodeBitmap` hands you a hardware bitmap by default**, and
the first `getPixel` call in your codebase discovers it.

Finally, the "hardware" decode isn't hardware. BitmapFactory's HARDWARE path
decodes to a heap allocation and *then* calls
`Bitmap::allocateHardwareBitmap(outputBitmap)` — the comment says "extra
swizzling & upload to gralloc step" (BitmapFactory.cpp:473-477, 643-657). The
peak memory during decode is unchanged; the steady state is what improves.

### 2.7 EXIF orientation: the asymmetry that produces upside-down photos

BitmapFactory ignores EXIF. There is no orientation code in `doDecode` at all.
ImageDecoder always applies it — in native code, invisible from Java (`rg -i
orientation ImageDecoder.java` finds nothing). The work happens in hwui:
`getOrigin()` returns `mCodec->codec()->getOrigin()` (SkCodec's `SkEncodedOrigin`:
EXIF for JPEG, **irot/imir container properties for HEIF/AVIF**), and `decode()`
folds it into the output matrix:

```cpp
const bool handleOrigin = origin != kDefault_SkEncodedOrigin;
if (scale || handleOrigin || mCropRect) {
    outputMatrix.preConcat(SkEncodedOriginToMatrix(origin, targetWidth, targetHeight));
    if (SkEncodedOriginSwapsWidthHeight(origin)) std::swap(targetWidth, targetHeight);
```

(hwui-ImageDecoder.cpp:420-500). The codec decodes into a temp heap bitmap, then
it's drawn through the orientation+scale+crop matrix with `SkBlendMode::kSrc` and
linear filtering. Reported dimensions are pre-swapped (`mTargetSize =
swapWidthHeight() ? swapped(mDecodeSize) : ...`, lines 76, 216-227). It costs an
extra full-size temp allocation and a draw — there's a FIXME in the code about it:
"Use scanline decoding on only a couple lines to save memory. b/70709380".

Every BitmapFactory-based library therefore reimplements EXIF from scratch, and
they've each drawn the cost/coverage line differently:

- **Glide** parses orientation off the raw stream *before* decode with its own
  `DefaultImageHeaderParser`: walk JPEG segments (`0xFF` starts, stop at SOS 0xDA
  or EOI 0xD9), find APP1 (`0xE1`), require the `"Exif\0\0"` preamble, read byte
  order from `MM`/`II`, walk 12-byte IFD tags at `ifdOffset + 2 + 12*tagIndex` for
  tag `0x0112` (lines 420-557). Segment bytes come from the `ArrayPool`. The
  orientation is used three ways: swap the target dims, **veto the hardware
  config**, and rotate after decode via
  `TransformationUtils.rotateImageExif`. Notably `getOrientation` `handles()` only
  JPEG and raw TIFF (lines 563-567) — it returns `UNKNOWN_ORIENTATION` for HEIF,
  PNG, and WebP. That is a deliberate scope, and also a bug report
  ([glide #4265](https://github.com/bumptech/glide/issues/4265), "HEIC Images not
  rotated based on Exif info"; maintainer's answer: use androidx
  `exifinterface` 1.2.0, the release that added HEIF orientation reading).
- **Coil** uses androidx `ExifInterface` over a peeked stream, with a magnificent
  hack — the peeked source reports `available()` as 1 GB "so ExifInterface won't
  stop reading the stream prematurely" (ExifUtils.kt:27, 88-113). Its default
  strategy is `RESPECT_PERFORMANCE`: honor EXIF only for `image/jpeg`,
  `image/webp`, `image/heic`, `image/heif`. `RESPECT_ALL` carries a warning —
  it "can potentially cause out of memory errors as certain image formats (e.g.
  PNG) will be buffered entirely into memory" (ExifOrientationStrategy.kt:30-57).
  EXIF is why Coil's ImageDecoder path is conditional at all: `StaticImageDecoder`
  registers ahead of `BitmapFactoryDecoder` only when
  `SDK_INT >= 29 && imageDecoderEnabled && strategy == RESPECT_PERFORMANCE`.
- **Telegram** doesn't rotate at all. It parses orientation + mirror from the
  stream, rewinds (`is.getChannel().position(0)`), decodes, and ships an
  `ExtendedBitmapDrawable(image, orientation, invert)` — the rotation is applied
  **at draw time** by `ImageReceiver` (ImageLoader.java:1482-1496, 1606-1609). No
  second bitmap, no second full-size allocation. The cost is that every consumer
  must go through Telegram's own drawable.

The rotation itself is a `Matrix` + `Bitmap.createBitmap(in, 0, 0, w, h, matrix,
true)` in both Glide and Coil — meaning a full second allocation of the rotated
result and a recycle of the first. Glide used to draw into a pooled bitmap via
Canvas and deliberately stopped: "BitmapPool doesn't preserve gainmaps and color
space, so use Bitmap.create to apply the matrix" (TransformationUtils.java:345-364).
Ultra HDR broke the optimization.

Now the trap, and it is a good one. Since ImageDecoder rotates and BitmapFactory
doesn't, "does the platform apply EXIF?" has no single answer — **and it isn't
even consistent within ImageDecoder**. From Immich's fix
([immich #29337](https://github.com/immich-app/immich/pull/29337)):

> Android's ImageDecoder and loadThumbnail (API 29+) don't apply the EXIF
> orientation tag for raw files, so the decoded bitmap comes back unrotated...
> the jpeg/heic decoders rotate on their own, raw doesn't

Portrait DNGs displayed sideways. Fixed by rotating raw *only*. An app that
uniformly applies EXIF rotation after ImageDecoder double-rotates its JPEGs; an
app that applies none shows raws sideways. There is no uniform rule — the
asymmetry *is* the trap.

HEIF adds a second axis. Immich again
([#27820](https://github.com/immich-app/immich/pull/27820)), quoting MIAF
(ISO/IEC 23000-22:2024) §7.3.10.1:

> There should be no image transformations expressed by Exif (rotation,
> mirroring, etc.) ... If present, they shall be ignored by a MIAF renderer.

HEIF orientation lives in `irot`/`imir` container properties; EXIF orientation in
a HEIC is at best redundant and at worst wrong. The platform surfaces the
container rotation as `VideoFrame::mRotationAngle` (HeifDecoderImpl.cpp:50) and
Skia applies it as origin — so an app that *also* reads EXIF double-rotates HEICs
whose EXIF disagrees with irot.

And one consequence that costs real memory: because a hardware bitmap can't be
redrawn, needing EXIF rotation forces a software decode. Glide:
`isExifOrientationRequired` vetoes the hardware config outright
(HardwareConfigState.java:140-145). Coil: `if (exifData.isFlipped ||
exifData.isRotated) config = config.toSoftware()` (BitmapFactoryDecoder.kt:95-98).
Your camera's own photos, which are exactly the ones with orientation tags, are
exactly the ones that lose the hardware-bitmap win.

### 2.8 Config, color space, and the 565 question

`Downsampler.calculateConfig` (lines 736-780) is a decision tree worth reading as
a spec: try HARDWARE first; else `PREFER_ARGB_8888` (Glide's default
`DecodeFormat`); force ARGB_8888 on Jelly Bean because "Changing configs can cause
skewing on 4.1, see issue #128"; else header-parse alpha — alpha → ARGB_8888,
opaque → RGB_565 with `inDither = true`.

Coil is stricter about when 565 is allowed: `allowRgb565` downgrades **only for
JPEG** (`outMimeType == MIME_TYPE_JPEG`, lines 100-103), and RGBA_F16 sources stay
F16 unless hardware, because "High color depth images must be decoded as either
RGBA_F16 or HARDWARE" (lines 105-108). The platform agrees with the logic:
`MEMORY_POLICY_LOW_RAM` decodes opaque images to RGB_565 — but explicitly *not*
when the result will be hardware, since "decoding to 565 and then uploading to
the gpu as 8888 will not save memory" (ImageDecoder.cpp:273-286).

Telegram, having no such abstraction, hardcodes device tiers:
`SharedConfig.deviceIsHigh() || isRound ? ARGB_8888 : RGB_565` (ImageLoader.java:1810),
ARGB_8888 for blur/filter paths, RGB_565 for thumbs, `inDither = false`.

Color space is where Glide is quietly conservative: on P+, DISPLAY_P3 is used only
if the caller requested it *and* `options.outColorSpace` is already wide gamut;
on O, always SRGB (lines 441-452). A P3 photo will not decode into P3 unless you
ask. Fresco forces SRGB on O+ when configured
(`DefaultRawBitmapDecoder`, lines 270-277).

### 2.9 Concurrency: how many decodes at once?

Three libraries, three answers, and the folklore is wrong about Glide.

Glide wraps every BitmapFactory decode in
`TransformationUtils.getBitmapDrawableLock()` (Downsampler.java:824, 847), which
reads like a global decode lock and is not one. `BITMAP_DRAWABLE_LOCK` is a real
`ReentrantLock` **only** when `Build.MODEL` is in `MODELS_REQUIRING_BITMAP_LOCK` —
Moto X gen2 (XT1085–XT1098), Moto G gen1 (XT1031–XT1045), Moto G gen2
(XT1063–XT1079). Everywhere else it's a `NoLock` stub (TransformationUtils.java:44-87).
The comment: "On some devices, bitmap drawing is not thread safe. This lock only
locks for these specific devices"
([glide #738](https://github.com/bumptech/glide/issues/738)). **On modern devices
Glide decodes fully concurrently**, bounded only by its thread pools.

Coil bounds it explicitly. Decodes run inside
`parallelismLock.withPermit { runInterruptible { ... } }` with
`DEFAULT_MAX_PARALLELISM = 4` (BitmapFactoryDecoder.kt:39-41, 208-210), tunable via
`ImageLoader.Builder.bitmapFactoryMaxParallelism`. Crucially the *same* semaphore
instance is shared with `StaticImageDecoder.Factory`
(RealImageLoader.android.kt:88-101), so total BitmapFactory + ImageDecoder
concurrency is 4 per loader regardless of which decoder each request picks.

`BitmapRegionDecoder` is serialized by the platform. AOSP: `private final Object
mNativeLock` — "ensures that the native decoder object exists and that only one
decode can occur at a time" (BitmapRegionDecoder.java:43-45); `decodeRegion`,
`getWidth`, `getHeight`, and `recycle` all synchronize on it. **One decoder
instance = serialized tiles.** SSIV's read-write lock exists for an older reason,
and its comment is the best argument in the codebase for reading AOSP before
threading anything:

> Before SDK 21, BitmapRegionDecoder was not synchronized internally. Any
> attempt to decode regions from multiple threads with one decoder instance
> causes a segfault.

(SkiaImageRegionDecoder.java:153-157). Hence: read lock on 21+ (concurrency is
harmless because the platform serializes anyway), write lock below 21, and
`recycle()` takes the write lock so recycling blocks until no tile is mid-decode.
To get actual parallelism, SSIV ships `SkiaPooledImageRegionDecoder` — a *pool* of
`BitmapRegionDecoder` instances, one per tile thread, growing lazily against
file-size/core-count/low-memory limits with a hard cap of 4, and marked "highly
experimental" (lines 40-57).

### 2.10 Region decoding and tiling: when the image can't be a bitmap

At 8160 px wide a 50 MP photo exceeds the common 4096 GL texture limit; a
panorama exceeds it by more. These images cannot be drawn as a single
hardware-accelerated bitmap at any resolution — see Part 5's arithmetic. The
answer is `BitmapRegionDecoder` and a tile pyramid, and
[subsampling-scale-image-view](https://github.com/davemorrissey/subsampling-scale-image-view)
is the reference implementation (last commit 2020-12-30 — the problem stopped
changing).

The sizing logic (`calculateInSampleSize`, lines ~1376-1404) inverts Glide's
choice: requested dims = source × scale; `heightRatio = round(sHeight/reqHeight)`,
`widthRatio = round(sWidth/reqWidth)`; take the **smaller** ratio "to guarantee a
final image with both dimensions larger than or equal to the requested", then
round down to a power of two. It returns 32 when the view hasn't been measured yet
— a sane panic value.

The pyramid (`initialiseTileMap`, lines 1484-1529) is a
`LinkedHashMap<sampleSize, List<Tile>>` from `fullImageSampleSize` halving down to
1. Per level, the grid subdivides until each tile's decoded size fits
`maxTileDimensions` (device texture limits, or `setMaxTileSize`) and is no more
than 1.25× the view size. Each `Tile` carries its source-space `sRect` and its
sample size. Two decisions in there are worth stealing:
`initialiseBaseLayer` decodes the base one level *sharper* than needed
(`if (fullImageSampleSize > 1) fullImageSampleSize /= 2`, lines 1257-1259), and
base-layer tiles are **never discarded** on zoom-out — higher-res tiles are freed
when they scroll out of view, but there is always something to draw. If the whole
image fits one tile at sample size 1, SSIV skips the machinery and decodes the
bitmap normally (line 1262).

Format support is the catch, and it moved:
`BitmapRegionDecoder`'s javadoc said "Currently only the JPEG and PNG formats are
supported" through Android 11, and "Currently only the JPEG, PNG, **WebP and HEIF**
formats are supported" from Android 12 / API 31 (the `newInstance(ParcelFileDescriptor)`
overload landed in the same release). So on Android 11 and below, tiling a HEIC is
not an option — `newInstance` throws IOException "if the image format is not
supported or can not be decoded". SSIV's own wiki adds the older warnings: "you
may see problems displaying CMYK JPGs, and grayscale PNGs, especially on older
devices"; no GIF; and it "automatically falls back to BitmapFactory when the image
does not need to be subsampled". Its default `inPreferredConfig` for tiles is
**RGB_565** — halving tile memory on the assumption that photos are opaque.

Region decode does support `inBitmap` (since Jelly Bean): "BitmapRegionDecoder
will draw its requested content into the Bitmap provided, clipping if the output
content size (post scaling) is larger" (BitmapFactory.java:95-104). Fresco uses
exactly that, with `bitmapToReuse.reconfigure(targetWidth, targetHeight, config)`
(DefaultRawBitmapDecoder, lines ~285-292).

### 2.11 What the other libraries teach

**Fresco** is the archaeology. Its `DalvikPurgeableDecoder` documents the era
before ART:

> Base class for bitmap decodes for Dalvik VM (Gingerbread to KitKat). Native
> code used by this class is shipped as part of libimagepipeline.so

with `inPurgeable = true` — "Decode the image into a 'purgeable' bitmap that lives
on the ashmem heap" — plus `inInputShareable`, `inDither`, `inMutable` (lines
211-229). Purgeable decode is *lazy*; `pinBitmap()` forces it and locks the ashmem
pages so they can't be purged mid-draw: "Real decoding happens here - if the image
was corrupted, this will throw an exception" (lines 165-198). The failure mode gets
its own warning — bitmaps left unpinned "simply eat up ashmem memory and eventually
lead to unfortunate crashes." The whole point, per frescolib.org: "On Android 4.x
and lower, Fresco puts images in a special region of Android memory" — dodging the
Dalvik heap and its GC. All obsolete since Lollipop (`inPurgeable` is a no-op on
21+), which is why `PlatformDecoderFactory` now just picks `OreoDecoder` on O+ and
`ArtDecoder` below, both BitmapFactory-based.

Telegram carries the same fossil in one line:
`opts.inPurgeable = Build.VERSION.SDK_INT < 21` followed by
`Utilities.pinBitmap(image)` (ImageLoader.java:1202-1204, 1430). Its sample-size
loop is charmingly direct — `do { sample *= 2; } while (sample * 2 < scaleFactor)`
(lines 1291-1299) — with `scaleFactor < 1.2f` clamped to 1, and a screen-relative
heuristic for media thumbs: `scaleFactor = (min(photoH, photoW) / screenSize) * 6`
(lines 1310-1323). No pool, no `inBitmap`; exact sizing is a post-decode
`Bitmaps.createScaledBitmap` (lines 1543-1563). It's the everything-by-hand
architecture, and it works.

Two Fresco features nobody else ships are worth knowing:

- **Progressive JPEG**, end to end. `ProgressiveJpegParser` tracks scan boundaries
  as bytes arrive and `DecodeProducer` re-decodes at configured scan numbers:
  "Scans of the image will be shown in the view as you download them. Users will
  see the quality of the image start out low and gradually become clearer." The
  docs are honest about the limit: "This is only supported for network images.
  Local images are decoded at once, so no need for progressiveness." Irrelevant to
  a gallery, decisive for a feed.
- **The EOI trick.** Incomplete JPEGs get an end-of-image marker appended
  (`TailAppendingInputStream(stream, EOI_TAIL)` when `!isJpegComplete`) so the
  codec produces the partial image instead of failing. The platform's equivalent
  is ImageDecoder's `OnPartialImageListener` — return true and you get the bitmap
  with undecoded lines blank instead of a `DecodeException` (SOURCE_EXCEPTION /
  SOURCE_INCOMPLETE / SOURCE_MALFORMED_DATA). Coil just always says yes:
  `onPartialImageListener { true }` (StaticImageDecoder.kt:87).

Fresco's native libraries are also worth situating, because the name misleads:
`native-imagetranscoder` bundles libjpeg (`jpeglib.h` + `transupp.h`) for
**lossless transforms of encoded data** — resize/rotate JPEG bytes without a
full decode cycle — and `static-webp` bundles libwebp for old OS versions. Primary
photo decoding is still platform Skia. Nobody ships their own photo decoder.

Fresco also has one behavior the others lack: decode failures with a non-8888
config **retry once with ARGB_8888** (`retryOnFail`, DefaultDecoder.java:85-103) —
an admission that RGB_565 decode support is not uniform in the wild.

## Part 3: Thumbnails

A 12 MP photo is a 46.5 MB bitmap (Part 5). A grid cell is 512 px. The gap
between those two numbers is what the thumbnail APIs exist to close, and there
are three of them, two of which are the same one.

### 3.1 `ContentResolver.loadThumbnail` (Q+): what actually happens

The modern call is `ContentResolver.loadThumbnail(uri, size, cancellationSignal)`.
It is not a lookup; it's a full pipeline behind a binder call.

It arrives at the provider as `openTypedAssetFile` with `opts` carrying
`ContentResolver.EXTRA_SIZE`. MediaProvider dispatches on that:

```java
final boolean wantsThumb = (opts != null) && opts.containsKey(ContentResolver.EXTRA_SIZE)
        && (mimeTypeFilter.startsWith("image/"));
```

([MediaProvider.java:9503-9505](https://android.googlesource.com/platform/packages/providers/MediaProvider/+/refs/heads/main/src/com/android/providers/media/MediaProvider.java)),
then `ensureThumbnail(uri, signal)` returns an `AssetFileDescriptor` (lines
9557-9560), routed by URI type to `mImage`/`mVideo`/`mAudioThumbnailer` (lines
9567-9612), the whole thing wrapped in `Trace.beginSection("MP.ensureThumbnail")`
— which means it shows up in systrace if you look.

`Thumbnailer.ensureThumbnail` (lines 8243-8300) is two paths:

1. **Cache hit**: open the existing thumbnail file read-only. Cheap.
2. **Cache miss**: create a temp file, generate via
   `ThumbnailUtils.createImageThumbnail(file, mThumbSize, signal)`, compress with
   `Bitmap.CompressFormat.JPEG, 90`, atomically `Os.rename` into place, return a
   dup'd read FD.

The cache is `<primary volume>/<Pictures|Movies|Music>/.thumbnails/<media-id>.jpg`
(`DIRECTORY_THUMBNAILS = ".thumbnails"`, line 434), and always on primary storage
— "Always save generated thumbnails to primary storage" (lines 8233-8240), so SD
card photos get their thumbnails written to internal storage.

The size is not yours to choose:

```java
// Reasonable thumbnail size is half of the smallest screen edge width
final int thumbSize = Math.min(metrics.widthPixels, metrics.heightPixels) / 2;
mThumbSize = new Size(thumbSize, thumbSize);
```

(lines 1422-1425). On a 1080p phone every MediaStore thumbnail on disk is ~540 px,
JPEG q90 — regardless of the `size` you passed to `loadThumbnail`. Your `size`
influences what you get back, not what gets cached.

Two behaviors worth knowing. Concurrent racers on the same cache miss **each
regenerate independently** (documented in the comment) — no coalescing, so a grid
scrolling fast over cold items does duplicate work. And invalidation is
whole-directory: a `.thumbnails/.database_uuid` file is compared against the DB
UUID, and on mismatch **every thumbnail under the directory is deleted**
(`ensureThumbnailsValid`, lines 1345-1385). A MediaProvider database rebuild
therefore silently converts your entire gallery back to the cache-miss path.
`CancellationSignal` is honored end to end — over binder, and checked at each step
inside ThumbnailUtils.

### 3.2 The legacy API is the modern API

`MediaStore.Images.Thumbnails.getThumbnail(cr, id, MINI_KIND, opts)` is deprecated
and still works, because on Q+ it converts the kind to a size and calls
`cr.loadThumbnail(uri, size, signal)` (MediaStore.java:2873-2891). The pre-Q
`thumbnails` / `videothumbnails` tables survive only for stale-row cleanup
(MediaProvider.java:8218-8222).

The constants are `ThumbnailConstants` (MediaStore.java:2718-2737):

| Kind | Size |
|---|---|
| `MICRO_KIND` | 96×96 |
| `MINI_KIND` | 512×384 |
| `FULL_SCREEN_KIND` | 1024×**786** |

That 786 is not a typo in this report. It is a typo in AOSP, frozen into a public
constant, presumably forever.

This is why Telegram's picker still works: its cells resolve
`thumb://<mediaId>:<path>` to `MediaStore.Images.Thumbnails.getThumbnail(...,
MINI_KIND, opts)` (ImageLoader.java:1240-1243, 1466-1470), with a plain
`BitmapFactory.decodeStream(new FileInputStream(path))` fallback when the
thumbnail provider returns null (lines 1472-1499). Deprecated in 2019, load-bearing
in 2026, quietly redirected to the new implementation underneath.

### 3.3 `ThumbnailUtils`: the fast paths that make grids possible

`ThumbnailUtils.createImageThumbnail(File, Size, CancellationSignal)` is what
MediaProvider calls, and it's public API you can call yourself. Its order of
attempts is the interesting part
([ThumbnailUtils.java:272-305](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/media/java/android/media/ThumbnailUtils.java)):

1. **HEIF/HEIC/AVIF** → `MediaMetadataRetriever.getThumbnailImageAtIndex(-1, params,
   size.getWidth(), w*h)` — the *embedded* HEIF thumbnail image, via the media
   framework.
2. **EXIF thumbnail** → `exif.getThumbnailBytes()`; if present, decode those bytes
   with ImageDecoder. This never touches the main image. Camera JPEGs almost always
   carry one, typically ~160×120, and this is the difference between a grid that
   scrolls and one that doesn't.
3. **Fallback** → `ImageDecoder.decodeBitmap(createSource(file), resizer)` on the
   full file.

The `Resizer` (an `OnHeaderDecodedListener`) sets `ALLOCATOR_SOFTWARE` and
`setTargetSampleSize(max(width/targetW, height/targetH))` (lines 83-110) —
sample-size based, so it can overshoot the requested size; you get *at least* what
you asked for, possibly more.

Orientation is where paths 2 and 3 diverge, and it's the microcosm of 2.7:
orientation is read up front (lines 256-270) and applied manually via `Matrix`
**only** for the EXIF-thumbnail and MMR paths (lines 307-315) — the ImageDecoder
full-decode path returns already-rotated pixels, and there's a comment saying so
(line 303). Two rotation policies in one method, correctly.

`CancellationSignal.throwIfCanceled()` is checked before each expensive step (lines
248, 299) and inside the header callback (line 95) — a fast scroll can abandon work
mid-decode, which matters when step 3 is a 12 MP decode.

### 3.4 What libraries actually do for grids

Nothing special, mostly — and that's the finding.

Glide and Coil both point at the content URI and let their normal pipeline run:
bounds pass, `inSampleSize` to the cell size, decode. For a JPEG that's a
DC-only-ish IDCT (2.2) at 1/8 or 1/16 — often *faster* than the binder round-trip
plus JPEG-decode of a MediaStore thumbnail, and it doesn't depend on the thumbnail
cache being warm. Glide's `MemorySizeCalculator` sizes the memory cache at 2
screens' worth of ARGB_8888 (`widthPixels * heightPixels * 4`), which is exactly
the grid working set.

Telegram takes the other road (`MediaStore.Images.Thumbnails.getThumbnail`) because
its cells already have `_ID` and path from the picker cursor, and because it
projects `ORIENTATION` out of MediaStore — serving the EXIF rotation as a *column*,
so the grid never parses EXIF per file at all. That is a genuinely good trick and
it's available to anyone: `Images.Media.ORIENTATION` is right there in the
projection.

Fossify sidesteps the whole question by handing paths to Glide.

The one thing everyone avoids: **decoding the full image for a grid cell.** Part
5's arithmetic explains why, in one line — one 12 MP ARGB_8888 bitmap costs more
than forty 512×384 thumbnails.

## Part 4: Performance in the real world

The arithmetic in Part 5 bounds memory. This bounds time.

### 4.1 JPEG decode throughput

There is no authoritative modern per-device benchmark, so anchor it: libjpeg-turbo's
own ChangeLog reports NEON work that "sped up the decompression of full-color JPEGs
by nearly 2x on average on a Cavium ThunderX processor and by about 15% on average
on a Cortex-A53". Extrapolating from a 2011 measurement of 12 MP in ~1.7 s (≈7 MP/s)
on a Cortex-A9 with early NEON patches through roughly ten generations of core
improvements lands full-RGB decode in the ~50–150 MP/s range on a modern big core —
i.e. **a 12 MP JPEG full decode is roughly 80–250 ms** on a midrange phone core, and
a 1/8-scaled decode of the same file is about an order of magnitude cheaper (DC-only
IDCT over 1/64 the output pixels). *This is an estimate*, but it is the right order
of magnitude and it agrees with the measured Coil numbers in 2.1 (a ~1 MP JPEG at
~50 ms full / ~30 ms sampled).

The actionable form: **decode cost scales with output pixels, not file size**, as
long as the codec can subsample. Which brings us to the format where it can't.

### 4.2 HEIC: a different machine entirely

JPEG, PNG, and WebP decode in-process via Skia. HEIC does not. It goes through
`HeifDecoderImpl`
([frameworks/av/media/libheif/HeifDecoderImpl.cpp](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/av/media/libheif/HeifDecoderImpl.cpp)),
whose includes give the game away — `<media/mediametadataretriever.h>`,
`<binder/IMemory.h>`, `<binder/MemoryDealer.h>`, `<media/stagefright/MediaSource.h>`:

```cpp
sp<MediaMetadataRetriever> retriever = new MediaMetadataRetriever();
retriever->setDataSource(mDataSource, "image/heif");   // :350-351
...
retriever->getImageAtIndex(-1, mOutputColor);          // :635
```

`HeifDataSource` is a `BnDataSource` — a binder object proxying your app's stream to
the media process, with a 64 KB transfer buffer and a 4→64 MB rolling cache (lines
70-113). Results come back as a `VideoFrame` in binder shared memory. Grid HEICs
decode tile-row by tile-row when `mTileHeight >= 512` (lines 398-402).

So every HEIC decode is: binder round-trips + a MediaCodec HEVC decoder session +
YUV→RGB conversion. Three consequences:

1. **A fixed latency floor per image** — session setup dominates small images.
2. **Contention** — hardware codec instances are a limited system resource, and a
   grid scrolling fast wants many at once.
3. **Sample size doesn't help.** This is the one that surprises people. HEVC tiles
   must be fully decoded before anything can be scaled; there is no scaled-IDCT
   equivalent. The libjpeg-turbo trick that makes JPEG grids cheap has no analogue.

Which is exactly what [glide #4608](https://github.com/bumptech/glide/issues/4608)
("HEIF/HEIC images are loaded so SLOW", Glide 4.12.0, on Samsung S10/S20/Note and
Pixels) reports:

> It doesn't matter if I have default settings or `.override(50, 50)` ... the
> load speed stays the same for 2-3 seconds per image

`.override(50, 50)` on a JPEG is nearly free. On a HEIC it changes nothing, because
the expensive part already happened. Their logcat is full of
`E/HeifDecoderImpl: getSize: not supported!`, `decode: videoFrame is a nullptr`,
`E/MediaMetadataRetrieverJNI: getFrameAtTime: videoFrame is a NULL pointer`.
Coil hit the same code path via AVIF
([coil #1295](https://github.com/coil-kt/coil/issues/1295) — "getSize: not
supported! on some AVIF images"), because some platform versions route AVIF through
the HEIF decoder too.

2–3 seconds is a worst case under grid contention. A defensible per-image estimate on
midrange hardware is **~100–500 ms for HEIC versus ~20–60 ms for an equivalently
downsampled JPEG** — *estimate*, but the mechanism is verified and the direction is
not in doubt. And note the memory doesn't improve either: a 12 MP HEIC still produces
the same 46.5 MB RGBA output; the file is half the size, the bitmap is identical.

The practical rule: **if you control the capture format, JPEG decodes dramatically
faster than HEIC on Android**, and the storage saving is paid back in latency on
every single view.

### 4.3 `loadThumbnail` latency

From the mechanics in 3.1, the cost model is unambiguous even without a benchmark:

- **Warm**: binder call → open `Pictures/.thumbnails/<id>.jpg` → app decodes a
  ~540 px JPEG. Single-digit ms of decode plus IPC. Fine.
- **Cold**: binder call → full source decode (including HEIC-via-MediaCodec if the
  source is HEIF) → JPEG q90 *encode* → rename → *then* you get the fd → *then* you
  decode the result. Two decodes and an encode, serialized, before a single pixel
  reaches your grid.

First-population of a large gallery is therefore decode-bound and slow —
seconds-to-minutes for thousands of new items (*estimate; mechanics verified*), with
no coalescing between concurrent racers (3.1). This is the shape behind reports like
[coil #1337](https://github.com/coil-kt/coil/issues/1337): a gallery grid over
ContentResolver where "absolutely regular .jpg files ... load extremely slowly -
anywhere between 0.5 seconds and 5 seconds each" (Huawei P20 Pro, Android 10). See
also [fresco #155](https://github.com/facebook/fresco/issues/155) ("Slow loading from
MediaStore ContentProvider") and [fresco #2767](https://github.com/facebook/fresco/issues/2767),
which asks for `loadThumbnail` precisely because it beats `MediaMetadataRetriever` for
video thumbs.

The asymmetry is worth internalizing: `loadThumbnail` is a *cache-warming* API. When
it's warm it's the cheapest thing available. When it's cold it's the most expensive
thing available — more expensive than decoding the original yourself at
`inSampleSize = 8`, because that's a strict subset of what it does.

### 4.4 Query cost at scale

No authoritative public benchmark exists for MediaStore queries at 10k–100k rows.
The mechanics bound it: a cross-process cursor is a SQLite query in MediaProvider
plus rows marshalled through 2 MB `CursorWindow`s over binder (1.2). A narrow
projection — `_ID`, `DATE_TAKEN`, `MIME_TYPE`, `SIZE` — is on the order of tens of
bytes per row, so ~10–20k rows fit per window fill and a 100k-row scan is a handful
of fills: **commonly a few hundred milliseconds** on modern hardware (*estimate from
the 2,097,152-byte window size plus the issue anecdotes above*). Wide projections
that pull `DATA` and display names multiply the fills.

Which retroactively explains Part 1: Fossify and Telegram both fetch every row in one
pass and neither paginates, because at these numbers **enumeration is not the
bottleneck — decoding is**. The per-item `ContentResolver` round-trips that show up
in the bug reports above cost far more than the full-table scan ever did.


## Part 5: Performance arithmetic (computed for this report)

Decoded bitmap memory is `width × height × bytes-per-pixel` — 4 for
ARGB_8888, 2 for RGB_565, 8 for RGBA_F16 (hardware bitmaps are ~4 B/px
equivalent in gralloc memory, held off the Java heap). The following tables
were computed for this report from those documented constants; resolutions
are typical sensor outputs (verified in Part 6).

| Source | Pixels | ARGB_8888 | RGB_565 | RGBA_F16 |
|---|---|---|---|---|
| 12 MP 4:3 (4032×3024) | 12.2 M | **46.5 MB** | 23.3 MB | 93.0 MB |
| 12.5 MP 4:3 (4080×3060, Samsung binned) | 12.5 M | 47.6 MB | 23.8 MB | 95.3 MB |
| 50 MP 4:3 (8160×6120) | 49.9 M | **190.5 MB** | 95.3 MB | 381.0 MB |
| 200 MP 4:3 (16320×12240, Galaxy Ultra full-res) | 199.8 M | **762.0 MB** | 381.0 MB | 1.5 GB |
| 16:9 crop of 12 MP (4032×2268) | 9.1 M | 34.9 MB | 17.4 MB | 69.8 MB |
| FHD+ screenshot (1080×2340) | 2.5 M | 9.6 MB | 4.8 MB | 19.3 MB |
| QHD+ screenshot (1440×3120) | 4.5 M | 17.1 MB | 8.6 MB | 34.3 MB |
| Messenger-recompressed (1600×1200) | 1.9 M | 7.3 MB | 3.7 MB | 14.6 MB |
| Full-screen decode target (1080×810) | 0.9 M | 3.3 MB | 1.7 MB | 6.7 MB |
| Grid thumb (512×384, MINI_KIND size) | 0.2 M | 768 KB | 384 KB | 1.5 MB |

The `inSampleSize` ladder for a 12 MP source (ARGB_8888): 1 → 46.5 MB,
2 → 11.6 MB, 4 → 2.9 MB, 8 → 744 KB, 16 → 186 KB. A visible grid of ~40
thumbnails costs 10 MB at 256², 30 MB at 512×384 — i.e. **one decoded
full-resolution photo costs more than an entire thumbnail grid**, which is
the whole argument for never decoding full-size into a grid and for
downsampling to the view size in the detail view.

Texture-size ceiling: 50 MP (8160 px) exceeds the common 4096 GL texture
limit and 200 MP (16320 px) exceeds 8192 — such images cannot be drawn as a
single hardware-accelerated bitmap at full resolution at all; that is what
`BitmapRegionDecoder` tiling (Part 2) is for.

## Part 6: What is actually in a DCIM folder

Every performance decision above depends on assumptions about the input. Those
assumptions are mostly wrong, and the wrongness is systematic.

### 6.1 Dimensions

| Source | Resolution | Note |
|---|---|---|
| iPhone, default 12 MP | 4032×3024 | the canonical 4:3 |
| Pixel 8 | 4080×3072 | 50 MP sensor, "4-to-1 binning to produce 12.5MP resulting images"; **no full-res option** on the non-Pro |
| Galaxy S23 Ultra | 4000×3000 / 8160×6120 / 16320×12240 | "three shooting modes with exact resolutions"; the 200 MP mode is opt-in |
| Samsung 50 MP-sensor models (S23/S24 base) | 4080×3060 binned, 8160×6120 full | partially verified — dims consistent with sensor geometry |
| Front cameras | ~10–12 MP | S24 front 12 MP; Pixel 8 front 10.5 MP |
| 16:9 modes | 4032×2268, 4000×2252 | rows cropped from 4:3 sensor output — derived |
| Panoramas | 8k–16k+ px wide | exceed GPU per-dimension texture limits |

Two things stand out. First, **binning means the 200 MP phone produces 12 MP
files by default** — the Galaxy Ultra's 16320×12240 mode is a deliberate user
choice, and GSMArena notes Samsung's 16-to-1 binning "should result in a 12.5MP
image, however, Samsung seems to be cropping off half a megapixel to reach an
even 12MP". A 200 MP file exists (a Flickr sample runs 35.5 MB at 16320×12240) and
your gallery must not fall over on it, but it is rare. Second, panoramas break
things that megapixel counts don't: a 16k-px-wide strip exceeds every common
`GL_MAX_TEXTURE_SIZE` long before it approaches any byte-count limit.

### 6.2 Formats, and why "screenshots are PNG" is wrong

- **Samsung camera**: JPEG by default. HEIF is an opt-in toggle ("High efficiency
  pictures", Camera settings → Advanced picture options), default off — and
  community threads report updates occasionally flipping it on. So Samsung DCIM
  folders are JPEG with a seam of HEIC through them.
- **iPhone imports**: HEIC by default since iOS 11, arriving in Android galleries
  by cable, AirDrop-equivalents, and "send as document" transfers.
- **Pixel**: JPEG default; on Pixel 8+/Android 14, **Ultra HDR** JPEGs — gainmap
  embedded, still `.jpg`, backward compatible
  ([HDR image format](https://developer.android.com/media/platform/hdr-image-format)).
  BitmapFactory extracts the gainmap automatically via
  `codec->getGainmapAndroidCodec(...)` and a *second* decode at the same sample
  size (BitmapFactory.cpp:609-620) — so an Ultra HDR photo costs two decodes and
  carries extra memory you didn't ask for. ImageDecoder does the same unless a
  postProcessor or the low-RAM policy is set (ImageDecoder.cpp:362-371). This is
  also why Glide stopped rotating into pooled bitmaps (2.7): the pool doesn't
  preserve gainmaps.
- **Screenshots**: Pixel is PNG only, with no format option. **Samsung One UI
  defaults to JPG**, with PNG available under Settings → Advanced features →
  Screenshots. "Screenshots are PNG" is reliably true on Pixels and false on the
  most popular Android phones on earth — which matters, because PNG has no
  scaled-decode path (2.3: always floor, no fractional IDCT) and the Coil
  benchmark in 2.1 shows a 1 MP PNG costing 4× a 1 MP JPEG.
- **WhatsApp**: re-encodes to JPEG at ~1600 px longest side, ~70–100 KB typical;
  "HD" preserves larger (reported up to ~4096 px); "send as document" bypasses
  recompression entirely (*secondary sources; figures vary by version*).
- **Telegram** — verified from source: standard send scales to **1280 px, JPEG
  quality 87**; high-quality to **2560 px, quality 99**. `AndroidUtilities.getPhotoSize()`
  returns `photoSize = 1280` / `highQualityPhotoSize = 2560`
  (AndroidUtilities.java:3014-3026), used from MediaController.java:808 as
  `highQuality ? 99 : 87`.

### 6.3 File sizes

| Content | Size |
|---|---|
| 12 MP JPEG | ~2–4 MB |
| 12 MP HEIC | ~1.5–3 MB (roughly half the JPEG — *secondary sources*) |
| 200 MP JPEG | ~30–40 MB (35.5 MB sample) |
| PNG screenshot | ~0.5–1.5 MB at 1080p, ~2–3 MB at 1440p (*estimate*) |
| Motion photo | ~8–25 MB (*estimate* — see below) |

### 6.4 Motion photos: JPEGs with a video glued on

Google formalized what Samsung and others were already doing:

> Motion Photo files consist of a primary still image file (JPEG, HEIC, or AVIF)
> with a secondary video file appended to it
> — [motion photo format](https://developer.android.com/media/platform/motion-photo-format) v1.0

located via Container XMP directory entries. Samsung's older variant is a complete
JPEG, a 16-byte `MotionPhoto_Data` marker, then a complete MP4.

Decoding works fine — BitmapFactory and ImageDecoder stop at the JPEG EOI marker
and never see the tail. The damage is elsewhere: (a) file size is
**video-dominated**, so a "photo" is often 8–25 MB with 1–3 s of ~1080p video
appended; (b) any strategy that reasons from file length — "load the whole file
into a ByteBuffer", partial-read heuristics, size-based cache budgeting — pays for
the video; (c) EXIF/XMP parsers that scan the whole file get slower. Telegram's
picker parses XMP for exactly the **first 15 photos** to detect motion photos and
stops — an explicit budget on a per-file cost.

## Part 7: The known-issue catalogue

Everything above, compressed into the list you'd want taped to your monitor.

**HEIC decode is 5–25× slower than JPEG and downsampling doesn't help.**
Media-framework path: MediaMetadataRetriever + binder + a MediaCodec HEVC session
per image; HEVC has no scaled-IDCT equivalent
([glide #4608](https://github.com/bumptech/glide/issues/4608),
[coil #1295](https://github.com/coil-kt/coil/issues/1295)). *Fix*: capture JPEG if
you control capture; otherwise cache aggressively and never expect `.override()` to
save you.

**EXIF orientation has no uniform rule.** BitmapFactory ignores it; ImageDecoder
applies it — except for raw, where it doesn't
([immich #29337](https://github.com/immich-app/immich/pull/29337)). HEIF orientation
lives in `irot`/`imir`, and MIAF says EXIF orientation "shall be ignored by a MIAF
renderer" ([immich #27820](https://github.com/immich-app/immich/pull/27820)) — so
apps that read EXIF *and* let the platform rotate double-rotate HEICs. Glide's own
parser doesn't read HEIF/PNG/WebP orientation at all
([glide #4265](https://github.com/bumptech/glide/issues/4265); fix: androidx
`exifinterface` ≥ 1.2.0). *Fix*: pick one authority per decode path and know which.

**EXIF rotation silently costs you the hardware bitmap.** Glide vetoes hardware
config when orientation is required (HardwareConfigState.java:140-145); Coil
downgrades to software (BitmapFactoryDecoder.kt:95-98). Your camera photos are
precisely the affected set.

**Hardware bitmaps throw at runtime, far from the decode.** `getPixel`,
`getPixels`, `getColor`, `copyPixelsTo/FromBuffer` → IllegalStateException;
software-Canvas draw → IllegalStateException at draw time (breaking view-hierarchy
screenshots — use `PixelCopy`); mutable+HARDWARE → IAE. And
`ImageDecoder.decodeBitmap` returns one **by default**.

**Hardware bitmaps exhaust file descriptors.** ~2 FDs each against a 1024 rlimit on
O/O_MR1 (32768 on 28+), producing "graphics errors, Binder errors, and a variety of
crashes". Glide: P+ only, `/proc/self/fd` counted every 50 decodes, 20,000 cap,
500 on named OnePlus P devices (b/139097735). Coil: FD-counting on 26/27 with an
800 cap, plus a denylist covering **every Samsung `SM-` device on API 26**. Pre-Q,
Glide additionally blocks hardware bitmaps until the GL context exists
(b/126573603#comment12). *Fix*: if you write your own decode path on API 26/27,
don't use hardware bitmaps.

**`inBitmap` failures are indistinguishable from corrupt input.** Any error with
`inBitmap != null` throws IAE — including a too-small pooled bitmap
(BitmapFactory.java:635 et al.); an immutable one is silently ignored with a
logcat warning (BitmapFactory.cpp:448-458). *Fix*: Glide's pattern — catch IAE,
drop the pooled bitmap, retry without reuse. Or don't pool; Coil deleted its pool
in 2.0 and Glide cut its O+ budget to one screen.

**`Canvas: trying to draw too large bitmap`.** Pre-P,
`DisplayListCanvas.MAX_BITMAP_SIZE = 100 * 1024 * 1024`. Android 13 made it
`max(ro.hwui.max_texture_allocation_size, 100 MB)`. Current main raises the default
to **150 MB and exempts HARDWARE bitmaps**:
`if (bitmap.getConfig() != Bitmap.Config.HARDWARE && bitmapSize > MAX_BITMAP_SIZE) throw ...`
(RecordingCanvas.java:42-49, 262-269). At 100 MB the ceiling is ~26.2 MP ARGB_8888 —
a 64 MP 9248×6936 shot is 256.6 MB and *always* threw
([picasso #2255](https://github.com/square/picasso/issues/2255) reports exactly
"too large(256576512bytes)").

**GPU texture limits are a separate, lower ceiling.** Commonly 4096/8192/16384 px
per dimension. A software bitmap over the limit logs "Bitmap too large to be
uploaded into a texture" and simply doesn't render. Panoramas hit this long before
any byte-count limit. *Fix*: `BitmapRegionDecoder` tiling (2.10).

**`Row too big to fit into CursorWindow`.** The window is a fixed 2 MB shared-memory
region (`config_cursorWindowSize`); a single oversized row throws
`SQLiteBlobTooBigException`, and you cannot resize MediaStore's window from the
client. *Fix*: narrow projections, never blob columns. (*MediaStore-specific public
reports are thin; the generic 2 MB behavior is verified and the MediaStore
application is inference.*)

**`DATE_TAKEN` is unreliable and mis-united.** Milliseconds
(`@CurrentTimeMillisLong`) while `DATE_ADDED`/`DATE_MODIFIED` are seconds —
1000× sort corruption. The official doc concedes images "must define both
`ExifInterface#TAG_DATETIME_ORIGINAL` and `TAG_OFFSET_TIME_ORIGINAL` to reliably
determine this value in relation to the epoch". Fossify ships a user-facing "Fix
Date Taken value" action and its own `DateTakensDao` because the platform's values
are wrong often enough
([Fossify #792](https://github.com/FossifyOrg/Gallery/issues/792), also #926, #775).
*Fix*: fall back to `DATE_MODIFIED * 1000` (Fossify) or don't use `DATE_TAKEN` at
all on Q+ (Telegram).

**Motion photos make files 3–10× larger than the image in them.** Size-based
heuristics lie (6.4).

**Ultra HDR costs a second decode.** Gainmap extraction is automatic and on by
default (6.2).

**`loadThumbnail` on a cold cache is the most expensive call in your app**, not the
cheapest: full source decode + JPEG q90 encode + rename, with no coalescing between
concurrent callers, and the on-disk size is fixed at half your smallest screen edge
regardless of what you requested (3.1).

**A `ContentObserver` fires on the looper you registered it with.** Register with a
main-looper `Handler`, re-query inline, and you've put a binder call plus a SQLite
query on the main thread — and a downstream `flowOn(Dispatchers.IO)` will not save
you (1.5).

## Part 8: What to do for a read-only own-photos gallery

The motivating case: a CameraX companion app with an informational, read-only
gallery showing only the photos it captured itself, into `Pictures/Camera/`. Most
of this report is a catalogue of problems that case doesn't have — which is the
point. Enumerate the constraints and most of the decision tree collapses.

**Enumeration** (Part 1). Query `Images.Media.EXTERNAL_CONTENT_URI` with a
`RELATIVE_PATH = 'Pictures/Camera/'` selection, a narrow projection, and Bundle
query args for sort and limit — never the `sortOrder`-with-LIMIT hack, which throws
on Android 11+ for apps targeting R (1.2). Content URIs only; no `DATA`, no `File`.
No storage permission is needed to read media you contributed on Android 10+ — but
**request `READ_MEDIA_IMAGES` anyway**, because ownership does not survive
uninstall/reinstall, after which your own old photos become invisible (1.5). Include
`ORIENTATION` in the projection: it's free, and it saves an EXIF parse per grid cell
(3.4). Refresh with a `ContentObserver` registered on a **background** `Handler`
(1.5). If you ever need sync semantics, use `MediaStore.getGeneration()`, not dates
(1.7).

**Decoding** (Part 2). Use **Coil with default settings.** On API 29+ that means
`StaticImageDecoder` → `ImageDecoder` → arbitrary target size in one fused
subsampled-plus-bilinear pass, native EXIF handling, hardware bitmaps where the
platform allows; below 29 it falls back to `BitmapFactoryDecoder` with the
density-trick exact scaling and `ExifUtils`. Coil's `RESPECT_PERFORMANCE` EXIF
strategy is correct for a gallery of camera JPEGs. The decision here is not "Coil vs
Glide" on merits — both are excellent and Glide's `Downsampler` is the more
educational read — it's that **the correct amount of custom decode code for this app
is zero**. Every hand-rolled path in this report (Telegram's, Fresco's) exists to
serve a constraint you don't have.

Two settings worth knowing you *could* touch and probably shouldn't:
`bitmapFactoryMaxParallelism` (default 4, shared across both decoders — 2.9), and
`allowRgb565`, which would halve grid memory and is unnecessary given the arithmetic
below.

**Capture format.** If the app writes the files, **write JPEG**. HEIC halves storage
and pays it back with interest on every view: media-framework round-trip, MediaCodec
session, no scaled decode, identical bitmap memory (4.2). For an app whose entire
gallery is its own output, this is the single highest-leverage decision in the
report, and it belongs in the capture code, not the gallery.

**Thumbnails.** Don't call `loadThumbnail`. Point Coil at the content URI and let
`inSampleSize`/`setTargetSize` do the work — a 1/8 JPEG decode beats a cold
thumbnail generation (which is a full decode *plus* a q90 encode, 4.3) and doesn't
depend on a cache that a MediaProvider DB rebuild can wipe wholesale (3.1). The
platform's thumbnail cache is a fine thing to benefit from and a poor thing to
depend on.

**Region decoding.** Not needed. A 12 MP capture downsampled to screen size is 3.3 MB
(Part 5), and unless the app offers a full-res 200 MP mode there is no image that
exceeds the texture limit. If a 50 MP mode ever ships, revisit: 8160 px already
exceeds the common 4096 texture limit and the detail view would need
`SubsamplingScaleImageView` or equivalent (2.10).

**Memory.** Part 5 is the whole argument: a visible grid of ~40 thumbnails is
~10–30 MB; one full-resolution 12 MP decode is 46.5 MB. Never decode full-size into
a grid, always downsample to view size in the detail view, and the budget takes care
of itself.

**EXIF for the detail panel.** `ExifInterface` over
`contentResolver.openInputStream(uri)`, off the main thread, lazily per photo.
Note the shape of the problem: this is read-for-metadata, unrelated to the decode
path, and the two must not be conflated — the decoder's orientation handling is the
platform's business (2.7), and re-applying rotation because you also read
`TAG_ORIENTATION` for the panel is precisely how apps end up with sideways photos.

The summary: **for this app, the entire decode layer is `AsyncImage(model = uri)`,
and every interesting decision moved somewhere else** — into the query shape, the
observer's looper, and the capture format. That is what the two years of scar tissue
in Glide's `Downsampler` bought everyone.


## Sources

- [FossifyOrg/Gallery](https://github.com/FossifyOrg/Gallery) — `MediaFetcher.kt`, `Context.kt`, `GetMediaAsynctask.kt` (default branch, read 2026-07-17)
- [DrKLO/Telegram](https://github.com/DrKLO/Telegram) — `MediaController.java`, `ImageLoader.java`
- AOSP MediaProvider — [MediaProvider.java](https://android.googlesource.com/platform/packages/providers/MediaProvider/+/refs/heads/main/src/com/android/providers/media/MediaProvider.java), [FileUtils.java](https://android.googlesource.com/platform/packages/providers/MediaProvider/+/refs/heads/main/src/com/android/providers/media/util/FileUtils.java), [ModernMediaScanner.java](https://android.googlesource.com/platform/packages/providers/MediaProvider/+/refs/heads/main/src/com/android/providers/media/scan/ModernMediaScanner.java), [MediaStore.java](https://android.googlesource.com/platform/packages/providers/MediaProvider/+/refs/heads/main/apex/framework/java/android/provider/MediaStore.java)
- Android docs — [Android 11 storage](https://developer.android.com/about/versions/11/privacy/storage), [shared media](https://developer.android.com/training/data-storage/shared/media), [photo picker](https://developer.android.com/training/data-storage/shared/photopicker), [partial access](https://developer.android.com/about/versions/14/changes/partial-photo-video-access), [Android 13 changes](https://developer.android.com/about/versions/13/behavior-changes-13)
- Issue trackers — [media_gallery #19](https://github.com/aloisdeniel/media_gallery/issues/19) / [PR #24](https://github.com/aloisdeniel/media_gallery/pull/24/files), [Gligar #20](https://github.com/OpenSooq/Gligar/issues/20), [SAF grant limit](https://issuetracker.google.com/issues/149315521), [Simple-Gallery #1022](https://github.com/SimpleMobileTools/Simple-Gallery/issues/1022), [Fossify Gallery #140](https://github.com/FossifyOrg/Gallery/issues/140), [aves #1047](https://github.com/deckerst/aves/issues/1047), [storage-samples #25](https://github.com/android/storage-samples/issues/25)

Decode layer (Parts 2–4, 6–8):

- [bumptech/glide](https://github.com/bumptech/glide) (master, read 2026-07-17) — `Downsampler.java`, `HardwareConfigState.java` (+ v4.9.0/v4.11.0 tags for the O-vs-P history), `DefaultImageHeaderParser.java`, `TransformationUtils.java`, `MemorySizeCalculator.java`, `LruBitmapPool.java`, `LruArrayPool.java`; docs: [hardware bitmaps](https://bumptech.github.io/glide/doc/hardwarebitmaps.html)
- [coil-kt/coil](https://github.com/coil-kt/coil) (main @ c3c2e1d9, 2026-07-15) — `BitmapFactoryDecoder.kt`, `StaticImageDecoder.kt`, `DecodeUtils.kt`, `ExifUtils.kt`, `ExifOrientationStrategy.kt`, `hardwareBitmaps.kt`, `RealImageLoader.android.kt`, `imageLoaders.android.kt`, `AnimatedImageDecoder.kt`
- [davemorrissey/subsampling-scale-image-view](https://github.com/davemorrissey/subsampling-scale-image-view) (master @ 173e421f, last commit 2020-12-30) — `SubsamplingScaleImageView.java`, `SkiaImageRegionDecoder.java`, `SkiaPooledImageRegionDecoder.java`; [wiki: Displaying images](https://github.com/davemorrissey/subsampling-scale-image-view/wiki/02.-Displaying-images)
- [facebook/fresco](https://github.com/facebook/fresco) (main @ 7166ff48, 2026-07-14) — `PlatformDecoderFactory.kt`, `DefaultRawBitmapDecoder.kt`, `DefaultDecoder.java`, `DalvikPurgeableDecoder.kt`, `ProgressiveJpegParser`; docs: [progressive JPEGs](https://frescolib.org/docs/progressive-jpegs.html)
- [DrKLO/Telegram](https://github.com/DrKLO/Telegram) — `ImageLoader.java`, `AndroidUtilities.java` (`getPhotoSize`), `MediaController.java`
- AOSP graphics — [BitmapFactory.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/graphics/java/android/graphics/BitmapFactory.java), [BitmapFactory.cpp](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/libs/hwui/jni/BitmapFactory.cpp), [ImageDecoder.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/graphics/java/android/graphics/ImageDecoder.java), [ImageDecoder.cpp](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/libs/hwui/jni/ImageDecoder.cpp), [hwui/ImageDecoder.cpp](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/libs/hwui/hwui/ImageDecoder.cpp), [Bitmap.java](https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/graphics/java/android/graphics/Bitmap.java), [BitmapRegionDecoder.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/graphics/java/android/graphics/BitmapRegionDecoder.java), [RecordingCanvas.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/graphics/java/android/graphics/RecordingCanvas.java) (+ `DisplayListCanvas.java` at android-8.1.0_r1 / 9.0.0_r1 for the historical 100 MB limit), [ThumbnailUtils.java](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/media/java/android/media/ThumbnailUtils.java)
- AOSP media + Skia — [HeifDecoderImpl.cpp](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/av/media/libheif/HeifDecoderImpl.cpp), [SkJpegCodec.cpp](https://skia.googlesource.com/skia/+/refs/heads/main/src/codec/SkJpegCodec.cpp), [libjpeg-turbo ChangeLog](https://github.com/libjpeg-turbo/libjpeg-turbo/blob/main/ChangeLog.md)
- Android docs — [manage bitmap memory](https://developer.android.com/topic/performance/graphics/manage-memory), [Ultra HDR image format](https://developer.android.com/media/platform/hdr-image-format), [motion photo format](https://developer.android.com/media/platform/motion-photo-format)
- Decode issue trackers — [coil #1943](https://github.com/coil-kt/coil/issues/1943) (the ImageDecoder-vs-BitmapFactory benchmark), [coil #1295](https://github.com/coil-kt/coil/issues/1295), [coil #1337](https://github.com/coil-kt/coil/issues/1337), [coil #250](https://github.com/coil-kt/coil/issues/250), [coil #2434](https://github.com/coil-kt/coil/issues/2434), [glide #4608](https://github.com/bumptech/glide/issues/4608), [glide #4265](https://github.com/bumptech/glide/issues/4265), [glide #738](https://github.com/bumptech/glide/issues/738), [fresco #155](https://github.com/facebook/fresco/issues/155), [fresco #2767](https://github.com/facebook/fresco/issues/2767), [picasso #2255](https://github.com/square/picasso/issues/2255), [immich #29337](https://github.com/immich-app/immich/pull/29337), [immich #27820](https://github.com/immich-app/immich/pull/27820), [Fossify Gallery #792](https://github.com/FossifyOrg/Gallery/issues/792), [element-android #7184](https://github.com/element-hq/element-android/pull/7184)
- Image shapes — [GSMArena: Pixel 8 review](https://www.gsmarena.com/google_pixel_8-review-2628p5.php), [GSMArena: Galaxy S23 Ultra review](https://www.gsmarena.com/samsung_galaxy_s23_ultra-review-2539p5.php), [200 MP sample file](https://www.flickr.com/photos/janitors/52662555387), [Samsung HEIF toggle thread](https://us.community.samsung.com/t5/Galaxy-S23/Pictures-switched-to-HEIC-how-can-I-get-it-back-to-JPEG/td-p/2653534), [One UI screenshots default to JPG](https://www.androidpolice.com/2020/04/19/how-to-save-screenshots-as-png-on-samsung-galaxy-phones/), [Pixel screenshots are PNG only](https://www.digitalcitizen.life/change-screenshots-file-android-png-jpg/), [Ultra HDR on Pixel 8](https://www.androidpolice.com/android-14-ultra-hdr-hands-on/), [sm_motion_photo (Samsung layout)](https://github.com/g0ddest/sm_motion_photo)
- Benchmarks and anchors — [Benchmarking image loading libraries on Android](https://medium.com/swlh/benchmarking-image-loading-libraries-on-android-3ddf365a1927) (Colin White, Pixel 1, 2020), [faster JPEG decoding on ARM with libjpeg-turbo](https://www.cnx-software.com/2011/09/13/faster-jpeg-decoding-on-arm-with-libjpeg-turbo/), [optimizing turbo-jpeg for ARM64](https://miggs125.wordpress.com/2019/12/01/optimizing-turbo-jpeg-for-arm64/)

Claims marked *estimate* in the body are labelled as such at the point of use: the
libjpeg-turbo throughput extrapolation (4.1), HEIC per-image latency (4.2),
cold-`loadThumbnail` and query-at-scale costs (4.3, 4.4), motion-photo and
screenshot file sizes (6.3), and the CursorWindow-for-MediaStore application (Part 7).
