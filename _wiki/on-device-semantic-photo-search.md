---
share: true
title: On-device semantic photo search
categories:
  - Wiki
tags:
  - clip
  - on-device-ai
  - computer-vision
  - gallery
author: claude
type: concept
created: 2026-07-19
updated: 2026-07-19
sources:
  - "[[2026-07-19-on-device-ml-in-apple-and-samsung-camera-and-gallery]]"
aliases:
  - CLIP embeddings
  - natural-language photo search
  - image-text embedding search
  - ANN photo search
---

# On-device semantic photo search

How "find photos of a dog on a beach" works on a phone, and what Apple and
Samsung actually ship for it. The mechanism is vendor-neutral; the products are
partly documented (Apple) and partly inferred (Samsung). Runs on the
[NPU](/wiki/on-device-neural-accelerators/) via an [on-device
runtime](/wiki/on-device-ml-runtimes/); it is one head of the broader [photo-ML
feature set](/wiki/mobile-photo-ml-features/).

## The mechanism: joint embedding + ANN

Natural-language image retrieval maps every photo **and** the text query into one
shared vector space with a **CLIP-family dual encoder** (contrastive image+text
training; the reference CLIP ViT-B/32 emits a 512-dim embedding per modality),
then does **approximate nearest-neighbor (ANN)** search of the query vector
against the stored photo vectors. The decomposition:

1. **Encode-once-at-import** — run the image encoder per photo → one vector,
   persisted.
2. **ANN-at-query** — encode the text query once (cheap), search the vectors,
   return top-k.

Because the text tower is tiny, query cost is dominated by the ANN step and
import cost by the image tower — so the whole design hinges on a cheap image
encoder.

**The efficient encoder: MobileCLIP** (Apple, CVPR 2024) is exactly this — S0–S2
variants report single-digit-millisecond image/text encoders (S2 ≈3.6 ms image,
74.4% zero-shot ImageNet), trained by distilling a CLIP ensemble + a captioning
model into a "reinforced dataset". This is the [efficient-architecture](/wiki/efficient-small-model-training/) story applied to the image-text encoder.

## The search side: flat scan often suffices

At photo-library scale (**tens of thousands** of vectors; ~100 MB float32 for
50k×512) a **flat brute-force scan is often fast enough** and ANN is unnecessary.
When it is needed, the FAISS-style menu trades recall × latency × memory:

- **HNSW** (graph) — lowest query latency, highest RAM, slow/costly build+update.
- **IVF** (partitioned k-means) — ~3× less RAM than HNSW, cheap to rebuild
  (seconds) → good for a library that constantly gains/loses photos.
- **Product Quantization (PQ)** — compresses the vector store from ~100 MB to a
  few MB at some recall cost; the lever that makes a large index fit on a phone.

> ⚠️ Server-class ANN benchmarks are **not** phone numbers — they establish the
> *shape* of the tradeoff, not on-device latency.

## What the vendors ship

**Apple — documented.** The **Apple Neural Scene Analyzer (ANSA)**, a **CLIP-style
contrastive image–text model** (MobileNetV3-variant image encoder ~16M params;
text encoder pruned 44M→26M; all heads under ~9.7 ms, ~24.6 MB resident on the
ANE; 8-bit quantized) trained on "a few hundred million image-text pairs", whose
frozen embeddings power visual search in Photos/Spotlight, Memories,
deduplication, scene classification in Camera, and wallpaper suggestions.
Text→image search is embedding retrieval over these on-device vectors. Apple also
documents an on-device **knowledge graph** (people/places/events) powering
Memories.

> ⚠️ Apple does **not** document the ANN/index data structure — that it is
> HNSW/IVF/etc. is inferred, not stated. The ANSA per-head latency/memory figures
> and embedding dimensions should be re-verified against the source before being
> quoted as hard numbers.

**Samsung — partly documented.** True natural-language "Gallery Search" shipped
with the Galaxy S25 (One UI 7, January 2025), described as powered by a
**"vision-language model that learns by associating images with text"** plus
generative expansion of candidate query sentences; an independent breakdown lists
it as **on-device**. But Samsung names no architecture:

> ⚠️ Calling Samsung's engine "CLIP-like" is **inference**, not a Samsung claim.
> The "generate candidate sentences" detail hints at query-expansion/tag-matching
> rather than pure embedding cosine similarity; the actual retrieval mechanism is
> unknown. (The separate Gemini↔Gallery integration in One UI 8.5, 2026, is a
> cloud/Google path, distinct from Samsung's on-device engine.)

## Relation to quality/aesthetic scoring

Semantic search embeddings are a *content* index, orthogonal to the
quality/aesthetic scoring covered in [Image Quality Assessment (IQA)](/wiki/image-quality-assessment/) /
[Image Aesthetic Assessment (IAA)](/wiki/image-aesthetic-assessment/) — but the same CLIP-family backbone underlies both
the vision-language IQA/IAA methods (see [Multimodal-LLM Visual Scoring](/wiki/multimodal-llm-visual-scoring/)) and
this retrieval task, which is why one on-device encoder can feed several features.

## See also

- [Mobile photo ML features (Apple vs Samsung)](/wiki/mobile-photo-ml-features/) — the full Camera/Gallery ML feature set.
- [On-device ML runtimes (Core ML vs LiteRT)](/wiki/on-device-ml-runtimes/) — the runtime that hosts the encoder + ANN.
- [Efficient small-model training](/wiki/efficient-small-model-training/) — why MobileCLIP-class encoders fit on device.
