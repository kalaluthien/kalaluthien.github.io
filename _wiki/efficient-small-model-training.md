---
share: true
title: Efficient small-model training
categories:
  - Wiki
tags:
  - machine-learning
  - training
  - transfer-learning
  - pytorch
author: claude
type: concept
created: 2026-07-17
updated: 2026-07-22
sources:
  - "[[2026-07-17-where-no-server-no-cloud-bill-stops-being-true]]"
  - "[[2026-07-22-tflite-object-detection-survey]]"
aliases:
  - cheap training techniques
  - CIFAR-10 speedrun
---

# Efficient small-model training

The practices that turn neural-net training from a cluster job into a laptop
job, each with a primary source. These are why small-CNN training fits on
[ML training on consumer hardware](/wiki/ml-training-on-consumer-hardware/) or inside [Free cloud compute tiers](/wiki/free-cloud-compute-tiers/)
quotas with orders of magnitude to spare — **at CIFAR scale**. At ImageNet
scale the same levers stratify sharply and several invert; see
[ImageNet-scale training logistics](/wiki/imagenet-scale-training-logistics/) for where the threshold sits and why
most of these stop mattering past it.

## The levers do not generalise across scale

Measured 2026-07-17 (see [Where "No Server, No Cloud Bill" Stops Being True](/posts/where-no-server-no-cloud-bill-stops-being-true/)),
the levers split into two kinds, and this is the load-bearing distinction:

- **Levers that shrink the problem** — transfer learning, input resolution,
  subset/downsampled datasets — **work at every scale**, because they change
  what is being trained.
- **Levers that speed up the run** — AMP, dataloading, LR schedules —
  **fail at ImageNet scale**, because no realistic multiplier on a laptop
  bridges the ~46× gap to a reference recipe.

| Lever | CIFAR scale | ImageNet scale on an M4 laptop |
|---|---|---|
| Transfer learning | 2×, nice-to-have | **load-bearing — it IS the answer. Measured ~20,600×** |
| Partial unfreeze vs full | rarely worth it | **94.91% vs 94.65% at 52% of wall-clock** |
| Input resolution | minor | **load-bearing — 7.6× measured** (64 px vs 224 px) |
| Efficient architecture | helpful | **insufficient + misleading** (cheap per step, trained 4–36× longer) |
| Mixed precision (AMP) | 2–3× on NVIDIA | **~6% on MPS — measured** |
| Dataloading (FFCV/DALI) | irrelevant | irrelevant on a laptop (18× headroom); binding on free-tier vCPUs |
| One-cycle / early stop | 33% | real but bounded |
| Distillation | "accurate, not cheap" | **increases cost** (MNv4: 2000 epochs) |
| Subset datasets | trivial | **load-bearing — the practical answer** |

## The levers, roughly by impact

1. **Transfer learning over from-scratch** — the biggest lever, by orders of
   magnitude. Measured: **69 days to pre-train vs 4.83 min to fine-tune**
   (~20,600×), and **+60.1 accuracy points over random init at identical
   cost** — see the ratio section below. Kornblith et al. (arXiv:1805.08974)
   found fine-tuning beat logistic regression in **179/192** and beat random
   init in **189/192** dataset×model combinations. Transferred features help
   almost regardless of task distance (Yosinski et al., NeurIPS 2014,
   arXiv:1411.1792).
2. **Efficient architectures built for small budgets** — start from an
   edge-designed backbone rather than scaling down a server-class one.
   MobileNetV3 (Howard et al., arXiv:1905.02244) is in torchvision with
   ImageNet weights — the natural cheap fine-tuning backbone (V3-Small:
   2.54M params for the 1000-class backbone, ~1.5M with a 10-class fine-tune
   head; measured 2026-07-17 on an M4 MacBook: fine-tuned to 92.5%
   CIFAR-10 in 4.9 min vs 88.5% in 8.5 min for a from-scratch ResNet9 —
   see [ML training on consumer hardware](/wiki/ml-training-on-consumer-hardware/)). MobileNetV4 (arXiv:2404.10518, ECCV 2024; universal
   inverted bottleneck + Mobile MQA, Pareto-optimal across mobile hardware)
   is available pretrained via timm. MobileMamba (He et al., CVPR 2025,
   arXiv:2411.15941) is the research-stage state-space alternative —
   linear-complexity global receptive fields at lightweight scale, no
   torchvision/timm integration yet.
   ⚠️ **Caveat discovered 2026-07-17: "efficient" means cheap at _inference_,
   not cheap to _train_.** Mobile-class backbones are trained far longer than
   server-class ones to extract their accuracy — torchvision's
   `mobilenet_v3_large` recipe is 600 epochs, timm's `ra4` checkpoint 3600,
   MobileNetV4's paper budget 500–9600. Picking a MobileNet over a ResNet-50
   does **not** make ImageNet training affordable; ResNet-50 reaches 78.4% in
   88 epochs (FFCV). The efficiency is real and it is on the other side of
   the deployment boundary — depthwise-separable convs map onto the NPU MAC
   array as first-class operators, which is *why* MobileNet-class nets (and the
   MobileCLIP encoder) dominate on-device vision; see
   [On-device neural accelerators (NPU / ANE / Hexagon)](/wiki/on-device-neural-accelerators/) and [On-device ML runtimes (Core ML vs LiteRT)](/wiki/on-device-ml-runtimes/) for how a
   trained model is quantized and dispatched there, and
   [Mobile photo ML features (Apple vs Samsung)](/wiki/mobile-photo-ml-features/) for what actually ships.
   on-device-object-detection is a concrete deployment-side instance of
   quantization as a lever — measured int8 TFLite detectors, not training-time
   compression.
3. **One-cycle LR / super-convergence** — order-of-magnitude fewer
   iterations (Smith & Topin, arXiv:1708.07120); measured ~33% cut on a tuned
   MNIST run (tuomaso/train_mnist_fast).
4. **Early stopping / fewer epochs** — stock PyTorch MNIST example, 14→1
   epochs: 2 m 52 s → 57 s at 99.19% → 99.07% (train_mnist_fast). Most
   accuracy arrives early.
5. **Mixed precision (AMP)** — 2–3× on Tensor-Core NVIDIA GPUs (official
   PyTorch AMP recipe). ⚠️ **On Apple MPS it is worth 4–8%, measured**
   (MobileNetV3-Large 224 px, M4: fp32 120.8 → bf16 128.2 → fp16 129.8
   img/s). Treat the 2–3× as NVIDIA-specific; on Apple Silicon AMP is a
   rounding error, not a lever.
6. **Data augmentation** — crop/flip/cutout buys accuracy that would
   otherwise cost model size or data (Cutout, arXiv:1708.04552).
7. **Knowledge distillation** — makes small models *accurate* more than it
   makes training *cheap* (presupposes a teacher): Hinton et al.,
   arXiv:1503.02531. **Stronger claim, confirmed 2026-07-17: at ImageNet
   scale it makes training strictly _more_ expensive.** MobileNetV4 §8
   distils an MNv4-Conv-L student for **2000 epochs** to reach 85.9% top-1;
   its 87.0% headline additionally requires pretraining on **JFT-300M**
   (private Google data) — that number is data-gated and permanently
   irreproducible at any budget, not merely expensive.
8. **Input resolution** — the most under-rated lever, and the dominant one on
   Apple Silicon: MobileNetV3-Large on an M4 measures **982 img/s at 64 px vs
   129 img/s at 224 px (7.6×)**. This is what puts Downsampled ImageNet 64×64
   on the feasible side of the laptop threshold.

## The CIFAR-10 speedrun lineage (existence proof of how cheap this is)

Record progression for 94% test accuracy on a single GPU:
DAWNBench-era "How to train your ResNet" (David Page) 26 V100-s →
tysam-code/hlb-CIFAR10 6.3 A100-s → **KellerJordan/cifar10-airbench 2.59
A100-s** (current as of 2026-07-17; paper: arXiv:2404.00498, "94% on
CIFAR-10 in 3.29 Seconds on a Single GPU"). Also 95% in 10.4 s, 96% in
27.3 s.

Their bag of tricks, worth stealing when any run feels slow: patch/input
whitening, identity/dirac init, GPU-resident dataloading, derandomized
flip+translate+cutout, label smoothing, test-time augmentation,
channels-last memory format, tuned one-cycle-style schedules, Muon
optimizer (the 2025 record).

## Pretrained weights are the biggest lever of all

The strongest form of transfer learning: **don't train the expensive part at
all — someone else already paid.** torchvision's
`MobileNet_V3_Large_Weights.IMAGENET1K_V2` is **75.274% top-1** and downloads
in seconds; reproducing it on an M4 laptop is **69 days** of continuous
training. timm's `mobilenetv3_large_100.ra4_e3600_r224_in1k` reaches
**76.31%** — beating the MobileNetV3 paper's own 75.2% claim by spending ~6×
the epoch budget. The `e3600` in the checkpoint name is the receipt.

### The ratio, measured (M4, MobileNetV3-Large, 2026-07-17)

Pre-train ImageNet-1k (torchvision 600-ep recipe) = **69 days**. Fine-tune the
released weights on Flowers-102 (2,040 train imgs, 20 ep) = **4.83 min →
94.65%** test acc. **Ratio ≈ 20,600×** (≈39,400× for partial unfreeze at
2.52 min → 94.91%).

**The budget-matched control is the argument**: same architecture, same 20
epochs, same wall-clock — only the init differs.

| Init | Wall | Flowers-102 test |
|---|---|---|
| ImageNet-pretrained | 4.83 min | **94.65%** |
| Random | 4.75 min | **34.53%** |

**+60.1 points for 5 seconds.** ⚠️ 34.53% is a starved 20-epoch number, not
from-scratch's converged accuracy — Kornblith et al. (arXiv:1805.08974 §4.6)
measure the fair version: fine-tuning hits the bar in **~26 epochs vs ~444
from scratch, a 17× step gap**.

### How much to unfreeze

Measured on Flowers-102 (2,040 imgs): full 94.65% / 4.83 min · **partial (last
stage + head) 94.91% / 2.52 min** · head-only 90.80% / 1.39 min · scratch
34.53%. **Partial unfreeze is the sweet spot** — matched full accuracy at half
the time.

Rule, from Kornblith §4.7 ("when data is sparse (47-800 total examples),
logistic regression is a strong baseline, achieving accuracy comparable to or
better than fine-tuning"): **below ~1k examples freeze the trunk; above it
unfreeze at least the last stage.** The PyTorch transfer-learning tutorial
demonstrates the sparse side (~240 imgs: frozen **95.42%** *beats* full
fine-tune **93.46%**, and is faster).

**Epoch budgets for fine-tuning** — two independent primary answers:
Kornblith's **~26 epochs**, and BiT-HyperRule
(`google-research/big_transfer/bit_hyperrule.py`), which prescribes purely by
dataset size: **<20k examples → 500 steps**; <500k → 10k; ≥500k → 20k. Compare
pre-training's 600–3600 epochs.

**Frontier ratio**: NFNet-F7+ needed "roughly **110k TPU-v4 core hours to
pre-train and 1.6k TPU-v4 core hours to fine-tune**" (arXiv:2310.16764) —
**1.45%, ~1/69th**.

⚠️ **Not evidence for this**: Green AI (arXiv:1907.10597) doesn't quantify
pretrain-vs-finetune; Strubell et al. (arXiv:1906.02243) reports no fine-tuning
cost — its "tuning" is hyperparameter search, easily misread as adaptation.

### Fine-tuning is cheap but bounded

At the measured 137 img/s it busts a 36 h laptop budget past **~890k training
images at 20 epochs**. Food-101 (75,750) = 3.1 h; ImageNet-1k *as a fine-tune
target* = 2.2 days ✗. **"Fine-tuning is cheap" is a claim about downstream
datasets being small, not a law** — see [ImageNet-scale training logistics](/wiki/imagenet-scale-training-logistics/).

⚠️ **Imagenette is not a valid transfer target.** Its class dirs *are* ImageNet
WNIDs (`n01440764`…) — pretrained weights have already trained on those exact
images and labels, so fine-tuning on it measures memorisation. fastai's README:
"Ensure that you start from random weights - not from pretrained weights." Use
**Flowers-102** instead: Kornblith Table H.1 measures its ImageNet-train
duplicate overlap at **0.05% (1 image)**, and it is the one fine-grained set
they exclude from "much better-represented in ImageNet".

Reproduction is itself a research project: torchvision's *original* V3-Large
weights land at 74.042% (1.16 short of the paper); MMClassification's
independent attempt got only 73.39%. ⚠️ No one publishes what these weights
cost to produce — "600 epochs × 8 GPUs, cost undisclosed" is the honest claim;
there is no citable dollar figure.

## Scale anchor

MNIST to 99% ≈ one tuned epoch ≈ 4.3 TFLOPs (762 ms on a 2019 laptop GPU);
CIFAR-10 to 94% ≤ ~0.8 PFLOPs total (speedrun upper bound). Both are seconds
of a free-tier T4's throughput.

The far end of the same axis, for calibration: OpenCLIP ViT-H/14 on LAION-2B
is **229,665 A100-hours** — **22,319×** an FFCV ResNet-50 (10.3 A100-h to
78.4%), and 26 years on a single A100. The distance from "seconds of a T4" to
"decades of an A100" is the whole spectrum; see
[ImageNet-scale training logistics](/wiki/imagenet-scale-training-logistics/).
