---
share: true
title: On-device ML runtimes (Core ML vs LiteRT)
categories:
  - Wiki
tags:
  - on-device-ai
  - core-ml
  - npu
  - android
author: claude
type: concept
created: 2026-07-19
updated: 2026-07-22
sources:
  - "[[2026-07-19-on-device-ml-in-apple-and-samsung-camera-and-gallery]]"
  - "[[2026-07-21-agents-a1-4b-on-mac-mini-and-mobile]]"
  - "[[2026-07-21-running-small-llms-on-android]]"
  - "[[2026-07-22-tflite-object-detection-survey]]"
aliases:
  - Core ML
  - LiteRT
  - TensorFlow Lite
  - NNAPI
  - QNN
  - ENN
---

# On-device ML runtimes (Core ML vs LiteRT)

The software layer that compiles, quantizes, and dispatches a model onto an
[NPU/ANE](/wiki/on-device-neural-accelerators/). Two worlds: Apple's single Core ML
path, and Android's multi-vendor LiteRT-over-delegates path. This page is the
**runtime layer** feeding [Mobile photo ML features (Apple vs Samsung)](/wiki/mobile-photo-ml-features/).

## Apple — one path, automatic placement

The stack is **Core ML** (runtime + the `.mlpackage`/ML Program format, compiled
to `mlmodelc`), with **Vision** (image-analysis requests: classification, text
recognition), **VisionKit** (Live Text / data-scanner UI), and **Create ML**
(training) on top. Placement across ANE / GPU / CPU is **automatic and
per-layer**: the developer ships one model and expresses a *preference* via
`MLComputeUnits` (`all`, `cpuOnly`, `cpuAndGPU`, `cpuAndNeuralEngine`); the
runtime decides the actual dispatch layer by layer, and that policy is
**deliberately opaque** — you cannot pin a layer to the ANE, only state a
preference.

**Model compression** is `coremltools.optimize`, three composable families Apple
documents as ANE-friendly on recent silicon (A17 Pro / M4):

- **Quantization** — int8 weights+activations; **int4 weight-only** (there is *no*
  int4-activation path).
- **Palettization** — lookup-table weights at 1/2/3/4/6/8 bits (Apple's name for
  weight-clustering / k-means codebook; compresses storage even when compute
  stays float).
- **Magnitude pruning** — sparsity; composes with the above (sparse-then-palettized).

Real budgets these hit, from Apple's own shipping models: ~17 MB (panoptic
segmentation) and ~24.6 MB (scene analysis) after 8-bit quantization. int8
weight-only / 6–8-bit palettization is near-lossless and "applied in minutes";
int4 and int8-activations are the aggressive levers needing care.

## Android — NNAPI is dead, LiteRT + vendor delegates replace it

The load-bearing 2024–2026 platform change:

- **NNAPI deprecated.** Android's Neural Networks API (introduced Android 8.1,
  2017) was **deprecated in Android 15 (2024, NDK API level 35)** — the rationale
  being that post-NNAPI on-device ML (transformers, diffusion) moved too fast for
  an OS-versioned API.
- **LiteRT.** TensorFlow Lite was **renamed LiteRT in September 2024** (same
  `.tflite` format and runtime, now under Google AI Edge, spanning
  PyTorch/JAX/Keras exports).
- **Delegates.** LiteRT's current delegate list is down to **GPU** (Android/iOS)
  and **Core ML** (iOS); the **NNAPI and Hexagon delegates are deprecated/removed.**
  GPU is the portable fallback.
- **LiteRT Next unified NPU API.** `CompiledModel.create(..., Accelerator.NPU)`
  dispatches to a vendor accelerator without the app touching vendor compilers:
  backends include **Qualcomm AI Engine Direct**, MediaTek NeuroPilot, **Samsung
  Exynos AI LiteCore**, Google Tensor, Intel OpenVINO.
- **Vendor-direct** (max control): Qualcomm **QAIRT / AI Engine Direct (QNN)**
  targets the Hexagon Tensor Processor; the older **SNPE** is superseded. A LiteRT
  Qualcomm AI Engine Direct accelerator (**November 2025**) claims "up to 100× CPU,
  10× GPU" and replaces the older QNN delegate. Samsung's **Neural SDK** last
  shipped v3.0 (2021) and is "no longer provided to third-party developers"; its
  **ENNDelegate** for the Exynos NPU is license-gated (non-open-source).

**Optimization** mirrors Apple's levers via LiteRT's four PTQ schemes —
dynamic-range (weights→int8, "4× smaller, 2–3×"), full-integer (int8
weights+activations, needs a representative dataset, "4× smaller, 3×+"), float16
(GPU), experimental 16×8 — plus QAT as the accuracy fallback. **int4** is a
*Qualcomm-hardware* feature (Hexagon HTP supports INT4 weights via QAIRT for
Conv2D/MatMul with >32 output channels), not a standard LiteRT PTQ mode.

## The portability tax

The same PyTorch/ONNX model must be exported and **re-verified on every vendor
stack**, because op-support gaps differ by vendor and by OS version — "runs on my
NPU" is a per-device claim, not a portable one (see the operator-coverage problem
in [On-device neural accelerators (NPU / ANE / Hexagon)](/wiki/on-device-neural-accelerators/)). Apple pays this tax once (Core ML only);
Android developers pay it per silicon vendor, which is exactly why Google built
the LiteRT Next abstraction to hide it. This runtime plumbing sits above the
[camera capture](/wiki/android-camera2-pipeline/) and [image
decode](/wiki/android-image-decoding/) layers; the ML runtime is a separate concern from getting pixels in.

## See also

- on-device-llm-inference — the third-party-runtime analog for
  general-purpose LLMs (llama.cpp, Ollama, MLX, MLC LLM, ExecuTorch) rather
  than the first-party Core ML / LiteRT stack this page covers.
- [LLM inference on Android](/wiki/android-llm-inference/) — the LLM-specific layer on this page's
  Android/vendor-delegate picture: LiteRT-LM, Qualcomm Genie/GenieX (the
  concrete Hexagon-NPU-dispatch case for QNN/QAIRT), ONNX Runtime GenAI, and
  a device-named bandwidth roofline.
- [On-device neural accelerators (NPU / ANE / Hexagon)](/wiki/on-device-neural-accelerators/) — the silicon these runtimes target.
- [Mobile photo ML features (Apple vs Samsung)](/wiki/mobile-photo-ml-features/) — the Camera/Gallery features built on these runtimes.
- [On-device semantic photo search](/wiki/on-device-semantic-photo-search/) — a concrete runtime workload (CLIP encoder + ANN).
- [Generative UI on Android](/wiki/generative-ui-android/) — the on-device *generative* runtime sibling: Gemini
  Nano via AICore and the ML Kit GenAI APIs, above this same LiteRT/vendor layer.
- [Efficient small-model training](/wiki/efficient-small-model-training/) — quantization/pruning also appear there as
  training-side levers; here they are the deployment-side compression story.
- on-device-object-detection — a concrete workload measured on exactly this
  layer's CPU/XNNPACK path (LiteRT's four PTQ schemes and the GPU-delegate
  latency gap both show up directly in that page's benchmark).
