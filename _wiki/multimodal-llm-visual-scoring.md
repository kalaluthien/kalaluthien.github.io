---
share: true
title: Multimodal-LLM Visual Scoring
categories:
  - Wiki
author: claude
type: concept
created: 2026-07-18
updated: 2026-07-18
sources:
  - "[[2026-07-18-image-quality-and-aesthetic-assessment-convergence-mllm-scoring]]"
aliases:
  - MLLM scoring
  - LMM visual scoring
  - Q-Align
  - OneAlign
  - Q-Bench
  - Q-Instruct
  - levels-as-tokens
---

# Multimodal-LLM Visual Scoring

The 2023–2026 convergence era of [IQA](/wiki/image-quality-assessment/) and
[IAA](/wiki/image-aesthetic-assessment/): the field **stopped training a bespoke
regressor per task and started teaching a pretrained multimodal large language
model (MLLM / LMM) to rate.** The result is one model that scores technical
quality, aesthetics, and video quality at or above task-specific SOTA — R1's
"same MOS-prediction machine, opposite questions" thesis made literal. This page
holds the material that spans *both* tracks; the IQA- and IAA-specific rungs
below it live in [Image Quality Assessment (IQA)](/wiki/image-quality-assessment/) and [Image Aesthetic Assessment (IAA)](/wiki/image-aesthetic-assessment/).
Full detail is survey report
[R4](/posts/image-quality-and-aesthetic-assessment-convergence-mllm-scoring/).

## The pivot

A pretrained MLLM already carries a rich, general prior over what images are and
what people say about them (from web-scale image-text). So the task shrinks from
*"train a quality model"* to *"elicit and align the opinion the model already
has."* This is the terminus of R2's CLIP on-ramp (CLIP-IQA, LIQE) and R3's VILA
on-ramp — and it generalizes both.

It dissolves **both** frontier problems with one instrument — MLLM world-knowledge
plus language grounding:

- **IQA's cross-dataset generalization gap** ([Image Quality Assessment (IQA)](/wiki/image-quality-assessment/), R2):
  the external prior is not perturbed by any single dataset's camera/rater
  population, so it transfers. Q-Align holds **KonIQ→CLIVE 0.860** where HyperIQA
  fell to 0.785.
- **IAA's underdetermined scalar / subjectivity** ([Image Aesthetic Assessment (IAA)](/wiki/image-aesthetic-assessment/),
  R3): the MLLM's native output is *language*, so it can hold attributes, taste,
  and rater disagreement a scalar erased — then collapse them to a number if
  wanted.

The qualitative leap is **explainability**: a score *and* a natural-language
critique from one forward pass — auditable quality, defensible aesthetics.

## The levels-as-tokens trick (Q-Align)

The key mechanism, and why it works. Instead of regressing a number:

1. **Reframe the label as five discrete words.** Human studies
   ([IQA / IAA Evaluation Metrics](/wiki/iqa-iaa-evaluation-metrics/) ITU protocols) asked raters for a category, not
   a real number. Q-Align teaches the LMM exactly *excellent, good, fair, poor,
   bad* → $\{5,4,3,2,1\}$. The training target is the *word*, so the loss lives in
   the model's native next-token prediction — no regression head.
2. **Convert the level distribution to a continuous score at inference.** Softmax
   over just the five level-token logits, then a probability-weighted average
   (Q-Align Eq. 4):

   $$
   S = \sum_{i=1}^{5} p_{\ell_i}\cdot i,\qquad
   p_{\ell_i} = \frac{e^{\,x_{\ell_i}}}{\sum_{j=1}^{5} e^{\,x_{\ell_j}}}
   $$

**Why discrete words beat regressing a numeral directly:** an LLM is a token
predictor, not a function approximator — ordinal adjectives are in-distribution, a
continuous numeral is not (Q-Align Table 1: **96–100 % of LMMs spontaneously
answer a quality question with a word, not a number**). It also matches how the
labels were made (MOS = average of categories), and it *generalizes* far better —
the discrete-level syllabus beats direct score regression by **+52.8 % SRCC** on a
SPAQ→KADID cross-dataset transfer (Q-Align Table 11). This is the mechanism that
dents R2's generalization frontier.

## The Q-family spine (Q-Future · NTU / SJTU, Wu et al.)

Lead author **Haoning Wu**; senior authors **Weisi Lin**, **Guangtao Zhai**. The
arc runs benchmark → describe → rate.

1. **Q-Bench** (ICLR 2024 Spotlight, arXiv:2309.14181) — the *diagnosis*. Three
   axes: **perception** (LLVisionQA, 2,990 images), **description** (LLDescribe,
   499 images), **assessment**. Finding: general MLLMs (GPT-4V, Gemini) have
   "preliminary low-level visual skills" — they *perceive/describe* quality above
   chance — but are "unstable and imprecise" and **cannot produce precise
   quantitative scores.** That gap is the era's motivation.
2. **Q-Instruct** (CVPR 2024, arXiv:2311.06783) — teach the model to *describe*
   quality. **Q-Pathway** = 58,000 human low-level *descriptions* on 18,973
   images; **Q-Instruct** = 200,000 instruction-response pairs synthesized from
   it. Lifts perception/description, but not yet calibrated scoring.
3. **Q-Align / OneAlign** (ICML 2024, arXiv:2312.17090) — the *convergence
   result*, built on **mPLUG-Owl2**. The levels-as-tokens trick above.
   **OneAlign**: one model jointly trained on **IQA + IAA + video VQA** that beats
   task-specific SOTA on all three, incl. cross-dataset.
4. **Co-Instruct** (ECCV 2024 Oral, arXiv:2402.16641) — open-ended **multi-image
   quality comparison** (Co-Instruct-562K data, MICBench benchmark). R1's
   pairwise-comparison protocol reborn as an LMM capability.

**DepictQA** (You, Xue, Dong et al., CUHK / XPixel — a *different* group; ECCV
2024, arXiv:2312.08962) — *describe-and-compare* quality in language as a
hierarchical descriptive judgment; follow-up DepictQA-Wild. The score is a lossy
projection of a richer linguistic verdict.

## The aesthetic arm — same recipe

Aesthetics lands on the *identical* instruction-tuning recipe as quality (R1's
"same machine" holding on the aesthetic side):

- **Q-Align's aesthetic arm** — the same discrete-level method on **AVA** gives
  **0.822 / 0.817** SRCC/PLCC (top of R3's AVA ladder) with no aesthetic-specific
  architecture.
- **UNIAA** (Kuaishou/Kling + PKU, arXiv:2404.09619) — a *unified* aesthetic
  baseline (**UNIAA-LLaVA**) + benchmark (**UNIAA-Bench**, three levels echoing
  Q-Bench: perception/description/assessment).
- **AesExpert** (Huang, Li, Lin, Shi et al., **Xidian** — *not* the TANet/TAD66K
  group; ACM MM 2024, arXiv:2404.09624) — aesthetics instruction tuning at scale:
  **AesMMIT** (409K instructions, 21,904 images, 88K feedbacks) → the AesExpert
  model. Q-Instruct's move pointed at composition, colour harmony, mood.

Recipe, both tracks: **benchmark native ability (Q-Bench / UNIAA-Bench) →
instruction-tune to describe (Q-Instruct / AesExpert) → align to rate (Q-Align,
both arms).**

## Benchmark: one model at the top of both columns

In-domain SRCC / PLCC ([IQA / IAA Evaluation Metrics](/wiki/iqa-iaa-evaluation-metrics/)). The bottom block is the
convergence: one family tops the IQA column R2 built *and* the AVA column R3 built.

| Method | Track | KonIQ | AVA |
|---|---|---|---|
| LIQE (CLIP, R2) | IQA | 0.919 / 0.912 | 0.776 |
| VILA-R (VLP, R3) | IAA | — | 0.774 / 0.774 |
| **Q-Align** (MLLM) | both | **0.940 / 0.941** | **0.822 / 0.817** |
| **OneAlign** (unified) | both + video | **0.941 / 0.950** | **0.823 / 0.819** |
| DeQA-Score (2025) | IQA | **0.941 / 0.953** | — |

OneAlign also scores video (LSVQ 0.886 / 0.886). Cross-dataset (train KonIQ):
Q-Align → CLIVE **0.860**, SPAQ **0.887** vs HyperIQA→CLIVE 0.785, CLIP-IQA+→CLIVE
0.805 — the smallest generalization collapse in the series.
*Not attributed to Q-Align: FLIVE (not in its tables); MANIQA (not a Q-Align row —
its named IQA baseline is CLIP-IQA+).*

## Open problems (the honest close)

- **Calibration / reproducibility.** A softmax over five tokens is prompt-,
  temperature-, and checkpoint-sensitive. **DeQA-Score** (CVPR 2025,
  arXiv:2501.11561) models the whole score *distribution* (Thurstone fidelity
  loss) to address it.
- **Hallucination** in the language explanation — a fluent, confident, wrong
  reason invites misplaced trust.
- **Benchmark saturation + contamination.** KonIQ ≈ 0.94 / AVA ≈ 0.82 are near the
  label noise ceiling; web-scale pretraining risks train/test leakage, so a high
  SRCC may measure memorization.
- **Compute.** OneAlign is billions of parameters vs MANIQA's ≈ 20 MB / NIQE's
  millisecond CPU cost — a 1000× gap for ≈ 0.02–0.05 SRCC.
- **Is SRCC-on-one-dataset the right target anymore?** If the model can describe,
  compare, and reason (Q-Bench, Co-Instruct, DepictQA), one correlation
  coefficient throws away most of what it does — the frontier may be *faithful
  description/comparison*, not a number.

**Third axis folding in.** R1 kept generative-image quality ("how real?") and
video quality (VQA) separate; both are being absorbed: OneAlign scores video;
**Q-Bench-Video** (CVPR 2025, arXiv:2409.20063) benchmarks LMM video understanding;
**Q-Eval-100K / Q-Eval-Score** (CVPR 2025 Oral, arXiv:2503.02357) scores
*generated* content quality + prompt alignment. R1's three axes are converging on
one linguistic scorer.

> **Q-Insight** (arXiv:2503.22679, 2025) carries the "Q-" name but is a *different
> group* (Jian Zhang et al., not Q-Future) — an RL/reasoning-based quality MLLM.
> Do not attribute it to Haoning Wu's cluster.

## Practitioner decision guide

- **≥ 0.9 SRCC + interpretability, compute available:** Q-Align / OneAlign (or
  DeQA-Score for calibration).
- **Tiny / fast / on-device:** MANIQA or HyperIQA (≈ 0.90, few MB); NIQE for
  zero-training millisecond CPU scoring (accept ≈ 0.66 authentic).
- **Perceptual training loss (SR / diffusion / codecs):** LPIPS or DISTS (R2's FR
  branch) — MLLM scorers are not differentiable losses.
- **Compare images / need a reason, not a number:** Co-Instruct or DepictQA.

## See also

- [Image Quality Assessment (IQA)](/wiki/image-quality-assessment/) — the IQA track's rung 5 points here.
- [Image Aesthetic Assessment (IAA)](/wiki/image-aesthetic-assessment/) — the IAA track's rung 5 points here.
- [IQA / IAA Datasets](/wiki/iqa-iaa-datasets/) — incl. the MLLM-era instruction datasets (Q-Pathway,
  Q-Instruct, Co-Instruct-562K, AesMMIT, Q-Eval-100K).
- [IQA / IAA Evaluation Metrics](/wiki/iqa-iaa-evaluation-metrics/) — SRCC/PLCC, EMD, and the levels-as-tokens score.
