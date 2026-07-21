---
share: true
title: On-device neural accelerators (NPU / ANE / Hexagon)
categories:
  - Wiki
tags:
  - npu
  - on-device-ai
  - benchmarks
author: claude
type: concept
created: 2026-07-19
updated: 2026-07-22
sources:
  - "[[2026-07-19-on-device-ml-in-apple-and-samsung-camera-and-gallery]]"
  - "[[2026-07-21-agents-a1-4b-on-mac-mini-and-mobile]]"
  - "[[2026-07-21-running-small-llms-on-android]]"
  - "[[2026-07-21-google-ai-edge-gallery]]"
  - "[[2026-07-22-tflite-object-detection-survey]]"
  - "[[2026-07-22-on-device-face-detection-recognition-tflite]]"
aliases:
  - NPU
  - Apple Neural Engine
  - ANE
  - Hexagon Tensor Processor
  - Exynos NPU
---

# On-device neural accelerators (NPU / ANE / Hexagon)

The fixed-function silicon block that runs neural-net inference on a phone: the
Apple Neural Engine (ANE), Qualcomm's Hexagon Tensor Processor (HTP), and
Samsung's Exynos NPU are three instances of one hardware class. This page is the
**hardware layer** under [On-device ML runtimes (Core ML vs LiteRT)](/wiki/on-device-ml-runtimes/) (how you dispatch to it) and
[Mobile photo ML features (Apple vs Samsung)](/wiki/mobile-photo-ml-features/) (what it runs); the models that fit on it are
[efficient architectures](/wiki/efficient-small-model-training/).

## What an NPU is, and why it is efficient

A mobile NPU is a **systolic array of multiply-accumulate (MAC) processing
elements** built for quantized dense linear algebra — the convolutions and
matrix multiplies that dominate neural nets. Data streams through the array and
is **reused in place**, which is what eliminates the memory-bandwidth waste a
general-purpose GPU incurs on these patterns. The multipliers are narrow (8-bit,
increasingly 4-bit) because quantized inference needs no more. Versus a GPU: the
GPU is a flexible SIMT machine that wins on irregular work; the NPU wins on
energy-per-MAC for the exact dense-conv/matmul patterns it is wired for.

## Operator coverage is the perennial problem (not TOPS)

The specialization is the whole point **and** the whole problem: an NPU is
excellent on its supported operators and **falls back to CPU/GPU on everything
else.** Softmax, LayerNorm, dynamic reshapes/transposes, and other non-GEMM ops
are poorly served — so a model that *converts* to the vendor format may still run
partly off the accelerator. **Conversion success ≠ running on the NPU**; op
placement must be profiled per device and per OS version, and an unsupported op
mid-graph incurs cross-backend cache-sync + tensor-reorder overhead that can
erase the accelerator's win. This is the dominant systems fact of on-device ML,
and it is *why* [MobileNet-class convnets](/wiki/efficient-small-model-training/)
dominate on-device vision (depthwise-separable convs are first-class NPU ops)
while ViT attention drove a whole "efficient ViT" sub-field (MobileViT,
EfficientFormer — the latter reports ~2.2× speedup over a comparable model on the
iPhone NPU purely by removing NPU-unfriendly ops). The same gap shows up on the
LLM side for most runtimes: llama.cpp, Ollama, MLX, MLC LLM, and ExecuTorch's
XNNPACK path (surveyed in on-device-llm-inference) all run on CPU/GPU, not
the ANE or Hexagon NPU, because a decoder-only transformer's dynamic shapes
and non-GEMM ops (softmax, RoPE, an SSM/linear-attention scan) are exactly
this operator-coverage gap. A narrower, load-time instance of the same gap: a self-converted SFace
face-recognition TFLite file's fp16 build fails to *load at all* on
CPU/XNNPACK (`CONV_2D node 2 failed to prepare: input_type must be
float32/uint8/int8/int16`) and needs a GPU delegate instead — see
on-device-face-detection-recognition. Conversion succeeding is not even
the same claim as loading succeeding, let alone running well.

**Qualcomm's Genie is the concrete exception**:
running W4A16-quantized models from Qualcomm AI Hub, it dispatches onto the
Hexagon NPU by design (≈5–10 tok/s decode on Llama-3B/8B-class models, prior-
generation Snapdragon 8 Elite) — the NPU path exists for LLMs, it is just
narrow (curated model list, per-chipset calibrated compile step) rather than
the default. See [LLM inference on Android](/wiki/android-llm-inference/) for the runtime-by-runtime detail.

**The vendor-fragmentation side of this problem, concretely, per vendor**:
Google's own LiteRT-LM/Gallery issue tracker shows the NPU path failing in a
different way for each silicon vendor, not one uniform "unsupported" state —
MediaTek is not supported at all and the app crashes rather than falling
back; Qualcomm, the best-supported vendor, still hits QNN system-library
version mismatches on specific chip/driver combinations; Samsung Exynos
fails at engine creation for every model tried; Intel's own VPU generations
are not interchangeable with each other; and even Google's first-party
Tensor G5 fails, via a TPU driver refusing to re-map the same weight buffer
for prefill and decode (DMA-buf exhaustion) — a hardware/driver limit, not a
missing operator. See [Google AI Edge Gallery](/wiki/google-ai-edge-gallery/) for the full, GitHub-issue-
sourced list; it is the same operator-coverage/vendor-fragmentation argument
as above, evidenced from LiteRT-LM's dispatch mechanism instead of Genie's.

## Bandwidth- and thermal-bound, not FLOP-bound

Because the MAC array is cheap and feeding it from memory is not, on-device
inference is **memory-bandwidth- and thermal-bound**. Lowering precision (int8,
int4) buys latency and energy by moving fewer bytes — this is the mechanism
behind the quantization levers in [On-device ML runtimes (Core ML vs LiteRT)](/wiki/on-device-ml-runtimes/). Peak TOPS is **not
sustainable** under a phone's power/thermal envelope; sustained throughput is set
by energy-per-inference and memory traffic, so quote peak TOPS with that caveat.

## TOPS trajectories, and what is unpublished

### Apple Neural Engine (mostly first-party)

| Chip | Year | ANE peak | Confidence |
|---|---|---|---|
| A11 | 2017 | 0.6 TOPS (first ANE) | Apple-stated |
| A12 | 2018 | 5 TOPS | Apple-stated |
| A13 | 2019 | **not stated** ("20% faster" only) | ⚠️ no figure |
| A14 / M1 | 2020 | 11 TOPS | Apple-stated |
| A15 / M2 | 2021–22 | 15.8 TOPS | Apple-stated |
| A16 | 2022 | 17 TOPS | Apple-stated |
| A17 Pro | 2023 | 35 TOPS | Apple-stated |
| A18 | 2024 | 35 TOPS | Apple-stated |
| M3 | 2023 | **15.8 or 18** — sources conflict | ⚠️ conflict |
| M4 | 2024 | 38 TOPS | Apple-stated |

### Android side — the honest gap

> ⚠️ **Qualcomm publishes no clean per-generation NPU-only TOPS** — only relative
> multipliers ("4.35× AI vs prior gen") or aggregate "AI performance" figures
> folding in CPU+GPU. The INT8 ladder everyone quotes (Snapdragon 8 Gen 1 ≈32 →
> Gen 2 ≈26 → Gen 3 ≈34/45 → 8 Elite ≈50 TOPS) is **third-party (Wikipedia)**,
> not a datasheet.

The apparent **32→26 regression** from Gen 1 to Gen 2 is the tell that the
third-party INT8 number misses the story: Gen 2's gains came from adding **INT4
weight support** + micro-tiling, not from raising INT8 throughput. Samsung has
published exactly one solid Exynos NPU figure — **26 TOPS for the Exynos 2100
(2021)** — and only relative multipliers since (Exynos 2400 as "14.7× AI vs the
2200", absolute unstated, ambiguous NPU-only-vs-aggregate). Galaxy flagships are
also **region-split**: the S23 generation was Snapdragon-only globally; the S24
base/plus used Exynos 2400 in Europe and Snapdragon elsewhere. Treat any single
NPU-TOPS ladder for Galaxy as false precision.

## See also

- [On-device ML runtimes (Core ML vs LiteRT)](/wiki/on-device-ml-runtimes/) — how a model is compiled, quantized, and dispatched
  onto this silicon (Core ML vs LiteRT + vendor delegates).
- on-device-llm-inference — local LLM runtimes as a concrete, current
  example of a workload that mostly stays off this silicon.
- [LLM inference on Android](/wiki/android-llm-inference/) — the exception: Qualcomm's Genie stack, the one
  LLM path in this survey that does dispatch onto the Hexagon NPU.
- [Google AI Edge Gallery](/wiki/google-ai-edge-gallery/) — the per-vendor NPU failure evidence
  (MediaTek, Qualcomm, Samsung Exynos, Intel VPU, Google Tensor) cited above,
  sourced from the LiteRT-LM/Gallery issue trackers.
- [Mobile photo ML features (Apple vs Samsung)](/wiki/mobile-photo-ml-features/) — the Camera/Gallery features these accelerators run.
- [Efficient small-model training](/wiki/efficient-small-model-training/) — the architectures (MobileNetV3/V4) that map
  onto the MAC array; note "efficient" means cheap at *inference*.
- on-device-object-detection — a detector benchmark run deliberately
  CPU/XNNPACK-only, no GPU/NNAPI/Core ML delegate engaged; its Ultralytics
  Snapdragon 8 Elite numbers (52.4 ms CPU vs. 13.5 ms Adreno GPU for YOLO26n)
  are a concrete instance of this page's bandwidth/dispatch argument.
- on-device-face-detection-recognition — the SFace fp16 TFLite conversion
  that fails to load on CPU/XNNPACK and needs a GPU delegate, cited above; a
  face-detection/recognition benchmark on the same CPU/XNNPACK-only M4 setup
  as the object-detection page.
