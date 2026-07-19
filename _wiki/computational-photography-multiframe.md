---
share: true
title: "Computational photography: multi-frame merge"
categories:
  - Wiki
author: claude
type: concept
created: 2026-07-18
updated: 2026-07-18
sources:
  - "[[2026-07-18-hardcore-android-camera-engineering]]"
aliases:
  - hdr-plus
  - night-sight
  - multi-frame-merge
---

# Computational photography: multi-frame merge

How modern phone cameras turn a burst of mediocre frames into one good photo,
what of it is open source, and the honest gap between the two. Companion to
[Android Camera2 pipeline and CameraX interop](/wiki/android-camera2-pipeline/) (the capture mechanism that feeds the merge).

## The one-line conclusion

"GCam-class results" are **called, not reimplemented.** Production HDR+ / Night
Sight / Super-Res Zoom are closed, run on the vendor ISP (Qualcomm Spectra) +
Tensor silicon, and use a frequency-domain merge, motion metering, and learned
finishing that no open-source Android app reproduces. On-device, the only real
lever is the vendor's own pipeline exposed through CameraX Extensions (scene
modes: Night, HDR, Bokeh).

## HDR+ (Hasinoff et al., SIGGRAPH Asia 2016)

A same-exposure burst of **2–8 raw frames**, deliberately underexposed and held
in a ZSL ring buffer (underexposure avoids highlight clipping and shortens each
frame to limit blur; denoising is at the Bayer/raw level). A sharp reference
frame near the shutter press is chosen; alternates are aligned to it; the merge
is **frequency-domain** — per-tile per-frequency interpolation weighted by
measured difference vs. modeled noise (Wiener-style shrinkage down-weighting
misaligned tiles).

## What open source actually reproduces

- **`timothybrooks/hdr-plus`** — a **Halide offline binary**, not an Android app
  (~3.4 s to merge 8 frames on a desktop i7). Alignment (`align.cpp`):
  hierarchical Gaussian pyramid, downsample ×4, 32-px Bayer tiles, ±4 search, L1
  metric, **no subpixel** refinement (production adds a bivariate-polynomial fit
  it omits). Finish (`finish.cpp`): Malvar demosaic → bilateral chroma denoise →
  sRGB → Laplacian-pyramid exposure-fusion tone map (Mertens-flavored, iterated
  3×) → S-curve → unsharp mask.
  - ⚠️ Correction to a widespread belief: the OSS **`merge.cpp` is
    spatial-domain (L1 non-local-means + raised-cosine blend), NOT the paper's
    frequency-domain Wiener filter** — Brooks explicitly simplified it away ("No
    2D DFT, no Wiener filter"). Do not cite `hdr-plus` as a faithful HDR+ merge.
- **Open Camera `HDRProcessor.java`** — the most capable multi-frame merge in a
  real OSS Android app, but a *simpler class*: **exposure bracketing**, not
  same-exposure raw denoise. Accepts exactly 1 or 3 8-bit bitmaps, aligns with
  **Median Threshold Bitmap** (Ward 2003), fits a per-channel linear response,
  tone maps (`CLAMP/EXPONENTIAL/REINHARD/FILMIC/ACES`), local contrast via
  **CLAHE**.
- **OpenCV `photo`** (`merge.cpp`) — the OSS classics: `createMergeMertens`
  (Exposure Fusion, PG 2007 — LDR directly, no tonemap), `createMergeDebevec`
  (radiance map, SIGGRAPH 1997) + `TonemapReinhard/Drago/Mantiuk`.
- **No OSS Android app** ships a real-time on-device Night-Sight-class low-light
  burst merge or multi-frame RAW super-resolution.

## Night mode (Liba et al., SIGGRAPH Asia 2019)

Extends HDR+ with **motion metering** (optical flow before the shutter sets
per-frame exposure/gain), switches ZSL → **positive-shutter-lag** for longer
exposures, and adds **learned auto white balance**. Vendor-blog numbers (not the
paper text): default HDR+ caps per-frame exposure at 66 ms to preserve ZSL;
Night Sight allows up to 15 handheld frames (~48–333 ms each, ≈1 s) scaling to
6×1 s ≈ 6 s on a tripod; operates to 0.3 lux.

## Super-resolution (Wronski et al., SIGGRAPH 2019)

Pixel "Super Res Zoom": **replaces demosaicing** (full RGB directly from a burst
of CFA raw frames), exploits **natural hand tremor** (~0.9 px, 1σ) as sub-pixel
sampling diversity, accumulates via kernel regression under a per-pixel
robustness mask. ~100 ms per 12 MP frame on mobile (project page).

## Where this attaches in a CameraX app

The pragmatic path is CameraX Extensions — binding the vendor's extension-enabled
camera selector for Night/HDR/Bokeh runs the vendor's multi-frame processing
(the same Spectra pipeline described above). The alternative — reimplementing
HDR+ on-device — is not viable in open source today. See
[Android Camera2 pipeline and CameraX interop](/wiki/android-camera2-pipeline/) for the ZSL ring buffer that feeds a merge.

The *learned* layer that rides on top of this merge — semantic segmentation
masks (sky/skin/person) guiding tone/denoise, scene classifiers steering ISP
tuning — is a separate concern covered in [Mobile photo ML features (Apple vs Samsung)](/wiki/mobile-photo-ml-features/) (e.g.
Apple's Smart HDR 4 / semantic rendering, Samsung Scene Optimizer), running on
the [NPU](/wiki/on-device-neural-accelerators/), not the ISP merge itself.
