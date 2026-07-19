---
share: true
title: Mobile photo ML features (Apple vs Samsung)
categories:
  - Wiki
tags:
  - on-device-ai
  - computer-vision
  - android
  - gallery
author: claude
type: concept
created: 2026-07-19
updated: 2026-07-19
sources:
  - "[[2026-07-19-on-device-ml-in-apple-and-samsung-camera-and-gallery]]"
aliases:
  - Scene Optimizer
  - Apple scene classification
  - People album
  - Galaxy AI
  - Live Text
  - semantic rendering
---

# Mobile photo ML features (Apple vs Samsung)

The shipped Camera and Gallery/Photos ML capabilities in iPhone and Galaxy, as a
**product/systems layer** on top of [the NPU](/wiki/on-device-neural-accelerators/),
[the runtime](/wiki/on-device-ml-runtimes/), and mobile-class models. The recurring
finding: **Apple documents its pipeline (named models, on-device latency/memory);
Samsung documents its features (capabilities and training-set sizes) but rarely
names an architecture and never publishes a per-feature NPU budget** — so Apple's
"runs on the NPU at X ms" claims are sourced, Samsung's are mostly inference.

## Feature-by-feature

### Scene/object classification
- **Apple** — Vision's `VNClassifyImageRequest` (hierarchical labels,
  precision/recall filterable); in-product it is one *head* on the shared
  scene-analysis backbone (ANSA, see [On-device semantic photo search](/wiki/on-device-semantic-photo-search/)),
  on the ANE. ⚠️ The "~1000 classes" figure is third-party, not Apple.
- **Samsung Scene Optimizer** — on-device **preview-stage CNN** on the NPU whose
  scene label is handed to the ISP for scene-specific tuning; scene count grew ~20
  (Note 9/S9, 2018) → 30 (S10, 2019) → "up to 30" (S20+). A separate *post-capture*
  "AI detail enhancement engine" runs on the multi-frame result (the moon-photo
  case). Constraint: classify at preview framerate (~tens of ms/frame), latency
  unpublished.

### Face detection & clustering
- **Apple People (& Pets)** — best-documented case: 2021 paper, **two-network**
  design (face-crop + upper-body-crop embeddings; backbone inspired by AirFace +
  MobileNetV3), **two-pass agglomerative clustering**, run **entirely locally**
  while charging. **Embedding generation <4 ms on ANE, 8× faster than GPU.**
  Naming a person propagates the name+face association across iCloud devices;
  ⚠️ whether *embeddings* or only *labels* transit iCloud is press-asserted, not
  Apple-stated. "People & Pets" rename in iOS 18 (2024).
- **Samsung Gallery** — on-device faceprint templates + clustering; the strongest
  *source* is a 2024 U.S. court BIPA dismissal describing templates as generated
  and stored **on the device**, not alleged transmitted. ⚠️ That is a pleadings
  finding, not a "never cloud" guarantee; architecture proprietary/unknown.

### OCR / text-in-images
- **Apple Live Text** — on-device via Vision's `VNRecognizeTextRequest`; shipped
  iOS 15 (2021), A12+ (leans on ANE), 7 launch languages; `DataScannerViewController`
  (live camera) in iOS 16 (2022). On-device from the start.
- **Samsung — two opposite models.** **Bixby Vision** text/translate (since S8,
  2017) is **network-dependent** (cloud OCR). The **Gallery "extract text"** (T
  icon, One UI 4.1.1, 2022) is a separate on-device path; ⚠️ engine (own vs Google
  ML Kit) undocumented.

### Semantic / natural-language search
See [On-device semantic photo search](/wiki/on-device-semantic-photo-search/) for the full mechanism. Apple: ANSA
CLIP-style embeddings + on-device knowledge graph (documented). Samsung: S25 (One
UI 7, 2025) "vision-language model", on-device per an independent breakdown;
⚠️ "CLIP-like" and the retrieval mechanism are inference.

### In-product IQA / IAA (best-shot, curation, dedup)
The **weakest-documented** area on both sides — neither names an IQA/IAA model in
its shipping stack. (The academic lineage lives in [Image Quality Assessment (IQA)](/wiki/image-quality-assessment/) /
[Image Aesthetic Assessment (IAA)](/wiki/image-aesthetic-assessment/) / [Multimodal-LLM Visual Scoring](/wiki/multimodal-llm-visual-scoring/); do **not**
import it as if it described these products.)
- **Apple** — documents only on-device "photo quality analysis" + ANSA embeddings
  driving Featured Photos, Memories curation, and the Duplicates album (iOS 16,
  2022). No published best-shot method.
- **Samsung Single Take** — the most concrete aesthetic detail anyone publishes:
  an "aesthetic engine" learning "about 300,000 expert-selected pictures"
  (sharpness/quality/expression) + a composition engine trained on "about 100
  million images" + person/motion/capture engines → best-shot crown pick.
  ⚠️ Architecture unnamed (no NIMA/VILA attribution), no NPU/latency figure.
  Gallery dedup (Clean Out) appears to catch **exact re-saves (hash)** — a
  perceptual near-duplicate embedding is *not* established. Auto blur-culling is
  not documented (Samsung documents blur *remediation* via Enhance-X/Remaster,
  not detection-for-deletion).

### Camera-time & edit-time ML
- **Apple segmentation** — best-documented camera-time ML: 2021 **HyperDETR**
  transformer producing masks for **six classes — sky, person, hair, skin, teeth,
  glasses** (⚠️ **not** foliage), on the ANE within the A15 (iPhone 13), ~17 MB
  after 8-bit quant. Drives Portrait mode, **Smart HDR 4**, **Photographic Styles**,
  and **semantic rendering** (up to four people). **Deep Fusion** (2019) / Smart
  HDR are neural multi-frame fusion but documented via press, not a paper. (The
  multi-frame *capture* mechanics — HDR+/Night/super-res — are in
  [Computational photography: multi-frame merge](/wiki/computational-photography-multiframe/); these masks are the ML guidance layer
  on top.)
- **Generative edit** — on-device diffusion is feasible but heavy (~4 networks
  ~1.275B params; cost ≈ UNet × ~20+ steps → seconds; Apple's Core ML Stable
  Diffusion measured ~7.9 s on iPhone 14 Pro Max at 512²/20 steps, Dec 2022). So:
  Apple keeps light edits (Clean Up) on-device, routes heavy Apple-Intelligence
  requests to PCC (see privacy below). **Samsung** Object Eraser simple-erase
  (~2021) is on-device, but **Generative Edit / Sketch to Image / Photo Assist /
  Portrait Studio are cloud** (network + Samsung account + visible **"AI-generated"
  watermark**) on **Google Cloud (Gemini + Imagen 2 on Vertex AI)** — ⚠️ *not*
  Google Tensor silicon; Galaxy runs Snapdragon/Exynos.

## The privacy / on-device boundary

- **Apple — on-device by default, three named cloud mechanisms.** Classification,
  faces, segmentation, embeddings all local. **Differential privacy** is used for
  *aggregate telemetry* only (⚠️ not to protect your own photos' content — a common
  conflation). **Enhanced Visual Search** (landmark, 2024) is the "leaves the
  device" exception: on-device embedding → **BFV homomorphic encryption** →
  server private nearest-neighbor search on the *ciphertext* (DP noise ε=0.8,
  δ=10⁻⁶; OHTTP relay), on by default (criticized Jan 2025). **Private Cloud
  Compute** handles heavy Apple-Intelligence generative requests on Apple-silicon
  servers. No per-feature on-device/PCC table is published.
- **Samsung — hybrid with a switch.** "Process data only on device" toggle (One UI
  6.1 menu, 2024) forces Samsung's own features local and disables cloud ones —
  but only affects Samsung's features, not third-party apps (Gemini, ChatGPT).
  Public stance: data "never stored long-term or used for AI training". The
  boundary is documented exactly where Samsung had a reason (generative edits =
  cloud+watermark; NL search = on-device); curation/IQA features are only inferred
  on-device.

## See also

- [On-device neural accelerators (NPU / ANE / Hexagon)](/wiki/on-device-neural-accelerators/) · [On-device ML runtimes (Core ML vs LiteRT)](/wiki/on-device-ml-runtimes/) · [On-device semantic photo search](/wiki/on-device-semantic-photo-search/) — the three layers under these features.
- [Computational photography: multi-frame merge](/wiki/computational-photography-multiframe/) · [Android Camera2 pipeline and CameraX interop](/wiki/android-camera2-pipeline/) — the capture pipeline the camera-time ML sits on.
- [Image Quality Assessment (IQA)](/wiki/image-quality-assessment/) · [Image Aesthetic Assessment (IAA)](/wiki/image-aesthetic-assessment/) — the method lineage behind (undocumented) best-shot scoring.
