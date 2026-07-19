---
share: true
title: IQA / IAA Evaluation Metrics
categories:
  - Wiki
tags:
  - image-quality-assessment
  - image-aesthetics
  - benchmarks
  - evaluation-metrics
author: claude
type: concept
created: 2026-07-18
updated: 2026-07-18
sources:
  - "[[2026-07-18-image-quality-and-aesthetic-assessment-foundations]]"
  - "[[2026-07-18-image-quality-and-aesthetic-assessment-convergence-mllm-scoring]]"
aliases:
  - SRCC
  - SROCC
  - PLCC
  - KRCC
  - MOS
  - DMOS
  - EMD loss
  - IQA metrics
---

# IQA / IAA Evaluation Metrics

How [IQA](/wiki/image-quality-assessment/) and [IAA](/wiki/image-aesthetic-assessment/)
methods are scored. **Not by accuracy** — by how well a predicted score
*correlates* with the human MOS across a test set, along two independent
qualities: **monotonicity** (does higher predicted mean higher human?) and
**linear accuracy** (are the values close after alignment?). Four numbers,
always reported together, plus the training losses and label definitions that
make them meaningful.

## Labels: MOS and DMOS

- **MOS** (Mean Opinion Score) — mean of raw human opinion scores; higher
  usually = better. Used by TID2013, KADID, KonIQ, CLIVE, SPAQ, FLIVE, and the
  IAA sets ([IQA / IAA Datasets](/wiki/iqa-iaa-datasets/)).
- **DMOS** (Difference MOS) — mean of *difference* scores: per subject,
  score(reference) − score(test), then averaged. Cancels per-subject bias and
  normalises against reference quality. **Higher DMOS = more degradation =
  worse** (opposite polarity to MOS). Used by LIVE IQA and CSIQ. A model
  reported against DMOS predicts *degradation*; reconcile polarity before any
  cross-dataset comparison.

The label is intrinsically uncertain — a MOS is a lossy summary of a noisy vote
histogram, and the spread (rater disagreement) is signal, not noise. Methods
that model the distribution, the ranking, or a soft target beat methods that
regress the mean as if exact.

## The four reported correlation metrics

### SRCC / SROCC — Spearman rank-order correlation
Pearson correlation on the **ranks** of predicted vs. subjective scores.
Measures **monotonicity**, insensitive to any monotonic rescaling. Range
[−1, 1]. The usual headline number, because applications consume *ordering*.

### PLCC — Pearson linear correlation, after logistic re-mapping
Linear correlation measuring **accuracy** — but per VQEG convention, first fit
a **5-parameter logistic** mapping from raw prediction $x$ to the MOS scale,
then compute PLCC (and RMSE) on the mapped values:

$$
g(x) = \beta_1\left(\frac{1}{2} - \frac{1}{1 + e^{\beta_2 (x - \beta_3)}}\right) + \beta_4\, x + \beta_5
$$

$\beta_1 \ldots \beta_5$ least-squares fit to the subjective scores. The
logistic core handles the saturating quality scale; the $\beta_4 x + \beta_5$
tail makes it the *5*-parameter variant. **Without this step PLCC would punish
a perfectly-ranked predictor for outputting on a different scale** — it is the
silent default even when papers omit mentioning it.

### KRCC / KROCC — Kendall rank correlation

$$
\tau = \frac{n_{\text{concordant}} - n_{\text{discordant}}}{\tfrac{1}{2}\,n(n-1)}
$$

Stricter monotonicity than Spearman; numerically lower for the same data.
Reported as a robustness cross-check.

### RMSE
Root-mean-square error between **mapped** predictions and subjective scores —
accuracy in MOS units, lower better. Scale-dependent, comparable only *within*
one dataset's MOS range.

## Training losses that respect the label

### EMD — distribution loss (NIMA)
For AVA-style distribution labels, predict a probability vector over the $N$
ordered rating buckets and train with squared Earth Mover's Distance on the
*cumulative* distributions:

$$
\text{EMD}(p, \hat{p}) = \left(\frac{1}{N}\sum_{k=1}^{N} \left|\mathrm{CDF}_p(k) - \mathrm{CDF}_{\hat p}(k)\right|^{r}\right)^{1/r}, \quad r = 2
$$

Unlike cross-entropy (unordered classes), EMD respects order — predicting "8"
for a true "9" costs less than "2". The right inductive bias for an ordinal
score; empirically improves both correlation and binary accuracy over scalar
regression.

### Ranking / pairwise losses
Datasets tracking rater identity (AADB) or using pairwise comparison (TID2013)
support losses on relative order (A > B), robust to per-rater scale offsets.
Recurs across IQA and IAA; in the LLM era, Compare2Score and DeQA-Score are
the descendants.

### Levels-as-tokens score (MLLM era)
[MLLM scorers](/wiki/multimodal-llm-visual-scoring/) (Q-Align) do not regress a number:
they predict one of five discrete words *excellent…bad* → {5…1} and read the
score as the softmax-weighted average over just those level-token logits,
$S = \sum_{i} p_{\ell_i}\, i$. An LLM predicts *tokens*, so ordinal words are
in-distribution where a continuous numeral is not — the discrete syllabus also
*generalizes* better (Q-Align: +52.8 % SRCC on a SPAQ→KADID transfer). This is
the era's scoring mechanism; DeQA-Score refines it to a soft score *distribution*
for calibration.

## Binary accuracy on AVA — legacy, and criticised
Classic AVA papers threshold the mean at **5.0** for good/bad labels and report
classification accuracy (~66–82%). Poor metric: (1) discards magnitude (4.9 and
1.0 both "bad"); (2) inflated by class imbalance (scores cluster near 5–6, so a
near-constant predictor scores high); (3) threshold-sensitive. Reported only
for historical continuity.

## Reference frame — what "good" looks like

**NR-IQA on KonIQ-10k** (Q-Align / Q-Insight tables): CNNs ≈ 0.86–0.91 SRCC
(NIMA 0.859, HyperIQA 0.906) → transformers ≈ 0.92 (MUSIQ 0.916 paper-protocol;
some re-evaluations ~0.929) → CLIP ≈ 0.92
(LIQE 0.919) → MLLM ≈ **0.94** (Q-Align 0.940, DeQA-Score 0.941/0.953 PLCC).

**IAA on AVA:** NIMA 0.612 SRCC → MUSIQ 0.726 → LIQE 0.776 → VILA-R 0.774 →
Q-Align **0.822** (first to clearly break 0.80). Aesthetics correlations run well
below IQA. **OneAlign** reaches both ceilings with *one model* (KonIQ 0.941, AVA
0.823, LSVQ video 0.886) — the convergence (R4).

**Cross-dataset is where the MLLM edge shows.** In-domain SRCC saturates near the
label ceiling and hides overfitting; the honest number is train-on-one /
test-on-another. Q-Align KonIQ→CLIVE **0.860** vs HyperIQA 0.785, CLIP-IQA+
0.805 — the pretrained world-knowledge generalizes ([Image Quality Assessment (IQA)](/wiki/image-quality-assessment/)).

**Legacy synthetic sets** are saturated: FR ≈ 0.98 SRCC on LIVE; strong NR
transformers ≈ 0.97–0.98 on LIVE-synthetic, ≈ 0.93–0.95 on TID2013 (stated as
ballpark bands). This saturation is why the field moved to authentic sets.

## The five-era method arc (shared by IQA and IAA)

The skeleton both problems climbed; R2/R3/R4 fill in the branches.

1. **Handcrafted NSS** (2011–2016) — BRISQUE, NIQE, DIIVINE: hand-built priors
   + SVR.
2. **Deep CNN** (2014–2020) — NIMA, DBCNN, HyperIQA: learned features.
3. **Transformer / ViT** (2021–2023) — MUSIQ, MANIQA, TReS: full-resolution
   attention.
4. **Vision-language / CLIP** (2022–2023) — CLIP-IQA, LIQE: text-prompted
   quality.
5. **Multimodal-LLM scoring** (2023–2026) — Q-Bench, Q-Instruct, Q-Align:
   describe-then-rate via the levels-as-tokens score, unifying IQA + IAA + video
   in OneAlign (survey R4, [Multimodal-LLM Visual Scoring](/wiki/multimodal-llm-visual-scoring/)).

Each rung raised the KonIQ ceiling and improved cross-dataset generalisation —
the property synthetic-era methods most lacked.

> **Is a correlation coefficient still the right target?** Once an MLLM can
> *describe* and *compare* quality in language ([Multimodal-LLM Visual Scoring](/wiki/multimodal-llm-visual-scoring/)),
> a single SRCC on one dataset throws away most of what it does and rewards
> web-scale-pretraining contamination. The convergence-era frontier is arguably
> faithful description/comparison (Q-Bench-style), not a higher number.

## See also

- [Image Quality Assessment (IQA)](/wiki/image-quality-assessment/) · [Image Aesthetic Assessment (IAA)](/wiki/image-aesthetic-assessment/) ·
  [IQA / IAA Datasets](/wiki/iqa-iaa-datasets/) · [Multimodal-LLM Visual Scoring](/wiki/multimodal-llm-visual-scoring/)
