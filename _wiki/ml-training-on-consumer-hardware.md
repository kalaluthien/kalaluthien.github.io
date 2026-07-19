---
share: true
title: ML training on consumer hardware
categories:
  - Wiki
tags:
  - machine-learning
  - training
  - macos
  - gpu
author: claude
type: concept
created: 2026-07-17
updated: 2026-07-18
sources:
  - "[[2026-07-17-where-no-server-no-cloud-bill-stops-being-true]]"
aliases:
  - local ML training
  - training without a server
---

# ML training on consumer hardware

What an individual's own machine can train, without any server or cloud. Small
CNNs (MNIST/CIFAR-10 scale, ≤10M params) are 4–6 orders of magnitude below
what consumer hardware provides — the binding constraint is never compute at
this scale. See [Efficient small-model training](/wiki/efficient-small-model-training/) for the practices that keep
runs cheap, and [Free cloud compute tiers](/wiki/free-cloud-compute-tiers/) for the zero-cost cloud
alternative.

**But this only holds below a threshold**, located by measurement on
2026-07-17: a laptop tops out around **24–36 hours of continuous GPU
training**. Full ImageNet-1k at reference resolution and reference epochs
overshoots that by ~46× and is not reachable on consumer hardware at all.
Details and the far side of the threshold: [ImageNet-scale training logistics](/wiki/imagenet-scale-training-logistics/).

## The threshold, measured (M4 MacBook, MobileNetV3-Large held constant)

| Rung | Res | Epochs | Wall-clock | Verdict |
|---|---|---|---|---|
| CIFAR-10, ResNet9 from scratch | 32 | 10 | 8.5 min | coffee |
| Imagenette (1.45 GB) | 224 | 200 | 4.1 h | an evening |
| Downsampled ImageNet 64×64 (11.7 GB) | 64 | 40 | 14.5 h | overnight |
| ImageNet-100 (~13 GB) | 224 | 100 | 27.6 h | just barely |
| ImageNet-1k, torchvision recipe | 224 | 600 | **69 days** | **no** |
| ImageNet-1k, timm `ra4` recipe | 224 | 3600 | **1.13 years** | **no** |

The transition is *narrow*, not gradual: same model, same laptop, same
resolution — Imagenette is an evening, full ImageNet at its own recipe's epoch
count is 69 days. Nothing on the far side is recovered by tuning.

**Fine-tuning sits far to the left of the same threshold** (measured, same M4,
same MobileNetV3-Large, Flowers-102 2,040 imgs / 20 ep):

| Arm | Wall | Test acc | Max dataset in 36 h @20 ep |
|---|---|---|---|
| Full fine-tune | **4.83 min** | **94.65%** | ~888,000 images |
| Partial (last stage + head) | **2.52 min** | **94.91%** | ~1.70 M images |
| Head-only | 1.39 min | 90.80% | ~3.09 M images |
| Scratch (control) | 4.75 min | **34.53%** | — |

**~20,600× cheaper than pre-training**, and +60.1 points over random init at
the same cost. But bounded: fine-tuning busts the 36 h budget past ~890k
training images — Food-101 (75,750) is 3.1 h ✓, ImageNet-1k as a fine-tune
target is 2.2 days ✗. See [Efficient small-model training](/wiki/efficient-small-model-training/) for how much to
unfreeze and why Imagenette is an invalid transfer target.

## Apple Silicon (PyTorch MPS backend)

- Trains on the M-series GPU via Metal. State as of 2026 (PyTorch 2.13):
  works well for mainstream CNN training, still officially experimental —
  incomplete op coverage (`PYTORCH_ENABLE_MPS_FALLBACK=1` falls back to CPU
  per-op), no float64, bf16 needs macOS 14+, model must fit in unified
  memory, `torch.compile` support limited.
- **Measured on this household's M4 MacBook (16 GB), 2026-07-17**: MNIST
  CNN (1.6M params, 3 epochs, one-cycle SGD) → 99.07% test accuracy in
  **21.0 s on MPS vs 100.6 s on CPU** (~4.8× GPU speedup). Consistent with the
  published M2 measurement (~2.5× for the same workload). CIFAR-10 with a
  6.6M-param ResNet9 from scratch, 10 epochs on MPS → **88.5% in 8.5 min**
  (~48 s/epoch); fine-tuning an ImageNet-pretrained MobileNetV3-Small
  (1.5M params with the 10-class fine-tune head — the 1000-class ImageNet
  backbone is 2.54M, see the throughput table below; 160 px inputs), 3 epochs
  on MPS → **92.5% in 4.9 min** —
  transfer learning beat from-scratch on both time and accuracy.
- Apple Silicon is far below discrete NVIDIA on heavy workloads (M1 Pro
  ~14× slower than an A6000 on a large contrastive job; systematic gap
  analysis in arXiv:2501.14925) — irrelevant at small-CNN scale, **decisive
  at ImageNet scale**.

### Measured ImageNet-resolution training throughput (M4, MPS, 224 px, 2026-07-17)

Synthetic inputs, `mps.synchronize()`-bracketed timing — a pure compute
ceiling; real runs can only be slower.

| Model | Params | img/s | h/epoch (1.28M imgs) |
|---|---|---|---|
| MobileNetV3-Small | 2.54 M | 317.2 | 1.12 |
| MobileNetV3-Large | 5.48 M | 129–138 | 2.58–2.76 |
| MobileNetV4-Conv-Small | 3.77 M | 309.5 | 1.15 |
| MobileNetV4-Conv-Medium | 9.72 M | 94.0 | 3.79 |

Resolution dominates on this hardware (MobileNetV3-Large, batch 128):
**982 img/s @64 px → 377 @128 → 243 @160 → 129 @224 (7.6× span)**.
Run-to-run variance ~7% at sustained load (thermal throttling).

### On a laptop the GPU binds — not JPEG decode

Counter to the standard wisdom (FFCV: decode is ~16× the read cost; DALI
exists to offload it), the **M4's 10 CPU cores are wildly overprovisioned**
relative to its GPU. Measured with the real ImageFolder/DataLoader/
RandomResizedCrop(224) path over ImageNet-realistic 132 KB JPEGs:
**2519 img/s at 10 workers** — vs the GPU's 129–138 img/s, i.e. **18×
headroom**. FFCV/DALI-class tooling buys nothing on this machine.

The decode wall is real, but it appears on **free-tier notebooks** (a T4/P100
paired with ~2–4 weak vCPUs), not on the laptop. The laptop's problem is
compute; the notebook's problem is decode.

**Headroom shrinks as you freeze more** (pure-compute ceiling per fine-tuning
arm vs the 2519 img/s loader): full 145.0 img/s → **17.4×**; partial 352.9 →
7.1×; head-only 557.6 → **4.5×**. A frozen trunk backprops through less, so the
GPU speeds up while decode cost is unchanged — **3.8× erosion**, though it never
flips on this machine. ⚠️ Head-only on a free-tier notebook (fastest GPU-side
arm + weakest CPU) is where decode would bind first — direction argued from
measured erosion, no notebook benchmarked.

## CPU-only

- MNIST to 99% is a minutes-scale CPU job (measured: 100.6 s for 3 epochs on
  the M4's CPU). Entry GPUs beat ultrabook CPUs by only 4–6× on MNIST.
- CIFAR-10 from scratch on pure CPU: expect an hour-to-overnight for a small
  ResNet; minutes if fine-tuning a pretrained backbone instead.

## Consumer NVIDIA GPU (buy once, no recurring cost)

- Tim Dettmers' GPU guide (still the standard reference) recommends buying
  used; small cards suffice for Kaggle-scale work. A 6 GB GTX 1660 Ti
  (2019, no Tensor Cores) trains MNIST to 99% in under a second
  (tuomaso/train_mnist_fast). Any 10-series+ card with 6 GB+ is overkill for
  small CNNs. Mixed precision (2–3×) only pays on Tensor-Core cards
  (Volta/Turing/Ampere+) — relevant when choosing a used card.

## Rules of thumb

- MNIST: ~4.3 TFLOPs per training epoch for the stock PyTorch example CNN;
  one tuned epoch reaches 99%.
- CIFAR-10 to 94%: total training compute bounded under ~0.8 PFLOPs (the
  2.59 s A100 speedrun record, see [Efficient small-model training](/wiki/efficient-small-model-training/));
  ordinary runs are 10–100× less efficient and still finish in minutes on a
  laptop GPU.
- **ImageNet-1k epoch on an M4 ≈ 2.6 h at 224 px, ≈ 0.36 h at 64 px.** Multiply
  by the recipe's epoch count before assuming anything is feasible — reference
  epoch counts are 500–3600, not 10.
- **Laptop budget ceiling ≈ 24–36 continuous hours.** Anything above that
  wants an owned GPU or a rented one; note FFCV trains ResNet-18 to 72.4% on
  full ImageNet in **3.1 hours on one A100**, so a single consumer GPU is a
  genuinely different regime from a laptop — the blocker there becomes the
  144 GB gated download, not compute ([ImageNet-scale training logistics](/wiki/imagenet-scale-training-logistics/)).
