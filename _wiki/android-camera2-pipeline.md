---
share: true
title: Android Camera2 pipeline and CameraX interop
categories:
  - Wiki
tags:
  - camera2
  - camerax
  - android
  - computational-photography
author: claude
type: concept
created: 2026-07-18
updated: 2026-07-18
sources:
  - "[[2026-07-18-hardcore-android-camera-engineering]]"
aliases:
  - camera2
  - camerax
  - camera2-interop
---

# Android Camera2 pipeline and CameraX interop

How Android's camera stack is controlled below the convenience layer, and where
CameraX (the policy layer) hides Camera2 and lets you reach back down. Companion
to [Computational photography: multi-frame merge](/wiki/computational-photography-multiframe/) (what to do with the frames) and
[Android image decoding](/wiki/android-image-decoding/) (the bitmap-memory arithmetic reused below).

## The layer cake

`Your UseCases + Camera2Interop → CameraX core → Camera2 (CaptureRequest +
CameraCaptureSession) → HAL3 + vendor ISP → sensor`. CameraX picks a legal
stream combination and drives the session; Camera2 owns the requests and the
session; the HAL owns multi-frame merge. The single most load-bearing spec
artifact is the **stream-combination table** — it applies at the Camera2
session layer and governs everything above it.

## Stream-combination tables (the contract)

The AOSP `CameraDevice.createCaptureSession` javadoc guarantees, per hardware
level, which output targets may be configured at once. Format tokens: `PRIV`
(opaque, `getOutputSizes(Class)`), `YUV` (`YUV_420_888`), `JPEG`, `RAW`
(`RAW_SENSOR`); size tokens: `PREVIEW` (≤1080p), `RECORD` (max recording),
`MAXIMUM` (max output).

- **LEGACY**: ≤3 targets, no RAW; floor is `PRIV PREVIEW + JPEG MAXIMUM`.
- **LIMITED**: adds high-res `RECORD` combinations (video snapshot).
- **FULL**: adds `MAXIMUM`-res processing concurrent with preview.
- **RAW capability** (independent, possible on LIMITED): adds
  `PRIV PREVIEW + JPEG MAXIMUM + RAW MAXIMUM` (simultaneous JPEG + DNG).
- **LEVEL_3**: adds a 4th target
  (`PRIV PREVIEW + PRIV 640×480 + YUV/JPEG MAXIMUM + RAW MAXIMUM`).

Queryable at runtime via `MandatoryStreamCombination` (never hard-code). Two
practical facts from the same doc: exceeding a table may still work but forfeit
the frame-rate guarantee (or fail via `onConfigureFailed`); and session
(re)configuration takes "several hundred milliseconds" — the reason to
reconfigure rarely.

## Capabilities gate features, not hardware levels

- **RAW** ⇐ `REQUEST_AVAILABLE_CAPABILITIES` contains `RAW`.
- **Manual sensor** (ISO/shutter/focus-distance) ⇐ the `MANUAL_SENSOR`
  *capability* — **not** "FULL." All FULL devices advertise it, but a LIMITED
  device may too. "Manual = FULL" is a common but imprecise claim.
- Hardware-level constants are **not monotonically ordered** (`LEGACY=2,
  LIMITED=0, FULL=1, LEVEL_3=3`), so a numeric `>=` comparison is a bug — branch
  on equality (Open Camera does).

## RAW / DNG

`DngCreator(CameraCharacteristics, CaptureResult)` needs the `CaptureResult`
**from the same capture** that produced the `RAW_SENSOR` buffer (timestamp,
black/white levels, color transform, neutral point, noise profile, lens shading
come from it); `writeImage(OutputStream, Image)` validates `RAW_SENSOR`. Open
Camera's `CameraController2`/`ImageSaver` write straight to the destination
stream for RAW because a temp-file copy is slow and DNG needs no post-hoc EXIF
rewrite.

## Manual exposure/ISO/focus

Turn the auto pipeline off, then set sensor keys:
`CONTROL_AE_MODE_OFF` + `SENSOR_SENSITIVITY` + `SENSOR_EXPOSURE_TIME`;
`CONTROL_AF_MODE_OFF` + `LENS_FOCUS_DISTANCE` (diopters). The ranges
(`SENSOR_INFO_EXPOSURE_TIME_RANGE`, `SENSOR_INFO_SENSITIVITY_RANGE`,
`LENS_INFO_MINIMUM_FOCUS_DISTANCE`) **may be null on device** — probe and
degrade. `LENS_INFO_FOCUS_DISTANCE_CALIBRATION` (UNCALIBRATED/APPROXIMATE/
CALIBRATED) qualifies what the dioptre value means. Manual exposure conflicts
with vendor extensions and flash metering (an extension owns the pipeline) —
the reason a "Pro mode" is a distinct UI, not settings rows.

## Requests: templates, repeating vs. single

Start from a template (`TEMPLATE_PREVIEW/STILL_CAPTURE/RECORD` on all
backward-compatible devices; `TEMPLATE_MANUAL` needs `MANUAL_SENSOR`;
`TEMPLATE_ZERO_SHUTTER_LAG` needs a reprocessing capability). Viewfinder =
`setRepeatingRequest(...)`; still = one-shot `capture(...)`.

## Zero-shutter-lag via reprocessing

Stream full-res `PRIVATE` frames into a ring buffer continuously; on the
shutter, pick the frame closest to the button press and feed it *back* through
`createReprocessableCaptureSession` + `ImageWriter.queueInputImage` with a
`TEMPLATE_ZERO_SHUTTER_LAG` reprocess request. For PRIVATE buffers, `ImageWriter`
is the only way to return data to the HAL (zero-copy). Gated on
`PRIVATE_REPROCESSING`/`YUV_REPROCESSING`; dropped under flash-ON/AUTO and vendor
extensions. Ring size is an app choice: CameraX uses **3 frames**, Google's
`zsldemo` uses **10** and reports **50–100 ms** button-to-image on Pixel.

## CameraX interop (reaching down)

Package `androidx.camera.camera2.interop`, all `@ExperimentalCamera2Interop`:

- **`Camera2Interop.Extender`** — inject `CaptureRequest` keys at UseCase-build
  time.
- **`Camera2CameraControl`** — inject them at runtime via
  `setCaptureRequestOptions(CaptureRequestOptions)`; the returned
  `ListenableFuture` completes when the repeating `CaptureResult` shows the
  options submitted. You set options on the repeating request; you never own the
  builder or session. This is the seam a CameraX app uses for e.g. white-balance
  presets (`CONTROL_AWB_MODE`).
- **`Camera2CameraInfo.getCameraCharacteristic(KEY)`** — probe raw
  characteristics.

**CameraX 1.5 (Nov 2025)** added RAW to `ImageCapture`: probe
`getImageCaptureCapabilities().getSupportedOutputFormats()`, then choose
`OUTPUT_FORMAT_RAW` / `OUTPUT_FORMAT_RAW_JPEG` / `OUTPUT_FORMAT_JPEG_ULTRA_HDR`.
Before 1.5 there was no CameraX RAW path — the answer was "drop to Camera2."
Interop stays experimental and CameraX still owns stream management: you set
keys, not stream-combination tables.

## Pipeline performance

- **Buffers / backpressure**: if a consumer holds `maxImages` un-closed buffers,
  the producer "will eventually stall or drop Images" (AOSP `ImageReader`).
  `acquireLatestImage()` auto-drains the lag chain (preferred for real-time);
  `acquireNextImage()` returns the oldest. CameraX `ImageAnalysis`:
  `STRATEGY_KEEP_ONLY_LATEST` (default, 1-deep) vs `STRATEGY_BLOCK_PRODUCER`
  (deeper queue) — but blocking is device-scoped, so it stalls **preview** too
  when both are bound. Always `ImageProxy.close()`, never the wrapped
  `Media.Image.close()`.
- **Threading**: CameraX calls the blocking platform APIs (hundreds of ms IPC)
  only from background threads and serializes device access via
  `newSequentialExecutor`; callbacks run on whichever `Executor` you pass, so
  UI-touching callbacks need `mainThreadExecutor()`. Raw Camera2 uses a
  dedicated `HandlerThread`.
- **Memory**: format bytes/px — RAW16 = 2, YUV_420_888 = 1.5, ARGB_8888 = 4. A
  12 MP RAW16 frame is `4000×3000×2 = 24 MB` (22.9 MiB); burst memory is linear
  in frame count, so `maxImages` is the OOM knob. (Cf. the 12 MP ARGB = 46.5 MiB
  bitmap in [Android image decoding](/wiki/android-image-decoding/).)

## GPU after RenderScript

RenderScript is deprecated since API 31 and losing hardware support. Successors:
**Vulkan compute** (NDK/JNI, no SDK bindings), OpenGL ES 3.1 compute, **AGSL**;
`RenderEffect` (API 31) is a RenderNode/View compositing effect, not general
compute. `android/renderscript-intrinsics-replacement-toolkit` replaces the
intrinsics (blur/blend/colormatrix/resize/YUV→RGB) but is **CPU + SIMD only, no
GPU**, ~2× faster. The real OSS GPU pipeline is `cats-oss/android-gpuimage`
(`GPUImageRenderer` is both `GLSurfaceView.Renderer` and camera `PreviewCallback`,
native YUV→RGB, GLSL filter chain) — it owns its own `GLSurfaceView`, so it does
not compose cleanly with CameraX's `PreviewView`. YUV→RGB is a real cost
(~28 ms RenderScript / ~12 ms NEON on a Pixel 4A) and inflates the buffer 1.5→4
B/px.
