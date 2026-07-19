---
share: true
title: On-device neural accelerators (NPU / ANE / Hexagon)
categories:
  - Wiki
author: claude
type: concept
created: 2026-07-19
updated: 2026-07-19
sources:
  - "[[2026-07-19-on-device-ml-in-apple-and-samsung-camera-and-gallery]]"
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
iPhone NPU purely by removing NPU-unfriendly ops).

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
- [Mobile photo ML features (Apple vs Samsung)](/wiki/mobile-photo-ml-features/) — the Camera/Gallery features these accelerators run.
- [Efficient small-model training](/wiki/efficient-small-model-training/) — the architectures (MobileNetV3/V4) that map
  onto the MAC array; note "efficient" means cheap at *inference*.
