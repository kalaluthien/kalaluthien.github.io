---
share: true
title: ImageNet-scale training logistics
categories:
  - Wiki
tags:
  - imagenet
  - machine-learning
  - training
  - datasets
author: claude
type: concept
created: 2026-07-17
updated: 2026-07-17
sources:
  - "[[2026-07-17-where-no-server-no-cloud-bill-stops-being-true]]"
aliases:
  - ImageNet logistics
  - LAION logistics
  - data wall
---

# ImageNet-scale training logistics

What changes when a training job moves from CIFAR-10 (170 MB, minutes) to
ImageNet-1k (144 GB, months) or LAION (50 TB, never). The short version:
**compute stops being the only constraint, and data logistics often bite
first**. Complement of [ML training on consumer hardware](/wiki/ml-training-on-consumer-hardware/) (which locates the
laptop threshold) and [Efficient small-model training](/wiki/efficient-small-model-training/) (which levers still
work past it).

## The threshold

A laptop tops out at **~24–36 hours of continuous GPU training**. Measured
2026-07-17 on an M4 MacBook, MobileNetV3-Large: full ImageNet-1k at
torchvision's own recipe (600 epochs, 224 px) is **69 days — ~46× over
budget**. No lever on that hardware closes 46×; the largest available (AMP on
MPS) is worth 6%.

## Reference epoch budgets are the shock

Mobile-class models are trained *far* longer than intuition suggests. Primary
sources:

- torchvision `references/classification` README: `mobilenet_v3_large`,
  **`--epochs 600`**, 8× V100, global batch 1024.
- timm checkpoint names encode it and are unfakeable:
  `mobilenetv3_large_100.ra4_e3600_r224_in1k` = **3600 epochs** → 76.31%;
  `mobilenetv4_conv_small.e2400_r224_in1k` = **2400 epochs**.
- MobileNetV4 paper (arXiv:2404.10518) Table 9: Conv-M/L **500**, Conv-S
  **9600**.
- ResNet Strikes Back (arXiv:2110.00476) is the one paper reporting cost
  honestly: A1 600 ep = 110 h on 4× V100 (**440 GPU-h**); A3 100 ep = 15 h
  (**60 GPU-h**).

⚠️ **Neither MobileNet paper states its training cost.** V3 (arXiv:1905.02244)
gives hardware ("4x4 TPU Pod" = 32 chips, batch 4096) but **no epoch count**;
V4 gives epochs but **no training hardware**. Any "MobileNetV3 took N
TPU-hours" claim is unsourceable.

## The data wall

**ImageNet-1k**: train tar **138 GB** + val **6.3 GB**; 1,281,167 / 50,000 /
100,000 images, 1000 classes. Size is *not* the wall — 144 GB is a weekend on
home broadband. **The gate is**: image-net.org requires login and agreement to
terms restricting use to "non-commercial research and educational purposes",
binding your employer and indemnifying Princeton/Stanford. The HF mirror
(`ILSVRC/imagenet-1k`) is gated behind the same terms.
**Consequence: you may not redistribute your preprocessed copy**, so every
fresh environment re-downloads 144 GB — which is what makes ephemeral
free-tier storage lethal rather than annoying ([Free cloud compute tiers](/wiki/free-cloud-compute-tiers/)).

**LAION** ships **URLs, not images** — "indexes to the internet… lists of URLs
together with the ALT texts". So "downloading LAION" means running
`img2dataset` yourself. Its own README: 100M urls in **20 h on one machine**
(~1350 img/s); the full 5.8B run took **7 days on 10 machines → 240 TB**, or
**50 TB** if resized to 256 px. Extrapolated to one machine: **~46 days of
continuous downloading** for 50 TB you have nowhere to put.

And it decays underneath you: img2dataset's MD5 check drops **~15%** (images
that *changed* — a floor on drift, not rot); broader link rot is measured at
**>20%** across 24 datasets / >270M URLs (Rossetto et al., MMM 2023 — medium
confidence, PDF host unreachable).

**LAION status**: original LAION-5B taken down **19 Dec 2023** after the
Stanford Internet Observatory CSAM report; re-released **30 Aug 2024** as
**Re-LAION-5B** (5,526,641,167 pairs, 2236 links removed, Apache 2.0 but
"not suitable for industrial applications"). Verified 2026-07-17: the original
has **not** returned; Re-LAION-5B is the only version, and the HF parquet
(467 GB) is itself gated.

## JPEG decode: the bottleneck that depends on your machine

The literature is emphatic that decode, not IO, bottlenecks ImageNet training.
FFCV (arXiv:2306.12517) has the cleanest proof: reading all of ImageNet from
memory = **75 s**; adding decode + crop + resize + flip + normalize = **1200 s**
(**~16×**). NVIDIA DALI exists to offload it to the GPU. webdataset's
sequential-IO argument addresses the *secondary* problem.

**But it does not generalise to a laptop.** Measured on an M4 (10 cores,
real DataLoader path, 132 KB ImageNet-realistic JPEGs): **2519 img/s** at 10
workers vs the GPU's 129–138 img/s — **18× headroom**. The CPU is wildly
overprovisioned relative to the GPU; FFCV/DALI buy nothing there.

The rule: **decode binds where the GPU is strong and the CPU is weak** — i.e.
free-tier notebooks (T4/P100 + ~2–4 vCPUs) and multi-GPU rigs (FFCV needed
96 vCPUs for 8 A100s). On a laptop, compute binds.

## What is actually reachable

| Route | Size | Gated? | Cost |
|---|---|---|---|
| **Fine-tune released weights** | ~20 MB | no | **4.8 min → 94.65% on Flowers-102 (measured)** |
| Imagenette / Imagewoof | 94 MB–1.45 GB | no | 4 h laptop → 95.11% / 90.38% |
| Downsampled ImageNet 64×64 (TFDS) | **11.7 GB** | **no** | 14–36 h laptop, full 1000-class difficulty |
| ImageNet-100 | ~13 GB | yes (wnid list only, needs full ImageNet) | ~28 h laptop |
| Full ImageNet, FFCV ResNet-18 | 144 GB | **yes** | **3.1 h on one A100** → 72.4% |
| LAION-2B, OpenCLIP ViT-H/14 | 50 TB | yes | **229,665 A100-h ≈ 26 years on one A100** |

**Fine-tuning is the answer on the wrong side of the threshold — and it is
bounded.** Measured 2026-07-17 (M4, MobileNetV3-Large): pre-train = 69 days;
fine-tune the released weights on Flowers-102 = **4.83 min → 94.65%**, a
**~20,600× ratio**; random-init control at the same budget = 34.53%
(**+60.1 points for free**). But at 137 img/s it busts a 36 h laptop budget past
**~890k training images at 20 epochs** — Food-101 (75,750) = 3.1 h ✓, ImageNet-1k
*as a fine-tune target* = 2.2 days ✗, Places365 = 3.0 days ✗. **The claim is
"downstream datasets are small", not "fine-tuning is free".** Epoch budgets:
Kornblith's ~26 epochs, or BiT-HyperRule's 500 steps for <20k examples — against
pre-training's 600–3600 epochs. See [Efficient small-model training](/wiki/efficient-small-model-training/).

**Two findings worth internalising:**

1. **Full ImageNet is compute-reachable on a single owned GPU** — FFCV's
   `rn18_88_epochs.yaml` is 72.4% top-1 in 3.1 A100-hours (~$6 rented, or an
   overnight 4090 run). **The 144 GB gated download and fast local disk are
   what stop you, not GPU-hours.** That inverts the intuition.
2. **Downsampled ImageNet 64×64 is the sweet spot for a laptop**: 11.7 GB,
   *ungated via TFDS*, full 1000-class difficulty. Chrabaszcz, Loshchilov &
   Hutter (arXiv:1707.08819) report WRN-28-10 @32×32 reaching 40.96%/18.87%
   error — matching AlexNet on full-size ImageNet, which has "roughly 50 times
   more pixels per image" — in **13.8 days on one Titan X**; WRN-36-2 @64×64 in
   6.4. Their opening line is the whole concept: "training a strong ImageNet
   model typically requires **several GPU months**."

Imagenette is the only benchmark purpose-built for this constraint — its README
sets explicit **$0.05–$2.00** budget targets and insists "**start from random
weights - not from pretrained weights**".

## LAION scale: quantified impossibility

Cherti et al. (arXiv:2212.07143) Table 18, JUWELS Booster A100-40GB:
ViT-B/32 @LAION-2B **42,307 GPU-h** (4.8 yr on one A100); ViT-L/14
**122,509**; ViT-H/14 **229,665** (26.2 yr, 80 MWh). Whole study:
**1,058,318 GPU-h / 370 MWh** ≈ 35 US households' annual electricity.
OpenCLIP's PRETRAINED.md corroborates independently ("ViT-B/32 was trained with
128 A100 GPUs for ~36 hours, 4600 GPU-hours").

**ViT-H/14 = 22,319× an FFCV ResNet-50.** This is not a budget problem; it is a
category error. Use the released weights — they exist so nobody repeats this.

⚠️ **Anomaly in Table 18**: B/32 @LAION-80M / 34B samples = 32,953 GPU-h
(344 GPUs × 96 h) vs B/32 @LAION-400M / **same 34B samples, same architecture**
= 11,177 GPU-h (128 × 87). Identical FLOPs, **3× gap**. Columns are internally
consistent, so it is either real scaling inefficiency at 344 GPUs or a
published error. **GPU-hours at this scale are configuration-dependent to ~3×**
— every derived dollar figure inherits that.

## Unverified — do not assert

- Kaggle's ToS on session chaining (kaggle.com/terms is client-rendered,
  unreadable by fetch).
- Colab's disk quota (Google declines to publish; "~100 GB" is folklore).
- What torchvision/Google spent to produce the released weights (never
  published).
- MosaicML's circulated "$15 / 76.6%" ResNet figures (not on the blog).
- Flowers-102's download size (no `Content-Length` served; ~330 MB unconfirmed).
- Any MobileNetV3-on-Flowers-102/Food-101 figure (no primary source; Kornblith
  predates V3 — v1/v2 is the proxy).
- Whether JPEG decode binds at head-only fine-tuning on a free-tier notebook
  (direction argued from measured headroom erosion; no notebook benchmarked).
- CUB-200-2011 split sizes; Oxford-IIIT Pets test size (Kornblith Tables 1 and
  H.1 disagree: 3,369 vs 3,669).
