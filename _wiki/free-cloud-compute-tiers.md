---
share: true
title: Free cloud compute tiers
categories:
  - Wiki
author: claude
type: concept
created: 2026-07-17
updated: 2026-07-17
sources:
  - "[[2026-07-17-where-no-server-no-cloud-bill-stops-being-true]]"
aliases:
  - free GPU tiers
  - free notebook compute
---

# Free cloud compute tiers

Cloud compute that never bills — for ML training without owning hardware.
Quota numbers rot fast; every figure below was verified against the cited
official page on **2026-07-17**. Re-verify before relying on any of them.
Complement of [ML training on consumer hardware](/wiki/ml-training-on-consumer-hardware/) (the strictly-local path).

## The tiers, as verified 2026-07-17

- **Kaggle Notebooks** — the only major tier with published, citable quotas
  (kaggle.com/docs/notebooks): **~30 h/week GPU** ("or sometimes higher"),
  choice of 1×P100 or 2×T4 (4 cores, 29 GB RAM), TPU v3-8 at 20 h/week,
  12 h sessions (9 h TPU), 20 GB persistent disk. GPU access requires phone
  verification. ⚠️ Kaggle's own docs self-contradict on idle timeout: 20 min
  (notebooks doc) vs 60 min (efficient-gpu-usage doc) — plan for the
  stricter.
- **Google Colab free** — real but **deliberately unpublished** limits: the
  official FAQ commits only to "at most 12 hours" sessions and says limits
  vary over time. The widely-cited free T4 (16 GB), TPU v5e, ~15–30 h/week,
  ~90 min idle timeout are community-measured folklore, not contract.
- **Lightning AI free** — 15 credits/month ("up to 80 GPU h" marketing;
  ~22 T4-h by third-party math), single GPU, 4 h studio restarts, 50 GB
  storage, phone verification required.
- **Modal starter** — $30 free credits/month (≈50 T4-h), no card required;
  serverless functions rather than notebooks (steeper learning curve). Bills
  only if payment is explicitly added.
- **AWS SageMaker Studio Lab** — free T4 without an AWS account, but
  **closed to new signups 2026-07-30**; existing users unaffected, feature-frozen.
- **HF Spaces ZeroGPU** — 5 min/day, 60 s max per function: effectively
  inference-only, not a training path.
- **Paperspace Gradient free** — nominally alive under DigitalOcean but being
  folded into GPU Droplets; too churny to recommend.

## The verdict is use-case-dependent

**For fine-tuning, the free tier is comfortably over-provisioned** (costed at
measured M4 rates, 2026-07-17):

| Workload | Cost | Runs per 30-h week | Runs per 12-h session |
|---|---|---|---|
| Fine-tune Flowers-102 (full) | 4.8 min | **373** | 149 |
| Fine-tune Flowers-102 (partial) | 2.5 min | 714 | 286 |
| Fine-tune Food-101 (75,750 imgs) | 3.1 h | 9.8 | 3.9 |
| **Pre-train ImageNet (600 ep)** | **69 days** | **0.018** | — |

A Flowers-102 fine-tune is **0.7% of one session**; even Food-101 fits ~4×
inside a single session with its 5 GB under the 20 GB persistent allowance.
**The session cap and ephemeral disk never engage** — so for fine-tuning,
neither checkpoint-chaining nor the ToS question below arises at all.

## Where the free tier stops working

Verified 2026-07-17. **For pre-training the quota is a hard ceiling, not a slow
lane** — the arithmetic simply does not close:

| Recipe | Epochs | Kaggle-weeks @30 GPU-h | 12-h sessions to chain |
|---|---|---|---|
| torchvision `mobilenet_v3_large` | 600 | **55.2** | 138 |
| timm `mobilenetv3_large_100.ra4` | 3600 | 331.3 | 828 |

(Costed at measured M4 throughput; Kaggle's P100/T4 is not an M4, so these are
order-of-magnitude, not a Kaggle benchmark. A 2× faster GPU turns a year into
six months — the conclusion survives.)

**Checkpoint-and-resume across sessions: mechanically partial, practically
no.** Kaggle persists 20 GB via Save Version — enough for MobileNet
checkpoints, **not** for the 144 GB ImageNet copy, which must be re-attached
every session ([ImageNet-scale training logistics](/wiki/imagenet-scale-training-logistics/)). Colab deletes the VM
entirely.

**ToS, stated precisely** (Colab [FAQ](https://research.google.com/colaboratory/faq.html)):
a single user hand-restarting a notebook and resuming from a checkpoint is
**not** on the prohibited list. But Colab explicitly prohibits every mechanism
that would make it practical at scale — "using multiple accounts to work around
access or resource usage restrictions… employing techniques such as
containerization to circumvent anti-abuse policies", and for the free tier
"remote control such as SSH shells… bypassing the notebook UI". Combined with
"Colab prioritizes users who are actively programming in a notebook", the
synthesis: **a manual chain is permitted-but-deprioritized; an automated one is
prohibited.** 129 sessions is not something a human hand-restarts. Google's own
stated remedy is "purchasing a dedicated VM at GCP Marketplace" — i.e. stop
using the free tier.

⚠️ **Kaggle's ToS is unverified**: kaggle.com/terms is client-rendered and
could not be read by any fetch method tried. No claim is made here about
Kaggle's policy on session chaining — read it in a browser before relying on
one. (Kaggle's *mechanics* support checkpoint-resume better than Colab's; that
is an inference from documented behavior, not ToS text.)

⚠️ **Colab's disk quota is unknowable**: Google "does not publish these limits."
The widely-repeated "~100 GB" is not sourceable; community reports scatter
across 35–78 GB. Do not state a number.

## Rules of thumb

- For small CNNs, one Kaggle week (30 T4-hours) is thousands of CIFAR-10
  training runs — quota is never the constraint at that scale.
- **For pre-training at ImageNet scale the free tier is not a path at all** —
  not slow, not awkward: ~55–330 weeks of quota and 138–828 chained sessions
  with no persistent room for the 144 GB dataset.
- **For fine-tuning it is over-provisioned** — 373 Flowers-102 runs per week's
  quota. This is the use case the free tier actually fits, and the
  recommendation flips accordingly.
- Prefer Kaggle when a run outgrows the laptop; Colab free as fallback.
- "Free tier" here means never-bills-without-opt-in; anything requiring a
  credit card on file was excluded.
