---
share: true
title: IQA / IAA Datasets
categories:
  - Wiki
author: claude
type: concept
created: 2026-07-18
updated: 2026-07-18
sources:
  - "[[2026-07-18-image-quality-and-aesthetic-assessment-foundations]]"
  - "[[2026-07-18-image-aesthetic-assessment-methods-nima-to-vila]]"
  - "[[2026-07-18-image-quality-and-aesthetic-assessment-convergence-mllm-scoring]]"
aliases:
  - IQA datasets
  - IAA datasets
  - aesthetics datasets
  - LIVE
  - TID2013
  - KonIQ-10k
  - AVA dataset
---

# IQA / IAA Datasets

The fixed coordinate system the whole field reports against. Methods come and
go ([Image Quality Assessment (IQA)](/wiki/image-quality-assessment/), [Image Aesthetic Assessment (IAA)](/wiki/image-aesthetic-assessment/)); these
datasets are what every SRCC/PLCC number ([IQA / IAA Evaluation Metrics](/wiki/iqa-iaa-evaluation-metrics/)) is
measured on.

## The spine

| Dataset | Year | Task | Images | Distortion | Label | Subjective study |
|---|---|---|---|---|---|---|
| **LIVE IQA** (R2) | 2006 | IQA | 779 (+29 ref) | synthetic, 5 types | DMOS 0–100 | ~23 subjects, in-lab, single-stimulus w/ hidden ref |
| **CSIQ** | 2010 | IQA | 866 (+30 ref) | synthetic, 6 types | DMOS 0–1 | ~35 subjects, ~5,000 ratings, in-lab |
| **TID2013** | 2013 | IQA | 3,000 (25×24×5) | synthetic, 24 types × 5 | MOS 0–9 | 971 observers, ~524k pairwise comparisons |
| **KADID-10k** | 2019 | IQA | 10,125 (81×25×5) | synthetic, 25 types × 5 | MOS | 30 crowd ratings/image |
| **CLIVE** (LIVE-in-the-Wild) | 2016 | IQA | 1,162 | authentic | MOS | >350k scores, >8,100 MTurk workers |
| **KonIQ-10k** | 2020 | IQA | 10,073 | authentic | MOS | 1.2M ratings, 1,459 crowd workers |
| **SPAQ** | 2020 | IQA | 11,125 | authentic (66 phones) | MOS + 5 attributes + EXIF | in-lab, controlled |
| **PaQ-2-PiQ / FLIVE** | 2020 | IQA | ~39,810 + 120k patches | authentic | MOS (picture + patch) | ~4M human judgments |
| **AVA** | 2012 | IAA | **255,530** | none | score distribution (1–10) + 66 tags + 14 styles | DPChallenge, avg ~210 votes/image |
| **AADB** | 2016 | IAA | 10,000 | none | score + 11 attributes | 5 MTurk raters/image, identity tracked |
| **PARA** | 2022 | IAA | 31,220 | none | score + 9 objective + 4 subjective attributes + rater traits | 438 subjects, ~25 annotations/image |
| **TAD66K** | 2022 | IAA | 66,327 | none | theme-aware scores, 47 themes | >1,200 annotations/image |
| **EVA** | 2020 | IAA | 4,070 | none | MOS + 4 attributes + difficulty + weights | ≥30 votes/image |
| **FLICKR-AES** | 2017 | IAA (**PIAA**) | ~40,000 | none | per-rater scores (~200 imgs/worker) | 210 workers, rater identity tracked |

*Ref = pristine reference-image count. Study-scale figures are primary-source
numbers where stated. **Unverified against primary source** (protocol
confirmed, exact count not stated in abstracts): SPAQ rating count, PARA total
annotations, EVA total votes. LIVE's single- vs double-stimulus wording differs
across secondary sources; primary description is single-stimulus with hidden
reference. KADID-10k has an unlabelled companion **KADIS-700k** (700k images)
for weak supervision. **FLICKR-AES** (Ren et al., ICCV 2017) is the first
personalized-IAA ([Image Aesthetic Assessment (IAA)](/wiki/image-aesthetic-assessment/)) benchmark — it tracks rater
identity so a model can be evaluated per person; its ~200-images/worker figure
is approximate. PARA (2022) is the modern PIAA benchmark.*

## The three transitions this table encodes

1. **Small synthetic → large synthetic** (LIVE/TID → KADID-10k). LIVE's 779
   and TID's 3,000 images are too small for deep nets; KADID-10k + KADIS-700k
   feed data-hungry CNNs while keeping the clean synthetic design.
2. **Synthetic → authentic** (→ KonIQ, CLIVE, SPAQ, FLIVE). The transition that
   mattered most: synthetic-trained models did not survive real photos. KonIQ
   (1.2M ratings) and FLIVE (~4M judgments, patch-level) are the modern NR-IQA
   proving grounds.
3. **Scalar MOS → score distribution** (AVA). AVA's per-image vote *histogram*
   is what enabled NIMA's EMD distribution prediction; AADB/PARA/EVA add named
   attributes (explainable aesthetics); TAD66K adds theme awareness.

> **Why this matters for reading method papers.** A 2013 method reporting only
> LIVE/TID SRCC and a 2023 method reporting KonIQ/FLIVE SRCC are not comparable
> — they solved different problems. NR-IQA methods after ~2019 benchmark on
> KonIQ and FLIVE; IAA methods benchmark on AVA.

## The MLLM era — instruction & benchmark datasets

A **fourth** kind of dataset arrived with [Multimodal-LLM Visual Scoring](/wiki/multimodal-llm-visual-scoring/)
(2023–2026): not MOS-regression sets but **instruction-tuning corpora** (human
*descriptions* of quality/aesthetics, and synthesized question-answer pairs) and
**capability benchmarks** (perception / description / comparison, scored beyond a
single correlation). The MOS spine above is still what SRCC/PLCC is measured on;
these teach and probe the *language* side.

| Dataset | Year | For | Content |
|---|---|---|---|
| **Q-Pathway** | 2023 | IQA (instruction) | 58,000 human low-level *descriptions* on 18,973 images (→ Q-Instruct) |
| **Q-Instruct** | 2023 | IQA (instruction) | 200,000 instruction-response pairs synthesized from Q-Pathway |
| **Co-Instruct-562K** | 2024 | IQA (instruction) | multi-image comparative quality instructions |
| **AesMMIT** | 2024 | IAA (instruction) | 409K aesthetic instructions over 21,904 images + 88K feedbacks (→ AesExpert) |
| **UNIAA-LLaVA** data | 2024 | IAA (instruction) | aesthetic visual-instruction data unified from existing IAA sets |
| **Q-Eval-100K** | 2025 | generative | 100K text-to-image/video instances, 960K human MOS (quality + prompt alignment) |
| **Q-Bench** (LLVisionQA / LLDescribe) | 2024 | IQA (benchmark) | 2,990-image perception MCQs; 499-image expert descriptions |
| **MICBench** | 2024 | IQA (benchmark) | first multi-image quality *comparison* benchmark for LMMs |
| **UNIAA-Bench** | 2024 | IAA (benchmark) | aesthetic perception / description / assessment |
| **Q-Bench-Video** | 2024 | VQA (benchmark) | 2,378 QA pairs on video quality understanding |

*These do not carry a single per-image MOS to regress; they carry descriptions,
comparisons, or QA. The convergence-era point (R4): the field is adding datasets
that measure *describe / compare / reason about quality*, not just predict a
number — and generative-image and video quality (Q-Eval-100K, Q-Bench-Video) are
folding into the same MLLM-scorer pipeline.*

## See also

- [Image Quality Assessment (IQA)](/wiki/image-quality-assessment/) · [Image Aesthetic Assessment (IAA)](/wiki/image-aesthetic-assessment/) ·
  [IQA / IAA Evaluation Metrics](/wiki/iqa-iaa-evaluation-metrics/) · [Multimodal-LLM Visual Scoring](/wiki/multimodal-llm-visual-scoring/)
- Data-loading and decoding context for building such datasets:
  [Android image decoding](/wiki/android-image-decoding/), [ImageNet-scale training logistics](/wiki/imagenet-scale-training-logistics/).
