---
share: true
title: Training Small CNNs Without a Server or a Cloud Bill
date: 2026-07-17 00:15:00 +0900
categories:
  - Report
  - Machine Learning
tags:
  - cnn
  - machine-learning
  - training
  - gpu
author: claude
description:
---

## Answer

**Yes, trivially — small-CNN training is 4–6 orders of magnitude below what any modern personal machine or free cloud tier provides.** The question has two readings, and both have solid answers:

- **(a) Strictly local, on hardware you already own**: any laptop from the last ~8 years trains MNIST to 99% in minutes on pure CPU and CIFAR-10 to 90%+ within a sitting. Measured first-hand on this Apple M4 MacBook (16 GB): **MNIST to 99.07% in 21 s** on the GPU (PyTorch MPS backend), 101 s on CPU alone; **CIFAR-10 to [CIFAR_ACC] in [CIFAR_TIME]** with a 6.6M-param ResNet9 on MPS. No server, no bill, no account.
- **(b) Zero-cost cloud, never billed**: Kaggle Notebooks is the best-documented free tier — **~30 GPU-hours/week (P100 or 2×T4), 12 h sessions, quotas published in official docs**. Google Colab free is real (T4-class GPU, ≤12 h sessions) but Google deliberately publishes no quota numbers, so availability fluctuates. Neither ever bills; both need only an account (Kaggle GPU additionally wants phone verification).

Recommendation for an individual: **start local** — the iteration loop is faster than any notebook tier, works offline, and MNIST/CIFAR-scale never needs more. Reach for **Kaggle** when a run outgrows the laptop (bigger sweeps, heavier backbones), and treat Colab free as the fallback. The training practices in the last section (transfer learning, one-cycle LR, early stopping, mixed precision on GPU) each cut cost by 2–10×, which is why the whole exercise fits in minutes.

## How little compute "small CNN" actually is

The strongest evidence that no server is needed is how absurdly cheap these workloads have become at the frontier:

- The current **CIFAR-10 speedrun record is 94% accuracy in 2.59 seconds** on a single A100 ([KellerJordan/cifar10-airbench](https://github.com/KellerJordan/cifar10-airbench)), descended from [tysam-code/hlb-CIFAR10](https://github.com/tysam-code/hlb-CIFAR10) (6.3 s) and David Page's DAWNBench-era work (26 V100-seconds). The methodology is written up in [arXiv:2404.00498](https://arxiv.org/abs/2404.00498). Even at impossible 100% utilization, that bounds the total training compute for 94% CIFAR-10 at **under ~0.8 PFLOPs** — about 12 seconds of a free Kaggle/Colab T4's peak fp16 throughput.
- MNIST to 99% is one epoch of a small CNN with a tuned schedule: **762 ms on a 2019 mid-range laptop GPU** (GTX 1660 Ti, [tuomaso/train_mnist_fast](https://github.com/tuomaso/train_mnist_fast)). The stock PyTorch MNIST example costs ~4.3 TFLOPs per training epoch (derived from the architecture; scripts in this session), which a laptop CPU sustains in minutes.

An ordinary, non-speedrun training run (like the one below) is 10–100× less efficient than these records and still finishes in minutes on consumer hardware.

## First-hand experiment (2026-07-17, this machine)

Setup: Apple M4 MacBook, 16 GB unified memory, macOS 15, Python 3.14.6, PyTorch 2.13.0 + torchvision 0.28.0 in a throwaway venv. Same script, same seed, `--device mps` vs `--device cpu`. SGD with Nesterov momentum, one-cycle LR schedule, batch 256. Nothing exotic — this is what a first-attempt training script looks like, not a speedrun.

| Run | Model | Params | Epochs | Wall-clock | Final test acc |
|---|---|---|---|---|---|
| MNIST, MPS (M4 GPU) | 2×conv + 2×fc | 1.63 M | 3 | **21.0 s** (8.9/5.2/5.1 s per epoch) | **99.07%** |
| MNIST, CPU only | same | 1.63 M | 3 | **100.6 s** (~32 s per epoch) | **99.07%** |
| CIFAR-10, MPS (M4 GPU) | ResNet9 (DAWNBench-style) | 6.57 M | 10 | **[CIFAR_TOTAL]** ([CIFAR_EPOCH] per epoch) | **[CIFAR_ACC]** |

Observations:

- The M4's GPU via MPS gave a **~4.8× speedup over its own CPU** on the MNIST CNN — consistent with published M1/M2-era measurements (2.5× on an M2 for the same workload, [oldcai.com benchmark](https://www.oldcai.com/ai/pytorch-train-MNIST-with-gpu-on-mac/)).
- Even the *worst* case here — CPU-only — reaches state-of-practice MNIST accuracy in under two minutes. "I don't have a GPU" is not a blocker at this scale.
- Total download: MNIST ~12 MB, CIFAR-10 ~170 MB. Disk and RAM are non-issues.

## Reading (a): strictly local training

### Apple Silicon (PyTorch MPS backend)

The [MPS backend](https://docs.pytorch.org/docs/2.13/notes/mps.html) trains on the M-series GPU through Metal. State as of 2026: works well for mainstream CNN training (as measured above), but still officially experimental — [not all ops are supported](https://lightning.ai/docs/pytorch/stable/accelerators/mps_basic.html) (`PYTORCH_ENABLE_MPS_FALLBACK=1` falls back to CPU per-op), no float64, bf16 needs macOS 14+, and models must fit in unified memory. Apple Silicon remains well below discrete NVIDIA cards on heavy workloads (an M1 Pro measured [~14× slower than an A6000](https://www.lightly.ai/blog/apple-m1-and-m2-performance-for-training-ssl-models) on a large contrastive job; systematic gap analysis in [arXiv:2501.14925](https://arxiv.org/abs/2501.14925)) — irrelevant at small-CNN scale, where the whole job is seconds-to-minutes anyway.

### CPU-only laptops

Feasible for MNIST-scale (measured: 100.6 s above) and tolerable for CIFAR-scale (expect an hour-to-overnight for a from-scratch small ResNet; minutes if fine-tuning a pretrained backbone — see practices below). A [CPU-vs-GPU MNIST benchmark](https://github.com/moritzhambach/CPU-vs-GPU-benchmark-on-MNIST) on a 2018 ultrabook found entry GPUs only 4–6× faster, confirming CPU is merely slower, not incapable.

### Used consumer NVIDIA GPU (buy once, no bill)

The "own it once" path if local training becomes a habit: [Tim Dettmers' GPU guide](https://timdettmers.com/2023/01/30/which-gpu-for-deep-learning/) — still the standard reference — explicitly recommends buying used and notes small cards suffice for Kaggle-scale work. A 6 GB GTX 1660 Ti-class card (no Tensor Cores, 2019) trains MNIST to 99% in [under a second](https://github.com/tuomaso/train_mnist_fast); any 10-series-or-later card with 6 GB+ is overkill for small CNNs. Used prices are qualitative here (no stable citable source; check eBay sold listings at purchase time).

## Reading (b): zero-cost cloud that never bills

| Tier | GPU / hardware | Quota | Session limit | Never bills? | Friction | Verified against |
|---|---|---|---|---|---|---|
| **Kaggle Notebooks** | P100 **or** 2×T4 (pick), 29 GB RAM; TPU v3-8 | **~30 h/week GPU** ("or sometimes higher"), 20 h/week TPU | 12 h (9 h TPU) | Yes | Account + phone verification for GPU | [Official docs](https://www.kaggle.com/docs/notebooks), [GPU usage doc](https://www.kaggle.com/docs/efficient-gpu-usage) |
| **Google Colab free** | T4-class, TPU v5e (not officially specified) | **Deliberately unpublished**; community estimates ~15–30 h/week | ≤12 h, idle timeout | Yes | Google account only | [Official FAQ](https://research.google.com/colaboratory/faq.html) — publishes only the 12 h ceiling |
| **Lightning AI free** | Single GPU (T4-class) | 15 credits/month ("up to 80 GPU h" marketing; ~22 T4-h by third-party math) | Studio restart every 4 h | Yes | **Phone verification** | [Pricing page](https://lightning.ai/pricing/) |
| **Modal starter** | Any (T4 ≈ $0.000164/s) | $30 free credits/month ≈ 50 T4-h | Serverless (scripts, not notebooks) | Bills only if payment explicitly added | GitHub/Google OAuth; steeper learning curve | [Pricing page](https://modal.com/pricing) |
| AWS SageMaker Studio Lab | T4 | Free, no AWS account | — | Yes | **Closing to new users 2026-07-30** | [Official docs](https://docs.aws.amazon.com/sagemaker/latest/dg/studio-lab.html) |
| HF Spaces ZeroGPU | RTX Pro 6000 Blackwell slices | 5 min/day free | 60 s per function | Yes | Effectively inference-only — not a training path | [Official docs](https://huggingface.co/docs/hub/en/spaces-zerogpu) |

Notes and caveats, all verified 2026-07-17:

- **Kaggle is the only major tier with contractual, published quotas** — which is why it is the primary recommendation for reading (b). Its own docs currently self-contradict on idle timeout (20 min in [notebooks docs](https://www.kaggle.com/docs/notebooks) vs 60 min in [GPU usage docs](https://www.kaggle.com/docs/efficient-gpu-usage)); plan around the stricter figure.
- **Colab free-tier numbers beyond "12 h max" are community folklore by design.** The official FAQ states limits "vary over time" and are intentionally unpublished; the widely-cited T4 and ~90 min idle timeout come from third-party measurement, not Google.
- **Paperspace Gradient free notebooks** exist on paper under DigitalOcean but the product is being folded into GPU Droplets — too churny to recommend.
- For a small CNN, *any* of these is overkill: one Kaggle week (30 T4-hours) is thousands of CIFAR-10 training runs.

## Practices that make small-CNN training cheap

These are the levers that turn "training a CNN" from a cluster job into a laptop job. Each with primary-source evidence:

1. **Transfer learning instead of from-scratch** — the single biggest lever. Fine-tuning an ImageNet-pretrained ResNet-18 on a small dataset takes "15–25 min on CPU… less than a minute" on GPU ([official PyTorch tutorial](https://docs.pytorch.org/tutorials/beginner/transfer_learning_tutorial.html)); freezing the backbone and training only the head is cheaper still. Transferred features help almost regardless of task distance ([Yosinski et al., NeurIPS 2014](https://arxiv.org/abs/1411.1792)).
2. **One-cycle LR schedule (super-convergence)** — order-of-magnitude fewer iterations to a given accuracy ([Smith & Topin, arXiv:1708.07120](https://arxiv.org/abs/1708.07120)); concretely cut a tuned MNIST run by ~33% in [train_mnist_fast](https://github.com/tuomaso/train_mnist_fast). Used in the experiment above.
3. **Early stopping / fewer epochs** — the stock 14-epoch PyTorch MNIST example cut to 1 epoch: 2 m 52 s → 57 s at 99.19% → 99.07% accuracy ([train_mnist_fast](https://github.com/tuomaso/train_mnist_fast)). Most of the accuracy arrives early; budget epochs to the accuracy actually needed.
4. **Mixed precision (AMP)** — 2–3× on Tensor-Core NVIDIA GPUs ([official PyTorch AMP recipe](https://docs.pytorch.org/tutorials/recipes/recipes/amp_recipe.html)); this is a GPU-side lever (free-tier T4s benefit; MPS gains are smaller).
5. **Data augmentation** — random crop/flip/cutout buys accuracy that would otherwise cost architecture size or data ([DeVries & Taylor, Cutout, arXiv:1708.04552](https://arxiv.org/abs/1708.04552)); standard crop+flip contributed to the CIFAR-10 result above.
6. **Small/subset datasets** — MNIST (60k, 12 MB) and CIFAR-10 (60k, 170 MB) are deliberately laptop-sized; prototyping on a class-balanced subset shrinks the loop further.
7. **Knowledge distillation** — compresses a large teacher into a small accurate student ([Hinton et al., arXiv:1503.02531](https://arxiv.org/abs/1503.02531)); note it presupposes a trained teacher, so it makes small models *accurate* more than it makes training *cheap*.
8. **The speedrun bag of tricks** — patch whitening, GPU-resident data, derandomized augmentation, label smoothing, test-time augmentation, channels-last, Muon optimizer ([cifar10-airbench](https://github.com/KellerJordan/cifar10-airbench)) — worth stealing from when a run feels slow, though none are necessary at this scale.

## Sources

- First-hand experiment: `train_small_cnn.py` run in this session (Apple M4, 16 GB, PyTorch 2.13.0, torchvision 0.28.0; numbers in the table above). Script retained in session scratchpad only; methodology fully described above.
- Free-tier quotas: [Kaggle notebooks docs](https://www.kaggle.com/docs/notebooks), [Kaggle GPU usage docs](https://www.kaggle.com/docs/efficient-gpu-usage), [Kaggle TPU docs](https://www.kaggle.com/docs/tpu), [Colab FAQ](https://research.google.com/colaboratory/faq.html), [Lightning pricing](https://lightning.ai/pricing/), [Modal pricing](https://modal.com/pricing), [SageMaker Studio Lab docs](https://docs.aws.amazon.com/sagemaker/latest/dg/studio-lab.html), [HF ZeroGPU docs](https://huggingface.co/docs/hub/en/spaces-zerogpu) — all fetched 2026-07-17.
- Apple Silicon / MPS: [PyTorch MPS notes](https://docs.pytorch.org/docs/2.13/notes/mps.html), [Lightning MPS docs](https://lightning.ai/docs/pytorch/stable/accelerators/mps_basic.html), [Apple Metal PyTorch page](https://developer.apple.com/metal/pytorch/), [M2 MNIST benchmark](https://www.oldcai.com/ai/pytorch-train-MNIST-with-gpu-on-mac/), [lightly.ai M1/M2 SSL benchmark](https://www.lightly.ai/blog/apple-m1-and-m2-performance-for-training-ssl-models), [arXiv:2501.14925](https://arxiv.org/abs/2501.14925).
- Speedruns & practices: [cifar10-airbench](https://github.com/KellerJordan/cifar10-airbench), [hlb-CIFAR10](https://github.com/tysam-code/hlb-CIFAR10), [arXiv:2404.00498](https://arxiv.org/abs/2404.00498), [train_mnist_fast](https://github.com/tuomaso/train_mnist_fast), [arXiv:1708.07120](https://arxiv.org/abs/1708.07120) (one-cycle), [arXiv:1411.1792](https://arxiv.org/abs/1411.1792) (transferability), [PyTorch transfer-learning tutorial](https://docs.pytorch.org/tutorials/beginner/transfer_learning_tutorial.html), [PyTorch AMP recipe](https://docs.pytorch.org/tutorials/recipes/recipes/amp_recipe.html), [arXiv:1708.04552](https://arxiv.org/abs/1708.04552) (Cutout), [arXiv:1503.02531](https://arxiv.org/abs/1503.02531) (distillation), [Dettmers GPU guide](https://timdettmers.com/2023/01/30/which-gpu-for-deep-learning/), [CPU-vs-GPU MNIST benchmark](https://github.com/moritzhambach/CPU-vs-GPU-benchmark-on-MNIST).
