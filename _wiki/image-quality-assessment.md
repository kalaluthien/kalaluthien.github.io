---
share: true
title: Image Quality Assessment (IQA)
categories:
  - Wiki
tags:
  - image-quality-assessment
  - computer-vision
  - benchmarks
author: claude
type: concept
created: 2026-07-18
updated: 2026-07-18
sources:
  - "[[2026-07-18-image-quality-and-aesthetic-assessment-foundations]]"
  - "[[2026-07-18-image-quality-assessment-methods-nss-to-clip]]"
  - "[[2026-07-18-image-quality-and-aesthetic-assessment-convergence-mllm-scoring]]"
aliases:
  - IQA
  - BIQA
  - NR-IQA
  - blind image quality assessment
---

# Image Quality Assessment (IQA)

Predicting **how technically degraded an image is** relative to some notion of
fidelity — noise, blur, compression, banding, transmission artifacts. The
output is a scalar quality score meant to track the human **Mean Opinion
Score** ([MOS](/wiki/iqa-iaa-evaluation-metrics/)). Distinct from
[Image Aesthetic Assessment (IAA)](/wiki/image-aesthetic-assessment/) (beauty, not fidelity) and from the perceptual
quality of generative output (realness, not fidelity) — see the contrast table
below. The two problems share their shape, metrics, and method arc, which is
why later work scores both with one model.

## Two taxonomic axes

IQA is cut along two independent axes; every method announces itself by where
it sits on both.

### Axis 1 — reference availability

- **Full-Reference (FR).** Pristine original *and* distorted copy, pixel-
  aligned; measure perceptual distance between them. PSNR, SSIM (classical);
  LPIPS, DISTS (learned). Largely solved — SRCC ≈ 0.98 on legacy sets. Used
  inside codecs and super-resolution where the clean source is in hand.
- **Reduced-Reference (RR).** Only a compact set of features from the original
  (side information over a channel). Least-studied branch; narrow use case.
- **No-Reference / Blind (NR / BIQA).** Only the image. The reference prior
  must live *inside the model* — classically as hand-crafted Natural Scene
  Statistics, now learned from data. **The hard, deployable case**: every real
  deployment (phone uploads, web images, generated images) has no reference.
  This is where the field concentrated.

### Axis 2 — distortion origin

- **Synthetic.** Known operators (JPEG, blur, noise) applied to pristine
  images at graded levels. Clean design matrix; supports FR *and* NR. LIVE,
  TID2013, CSIQ, KADID-10k.
- **Authentic / in-the-wild.** Real photos with entangled, unlabelled
  distortion mixtures and **no clean reference** — NR only. KonIQ-10k, CLIVE,
  SPAQ, FLIVE.

> **Why the field left synthetic-only.** Models trained on synthetic
> distortions learned those specific operators and collapsed on real photos
> (overfit-to-the-degradation-model). The ~2016–2020 move to authentic sets is
> the central dataset transition in IQA — a method's year tells you which
> distortion world it was built for. See [IQA / IAA Datasets](/wiki/iqa-iaa-datasets/).

## IQA vs IAA vs generative-perceptual quality

Three orthogonal axes, not one scale — an image can score high on any subset.

| | Question | Governed by | Reference exists? |
|---|---|---|---|
| **IQA** | How degraded? | noise, blur, compression | sometimes (FR/RR); usually not (NR) |
| **IAA** | How beautiful? | composition, lighting, subject | never |
| **Generative-perceptual** | How real? | artifact signatures, texture | distributional (FID, LPIPS over sets) |

A film photograph can be low-IQA (grainy) yet high-IAA (a masterpiece); a
diffusion sample can be high on both yet obviously fake. The generative axis is
measured distributionally (a *set* of generated vs. a *set* of real images),
so it sits adjacent to this concept rather than inside it.

## Method lineage (NR-IQA)

Both IQA and IAA climbed the same five-rung ladder; the shared skeleton lives
in [IQA / IAA Evaluation Metrics](/wiki/iqa-iaa-evaluation-metrics/). The IQA-specific detail — architectures,
losses, per-dataset SRCC/PLCC, code repos — is survey report
[R2](/posts/image-quality-assessment-methods-nss-to-clip/). The durable
takeaway is that **each era removes a specific dependency of the previous one**
(this failure/repair chain, not the SRCC numbers, is the model):

1. **Handcrafted NSS** (2011–2016) — DIIVINE, BLIINDS-II, BRISQUE, NIQE
   (opinion-unaware — trained on pristine images only, no MOS), CORNIA/HOSA
   (codebook). Fit generalised-Gaussian parameters of MSCN coefficients, regress
   with SVR. Still the speed/interpretability baseline. **Broke on** authentic
   photos (compound distortion ≠ clean synthetic operators). *KonIQ SRCC ≈ 0.66.*
2. **Deep CNN** (2014–2020) — CNNIQA/WaDIQaM (patch training, the small-data
   workaround), then the transfer-learning turn: DBCNN (two-stream bilinear:
   synthetic-distortion + ImageNet-semantic), HyperIQA (content-adaptive
   hyper-network generating per-image weights), PaQ-2-PiQ (patch-to-picture,
   FLIVE paper), MetaIQA (meta-learning). **Fixed** small-data via ImageNet
   transfer. **Broke on** fixed-resolution resize destroying quality cues, local
   receptive fields. *KonIQ ≈ 0.88–0.91.*
3. **Transformer / ViT** (2021–2023) — MUSIQ (multi-scale, **native-resolution**
   — the era's key IQA-specific fix), MANIQA (channel attention, won NTIRE 2022,
   strong on GAN distortion), TReS (relative-ranking + self-consistency), Re-IQA
   (contrastive content+quality experts). **Fixed** resize + global perception;
   needs the big authentic datasets. **Broke on** still needing MOS labels.
   *KonIQ ≈ 0.91–0.92.*
4. **Vision-language / CLIP** (2022–2023) — CLIP-IQA (antonym "Good/Bad photo"
   prompts, **zero-shot ≈ 0.70 KonIQ with no labels**), LIQE (multitask CLIP:
   joint scene + distortion + quality, ≈ 0.92). **Fixed** the label dependency:
   ask a pretrained VLM instead of training a regressor. *KonIQ ≈ 0.92.*
5. **Multimodal-LLM scoring** (2023–2026) — stop training a regressor; **teach a
   pretrained MLLM to rate.** Q-Bench diagnosed that general MLLMs (GPT-4V)
   *perceive* low-level attributes but *score* imprecisely; Q-Instruct taught them
   to *describe* quality; **Q-Align** (ICML 2024, on mPLUG-Owl2) teaches the five
   discrete words *excellent…bad* → {5…1} and reads a continuous score as the
   softmax-weighted average over just those level tokens — because an LLM predicts
   *tokens*, ordinal words are in-distribution where a numeral is not (the
   **levels-as-tokens trick**). **Fixed** the label dependency's successor — the
   *bespoke-model-per-task* dependency: **OneAlign** is one model for IQA + IAA +
   video VQA, at/above task-specific SOTA. **Dents** R2's cross-dataset frontier:
   Q-Align KonIQ→CLIVE **0.860** vs HyperIQA 0.785, CLIP-IQA+ 0.805 — the
   pretrained world-knowledge generalizes. In-domain **KonIQ 0.940/0.941**;
   DeQA-Score (2025) 0.941/0.953. Full detail and the mechanism live in
   [Multimodal-LLM Visual Scoring](/wiki/multimodal-llm-visual-scoring/); **survey R4**. *Still open:* score
   calibration/reproducibility, explanation hallucination, benchmark
   contamination via web-scale pretraining, and compute (billions of params vs
   MANIQA's ≈ 20 MB).

Also **FR perceptual metrics** (LPIPS, DISTS, PieAPP) — a parallel branch of
deep features as a perceptual *distance* that beats SSIM; reused across the
vision field as differentiable training losses and eval for super-resolution /
generative work (see R2), not as blind scorers.

*Reference ceiling:* NR-IQA on KonIQ-10k reaches ≈ **0.94 SRCC/PLCC** at the
MLLM frontier (Q-Align, DeQA-Score); transformers/CLIP ≈ 0.92; CNNs ≈ 0.88–0.91;
NSS ≈ 0.66. Legacy synthetic sets are saturated (≈ 0.97–0.98 SRCC on LIVE).
**The unsolved part is cross-dataset generalisation:** train KonIQ → test CLIVE
drops in-domain ≈ 0.91 to ≈ 0.72–0.79 (HyperIQA 0.906 → 0.785). Numbers and
their definitions live in [IQA / IAA Evaluation Metrics](/wiki/iqa-iaa-evaluation-metrics/).

## See also

- [Image Aesthetic Assessment (IAA)](/wiki/image-aesthetic-assessment/) — the aesthetic-quality sibling problem.
- [Multimodal-LLM Visual Scoring](/wiki/multimodal-llm-visual-scoring/) — where the IQA and IAA tracks merge (rung 5).
- [IQA / IAA Datasets](/wiki/iqa-iaa-datasets/) — the dataset spine (LIVE → KonIQ → FLIVE).
- [IQA / IAA Evaluation Metrics](/wiki/iqa-iaa-evaluation-metrics/) — SRCC/PLCC/KRCC/RMSE, EMD, the method arc.
- [Mobile photo ML features (Apple vs Samsung)](/wiki/mobile-photo-ml-features/) — how in-product "best-shot"/quality analysis
  actually ships in iPhone/Galaxy (and why neither vendor names an IQA/IAA model).
