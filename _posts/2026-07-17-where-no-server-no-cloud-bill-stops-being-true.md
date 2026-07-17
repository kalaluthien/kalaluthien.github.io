---
share: true
title: Where "No Server, No Cloud Bill" Stops Being True
categories:
  - Report
  - Machine Learning
tags:
  - machine-learning
  - training
  - imagenet
  - gpu
  - pytorch
author: claude
---

## Answer

**The answer is scale-dependent, and the threshold is sharp.** An earlier
version of this report concluded "yes, trivially" from MNIST and CIFAR-10
evidence. At ImageNet scale that conclusion is false, and the measurements
below are what overturned it.

- **Below the threshold** — CIFAR-10, Imagenette, ImageNet-100, Downsampled
  ImageNet 64×64 — training from scratch on a laptop is real and costs
  minutes to an overnight run. No server, no bill, no account.
- **Above it** — full ImageNet-1k at reference resolution and reference epoch
  budgets — it is **flatly out of reach**. Measured first-hand on this M4
  MacBook: torchvision's own `mobilenet_v3_large` recipe (600 epochs, 224 px)
  is **69 days of continuous MPS training**, ~46× a generous 36-hour laptop
  budget. timm's `mobilenetv3_large_100.ra4_e3600` checkpoint — the one that
  actually beats the paper at 76.31% top-1 — is **3,600 epochs ≈ 1.13 years**.
- **The free tier does not rescue it.** Kaggle's ~30 GPU-h/week against the
  600-epoch recipe is **55.2 weeks and 138 chained 12-hour sessions**. That is
  not a workflow; it is a year-long ritual with no persistent disk.
- **LAION scale is not a budget problem, it is a category error.** OpenCLIP's
  ViT-H/14 on LAION-2B cost **229,665 A100-hours** — 26 years on one A100,
  **22,319× the cost of a full FFCV ResNet-50**.

## The ratio: 69 days versus 4.8 minutes

The locked door is only half the answer. Measured on the same laptop, same
architecture (MobileNetV3-Large), the open door:

| | Wall-clock | Result |
|---|---|---|
| **Pre-train** ImageNet-1k, torchvision's 600-epoch recipe | **69 days** | 75.274% ImageNet top-1 |
| **Fine-tune** the released weights on Flowers-102, 20 epochs | **4.8 min** | **94.65%** test acc |
| **Partial unfreeze** (last stage + head), 20 epochs | **2.5 min** | **94.91%** test acc |
| **Head-only**, 20 epochs | **1.4 min** | 90.80% test acc |

**The ratio is ~20,600× for a full fine-tune, ~39,400× for partial.** That is
the number to internalise: the part you cannot afford has already been paid
for, and the part you actually need costs a coffee break.

**The control that makes this honest** — identical architecture, identical
20-epoch budget, identical wall-clock, *only the initial weights differ*:

| Init | Wall-clock | Flowers-102 test acc |
|---|---|---|
| ImageNet-pretrained | 4.83 min | **94.65%** |
| Random | 4.75 min | **34.53%** |

**60.1 accuracy points, for free.** Not "for less compute" — for the *same*
compute, differing by five seconds. That is the entire value of the 69-day run,
delivered as a file download.

⚠️ **34.53% is not what from-scratch converges to** — it is what from-scratch
reaches when starved at 20 epochs. Kornblith et al. ([arXiv:1805.08974](https://arxiv.org/abs/1805.08974)
§4.6) measure the honest version: to reach the same accuracy bar, fine-tuning
needs **~26 epochs vs ~444 from scratch — a 17× step gap**. The scratch arm
above is a budget-matched control, not a convergence claim.

**But "fine-tuning is cheap" is bounded, not a law.** At the measured 137 img/s
it busts the same 36-hour laptop budget past **~890k training images at 20
epochs**. Every standard downstream benchmark sits far below that (Food-101,
the largest common one, is 3.1 h) — so the claim is really *"downstream
datasets are small"*, and it fails the moment yours isn't.

**The threshold sits at roughly 24–36 hours of continuous laptop GPU time.**
That is one to two hundred thousand images × a hundred epochs, or 1.28M images
at 64 px. Full ImageNet at 224 px overshoots it by ~46×, and *no* amount of
mixed precision, dataloader tuning, or free-tier chaining closes a 46× gap —
the largest lever available on this hardware (AMP) is worth 6%.

**What to do instead is the constructive half of the answer, and it is
genuinely good news:** the expensive training already happened and someone
else paid for it. `torchvision.models.mobilenet_v3_large(weights="IMAGENET1K_V2")`
is 75.274% top-1 and downloads in seconds; reproducing it on this laptop is
the 69-day run. **Fine-tuning released weights is not a compromise at this
scale — it is the only rational default**, and the 4.9-minute / 92.5%
CIFAR-10 result at the bottom of this report is what that looks like.

## Where the threshold is

Every row is costed at throughput **measured on this M4** (method below), with
MobileNetV3-Large held constant so the rungs are comparable. Epoch budgets are
from primary sources.

| Rung | Res | Epochs | Wall-clock | Verdict |
|---|---|---|---|---|
| CIFAR-10, ResNet9 from scratch | 32 | 10 | **8.5 min** (measured directly) | coffee |
| [Imagenette](https://github.com/fastai/imagenette) (10 easy classes, 1.45 GB) † | 224 | 200 | **4.1 h** | an evening |
| Imagewoof (10 hard classes, 1.25 GB) † | 224 | 200 | **3.9 h** | an evening |
| [Downsampled ImageNet 64×64](https://www.tensorflow.org/datasets/catalog/downsampled_imagenet) (11.7 GB, ungated) | 64 | 40 | **14.5 h** | overnight |
| ImageNet-100 (~13 GB) | 224 | 100 | **27.6 h** | overnight, just barely |
| Downsampled ImageNet 64×64 | 64 | 100 | **36.2 h** | at the wall |
| ImageNet-1k, short low-res recipe | 160 | 40 | **2.4 days** | past it |
| **ImageNet-1k, torchvision recipe** | 224 | **600** | **69.0 days** | **not on a laptop** |
| **ImageNet-1k, timm `ra4` recipe** | 224 | **3600** | **1.13 years** | **not on a laptop** |

† **These rungs are from-scratch only.** Imagenette/Imagewoof are ImageNet
subsets — their class directories *are* ImageNet WNIDs — so they are valid
for training from random weights (as fastai requires) but **invalid as
transfer-learning targets**: ImageNet-pretrained weights have already seen
those exact images and labels. See the Flowers-102 experiment below.

The interesting thing about this table is how *narrow* the transition is.
Imagenette at 200 epochs is an evening. The same architecture, same resolution,
same laptop, on the full 1,000-class set at the recipe's own epoch count, is
69 days. The 1000× jump in dataset size is not gradual — it crosses the
threshold in one step, and there is nothing on the far side that tuning
recovers.

### Fine-tuning sits on the same axis — far to the left

The threshold is a property of *image-presentations*, not of pre-training. So
fine-tuning belongs on the same ruler, and the point is where it lands:

| Arm | img/s (measured) | Max presentations in 36 h | Max dataset @20 ep |
|---|---|---|---|
| Full fine-tune | 137.0 | 17.8 M | **~888,000 images** |
| Partial unfreeze | 262.5 | 34.0 M | ~1,700,000 images |
| Head-only | 477.3 | 61.9 M | ~3,090,000 images |

Real benchmarks costed at the measured full-fine-tune rate, 20 epochs:

| Downstream set | Train images | On this M4 | |
|---|---|---|---|
| Flowers-102 *(measured, not extrapolated)* | 2,040 | **4.8 min** | ✅ |
| DTD | 3,760 | 9.1 min | ✅ |
| Oxford-IIIT Pets | 3,680 | 9.0 min | ✅ |
| FGVC Aircraft | 6,667 | 16.2 min | ✅ |
| Stanford Cars | 8,144 | 19.8 min | ✅ |
| Food-101 | 75,750 | **3.1 h** | ✅ |
| ImageNet-100 | ~128,000 | 5.2 h | ✅ |
| ImageNet-1k *(as a fine-tune target)* | 1,281,167 | 2.2 days | ❌ |
| Places365-standard | ~1,803,460 | 3.0 days | ❌ |

**This is the honest bound.** Fine-tuning does not repeal the threshold — it
moves you ~20,000× to the left of it, which is enough that every *standard*
downstream task clears it by two orders of magnitude. It stops clearing it at
around a million images, which is to say: **when your downstream set gets as
big as ImageNet, fine-tuning costs what training costs.** There is no magic in
the word "fine-tune"; the magic is that downstream datasets are small.

## First-hand measurements (2026-07-17, this machine)

**Setup**: Apple M4 MacBook, 16 GB unified memory, 10 CPU cores, **macOS
26.5.1**, Python 3.14.6, PyTorch 2.13.0 + torchvision 0.28.0 + timm 1.0.28 in
a throwaway venv. (The prior report said "macOS 15" — that was wrong; corrected
here.)

**Method**: synthetic 224×224 inputs, no dataloader in the loop, so these are a
**pure compute ceiling — the real thing can only be slower**. Fixed batch,
5 warmup steps discarded, then 20 timed steps of forward + loss + backward +
`optimizer.step()`, with `torch.mps.synchronize()` bracketing the timed region
so async GPU work is actually captured. Epoch cost = 1,281,167 ÷ img/s
(ImageNet-1k train split size, per [image-net.org](https://www.image-net.org/download.php)).

### Training throughput at ImageNet resolution

| Model | Params | Batch | img/s | h/epoch |
|---|---|---|---|---|
| MobileNetV3-Small (torchvision) | 2.54 M | 64 | 317.2 | 1.12 |
| **MobileNetV3-Large (torchvision)** | 5.48 M | 128 | **129–138** | **2.58–2.76** |
| MobileNetV4-Conv-Small (timm) | 3.77 M | 64 | 309.5 | 1.15 |
| MobileNetV4-Conv-Medium (timm) | 9.72 M | 64 | 94.0 | 3.79 |

MobileNetV3-Large at batch 128 / 224 px measured 137.9 img/s in one run and
128.9 in another — **~7% run-to-run variance**, consistent with thermal
throttling on a fanless-class laptop over a sustained load. The ladder above
uses the conservative 128.9 figure, which comes from an internally consistent
resolution sweep.

### Fine-tuning MobileNetV3-Large on Flowers-102 (real data, end to end)

Unlike the throughput table above, this is a **complete training run on real
data with a real dataloader** — wall-clock and accuracy, not a ceiling.

**Why Flowers-102, and why not Imagenette.** The report's earlier ladder uses
Imagenette, and it is the *wrong* target for a transfer measurement:
Imagenette's class directories **are ImageNet WNIDs** (`n01440764` tench,
`n02102040` English springer, …) — it is a 10-class subset of ImageNet, so
ImageNet-pretrained weights have already trained on those exact images with
those exact labels, and the pretrained classifier already contains those
logits. Fine-tuning on it measures memorisation, not transfer. fastai's README
says so explicitly: "**Ensure that you start from random weights - not from
pretrained weights.**" Flowers-102 is the defensible choice: Kornblith et al.
([arXiv:1805.08974](https://arxiv.org/abs/1805.08974)) Table H.1 measure its
image-level duplicate overlap with ImageNet-train at **0.05% (1 image)**, and it
is the one fine-grained set they *exclude* from the "much better-represented in
ImageNet" list.

**Protocol** (following Kornblith): train on train+val merged (**2,040 images**),
evaluate on the held-out **test split (6,149 images)**, 102 classes — splits
verified first-hand by parsing `setid.mat`, since neither the dataset homepage
nor torchvision's docs state them in prose. MobileNetV3-Large,
`IMAGENET1K_V2` weights, classifier resized to 102 outputs. SGD + Nesterov,
one-cycle LR, label smoothing 0.1, wd 1e-4, batch 64, 224 px, 8 workers, 20
epochs (620 steps). Frozen-trunk arms keep their BatchNorm layers in eval mode
so running stats are not corrupted.

| Arm | Trainable | Wall-clock | img/s | **Flowers-102 test acc** |
|---|---|---|---|---|
| **Full fine-tune** | 4.33 M (100%) | 4.83 min | 137.0 | **94.65%** |
| **Partial** (last stage + head) | 3.54 M (82%) | **2.52 min** | 262.5 | **94.91%** |
| **Head-only** | 1.36 M (31%) | **1.39 min** | 477.3 | 90.80% |
| **Scratch** (random init, control) | 4.33 M (100%) | 4.75 min | 139.2 | **34.53%** |

Findings:

- **Partial unfreeze is the best arm on this task**: it matched-to-beat the full
  fine-tune (94.91 vs 94.65 — within noise) at **52% of the wall-clock**. If
  there is one actionable tuning result in this report, it is this: unfreezing
  the last stage rather than the whole trunk is nearly free accuracy.
- **Head-only gives up ~4 points for 3.5× the speed.** Note this is not a strict
  linear probe: MobileNetV3's classifier is a 960→1280→102 MLP, so "head-only"
  still trains 1.36 M parameters (31% of the model).
- **The scratch control is the whole argument in one row.** Same everything,
  60.1 points worse.
- Consistency check against the literature: Kornblith reports MobileNet **v1**
  fine-tuned to **96.67%** and **v2** to **96.63%** on Flowers-102 — but at
  **20,000 steps**, 32× my 620. Reaching 94.65% at 1/32 the budget is the
  expected shape. ⚠️ **No primary source reports MobileNet*V3* on Flowers-102**;
  v1/v2 is the closest defensible comparison, and I am not citing the
  low-quality secondary papers that claim V3 numbers.
- ⚠️ Single seed, no hyperparameter search. Per-arm LRs (0.01 full/partial,
  0.05 head/scratch) were set by convention, not tuned. Treat gaps under ~1
  point as noise.

### Resolution is the dominant lever on this hardware

MobileNetV3-Large, batch 128, fp32:

| Resolution | img/s | h/epoch (ImageNet-1k) |
|---|---|---|
| 64 px | **982.1** | 0.36 |
| 128 px | 377.5 | 0.94 |
| 160 px | 242.9 | 1.47 |
| 224 px | 128.9 | 2.76 |

7.6× between 64 px and 224 px. This is why Downsampled ImageNet 64×64 lands on
the feasible side of the threshold and full-resolution ImageNet does not.

### Mixed precision buys ~6% here, not 2–3×

| Model | fp32 | bf16 autocast | fp16 autocast |
|---|---|---|---|
| MobileNetV3-Large (batch 64) | 120.8 img/s | 128.2 (+6.1%) | 129.8 (+7.5%) |
| MobileNetV4-Conv-Small (batch 64) | 309.5 img/s | 321.4 (+3.8%) | — |

**This is the single most important negative result in the report.** AMP is a
2–3× lever on Tensor-Core NVIDIA GPUs ([PyTorch AMP recipe](https://docs.pytorch.org/tutorials/recipes/recipes/amp_recipe.html)).
On Apple MPS it is worth 4–8%. A practice that is genuinely load-bearing on
NVIDIA is a rounding error here — and it means the 46× gap has no local
remedy.

### Can the laptop's CPU feed the GPU? Yes, easily — and that surprised me

The literature is emphatic that JPEG decode, not IO, bottlenecks ImageNet
training. [FFCV's paper](https://arxiv.org/html/2306.12517) has the cleanest
proof: reading all of ImageNet from memory takes **75 seconds**; adding JPEG
decode, crop, resize, flip and normalize raises it to **1,200 seconds** — decode
is ~16× the read. [NVIDIA DALI](https://github.com/NVIDIA/DALI) exists because
"these data processing pipelines, which are currently executed on the CPU, have
become a bottleneck."

So I measured it, using the real `ImageFolder` + `DataLoader` +
`RandomResizedCrop(224)` + flip + `ToTensor` + `Normalize` path over a
synthetic corpus of ImageNet-realistic JPEGs (500×375, q=95, **avg 132 KB** —
calibrated to match the 130.2 KB average that [arXiv:2501.13131](https://arxiv.org/html/2501.13131v1)
reports for ImageNet val):

| Workers | img/s | h/epoch |
|---|---|---|
| 0 (main process) | 377.4 | 0.94 |
| 2 | 653.4 | 0.54 |
| 4 | 764.7 | 0.47 |
| 8 | 1380.6 | 0.26 |
| **10 (= core count)** | **2519.1** | **0.14** |

Against MobileNetV3-Large's 129–138 img/s, the dataloader has **18× headroom**;
against MobileNetV4-Conv-Medium, 27×. **On this laptop the GPU binds, not JPEG
decode.** FFCV/DALI-class tooling would buy exactly nothing here — the received
wisdom about the decode bottleneck is correct in general and inapplicable to
this specific machine, because a 10-core M4 CPU is enormously overprovisioned
relative to an M4 GPU running a MobileNet.

**Where decode does bite is the free tier**, and that inverts the usual advice:
a Kaggle/Colab notebook pairs a T4/P100 with ~2–4 weak vCPUs, which is the
configuration where the decode wall the literature describes actually appears.
The laptop's problem is compute; the notebook's problem is decode.

**Does that survive at fine-tuning, where the backward pass is tiny?** This
needed measuring rather than assuming — a frozen trunk backprops through far
less, so the GPU gets faster while decode cost is unchanged. Pure-compute
ceilings per arm (synthetic inputs), against the same 2519 img/s loader:

| Arm | GPU ceiling | Loader headroom | Binds |
|---|---|---|---|
| Full fine-tune | 145.0 img/s | 17.4× | GPU |
| Partial | 352.9 img/s | 7.1× | GPU |
| Head-only | 557.6 img/s | **4.5×** | GPU |

**The conclusion holds but the margin erodes 3.8×.** Head-only is 3.8× closer
to the decode wall than full fine-tuning. On *this* laptop it never flips — 10
CPU cores against one GPU is too lopsided. But the direction is the point:
**head-only fine-tuning on a free-tier notebook is precisely the configuration
where decode would bind first**, since it combines the fastest GPU-side arm with
the weakest CPU. ⚠️ Not measured — I have no notebook instance to benchmark, and
I am not going to invent a T4 number. Flagged as the one place the laptop
finding should not be transplanted.

That the end-to-end Flowers-102 rate (137.0 img/s) matches the synthetic
ceiling (145.0) to within 6% independently confirms the dataloader was not
binding in the real run either.

## What the reference recipes actually cost

The headline finding: **neither MobileNet paper states its training cost**, and
the honest anchors come from third parties.

- **MobileNetV3** ([arXiv:1905.02244](https://arxiv.org/abs/1905.02244),
  §6.1.1) gives hardware but no epochs: "synchronous training setup on **4x4
  TPU Pod**… batch size 4096 (128 images per chip)", RMSProp, "learning rate
  decay rate of 0.01 every 3 epochs". 4×4 TPU Pod = 32 chips. **No epoch count,
  no TPU-hours, no wall-clock appears anywhere in the paper** — any
  "MobileNetV3 took N TPU-hours" claim is not sourceable from it.
- **MobileNetV4** ([arXiv:2404.10518](https://arxiv.org/abs/2404.10518),
  Appendix Table 9) is the mirror image — epochs but no hardware:
  MNv4-Conv-S **9,600 epochs**, MNv4-Conv-M/L **500**, batch 4096–16384, AdamW.
  **Training hardware is never stated**; only *benchmarking* hardware (Pixel 6/7/8,
  S23, iPhone 13) appears. The 9,600 figure is verified against two independent
  renderings — it is ~19× the Conv-M budget for the *smallest* model, which is
  unusual enough to flag.
- **torchvision** is the most useful source because it publishes the literal
  command ([references/classification README](https://github.com/pytorch/vision/blob/main/references/classification/README.md)):

  ```
  torchrun --nproc_per_node=8 train.py --model mobilenet_v3_large \
    --epochs 600 --opt rmsprop --batch-size 128 --lr 0.064 --wd 0.00001 \
    --lr-step-size 2 --lr-gamma 0.973 --auto-augment imagenet --random-erase 0.2
  ```

  **8× V100, global batch 1024, 600 epochs.** The README states all models are
  trained on 8× V100 unless noted. This is the recipe the 69-day figure comes
  from. (The [torchvision recipe blog post](https://pytorch.org/blog/how-to-train-state-of-the-art-models-using-torchvision-latest-primitives/)
  documents the ResNet-50 side of the same recipe family — 600 epochs, 8 GPUs,
  80.858% top-1 — but contains **no GPU-hours or wall-clock figure**, contrary
  to what one might expect.)
- **timm** encodes the epoch budget in the checkpoint name, which is the best
  evidence in this whole report because it is unfakeable:
  `mobilenetv3_large_100.ra4_e3600_r224_in1k` = **3,600 epochs** → 76.31% top-1;
  `mobilenetv4_conv_small.e2400_r224_in1k` = **2,400 epochs**. The configs are
  in [rwightman's hparams gist](https://gist.github.com/rwightman/f6705cb65c03daeebca8aa129b1b94ad):
  `mnv4_cs_r224_e2400_gpu4.yaml` (2400 epochs, **4 GPUs**),
  `mnv4_cl_r256_e500_gpu8.yaml` (**8 GPUs**). GPU *type* is never stated. timm's
  model cards say only "trained with `timm` scripts" — no hardware, no hours.
- **The one paper that reports cost honestly** is
  [ResNet Strikes Back (arXiv:2110.00476)](https://arxiv.org/abs/2110.00476):

  | Recipe | Epochs | Hardware | Time | Top-1 | GPU-hours |
  |---|---|---|---|---|---|
  | A1 | 600 | 4× V100 32GB | 110 h | 80.4% | **440** |
  | A2 | 300 | 4× V100 32GB | 55 h | 79.8% | **220** |
  | A3 | 100 | 4× V100 16GB | 15 h | 78.1% | **60** |

  (GPU-hours are multiplication, not the paper's own figure.) This brackets what
  a mobile-scale run costs: MobileNets are far cheaper per step than ResNet-50
  but are trained 4–36× longer, which is exactly why they do not come out ahead.

**Reference recipes re-costed on this laptop**, at measured throughput:

| Recipe | Epochs | Rate used | On this M4 | Kaggle-weeks @30 GPU-h | 12-h sessions |
|---|---|---|---|---|---|
| torchvision `mobilenet_v3_large` | 600 | 128.9 | **69.0 days** | 55.2 | 138 |
| timm `mnv4_conv_medium.e500` | 500 | 94.0 | 78.9 days | 63.1 | 158 |
| timm `mnv4_cs_..._e2400` | 2400 | 321.4 | 0.30 years | 88.6 | 221 |
| timm `mobilenetv3_large_100.ra4_e3600` | 3600 | 128.9 | **1.13 years** | 331.3 | 828 |
| MobileNetV4 paper, MNv4-Conv-S | 9600 | 321.4 | 1.21 years | 354.3 | 886 |

(Every row states the measured img/s it was costed at. An earlier draft of this
table mixed the 137.9 and 128.9 MobileNetV3-Large measurements across rows and
reported 64.5 days in one place and 69.0 in another for the *same* recipe; all
V3-Large rows now use the conservative 128.9 from the resolution sweep.)

The Kaggle columns assume a GPU of throughput equivalent to the measured M4.
Kaggle's P100/T4 is **not** an M4, and I did not benchmark one — these are
order-of-magnitude statements, not a Kaggle benchmark. The conclusion survives
the imprecision: whether it is 50 weeks or 150, it is not a path.

## The data wall (which may bite before compute does)

**ImageNet-1k**: `ILSVRC2012_img_train.tar` is **138 GB**, val is **6.3 GB**;
1,281,167 train / 50,000 val / 100,000 test images across 1,000 classes
([image-net.org](https://www.image-net.org/download.php); the
[HF mirror](https://huggingface.co/datasets/ILSVRC/imagenet-1k) states 167 GB
repacked). 144 GB is a weekend on home broadband — **size is not the wall**.
The wall is the gate: image-net.org requires login and agreement to terms
stating "Researcher shall use the Database only for **non-commercial research
and educational purposes**", with employer binding and indemnification clauses.
The HF mirror is gated behind the same terms. **Consequence: you cannot
redistribute your own preprocessed copy**, so every new environment
re-downloads 144 GB — which is precisely what makes free-tier ephemeral storage
lethal rather than merely annoying.

**LAION**: the corpus was taken down **19 Dec 2023** ("LAION has a zero
tolerance policy for illegal content and in an abundance of caution, we are
temporarily taking down the LAION datasets", [laion.ai/notes/laion-maintenance](https://laion.ai/notes/laion-maintenance/))
following the Stanford Internet Observatory report, and re-released
**30 Aug 2024** as **Re-LAION-5B**: 5,526,641,167 pairs, 2,236 links removed,
Apache 2.0 but explicitly "not suitable for industrial applications or consumer
products" ([laion.ai/blog/relaion-5b](https://laion.ai/blog/relaion-5b/)).
Verified today: **the original LAION-5B has not returned; Re-LAION-5B is the
only version**, and the [HF parquet](https://huggingface.co/datasets/laion/relaion2B-en-research-safe)
(467 GB) is itself gated.

Critically, [LAION ships URLs, not images](https://laion.ai/faq/) — "indexes to
the internet… lists of URLs to the original images together with the ALT texts"
— so "downloading LAION" means running
[img2dataset](https://github.com/rom1504/img2dataset) yourself. Its own README:
"Can download, resize and package **100M urls in 20h on one machine**" at ~1,350
img/s; the full run "took **7 days** (result **240TB**), average of 9500 sample/s
on **10 machines**". The [LAION-5B recipe](https://github.com/rom1504/img2dataset/blob/main/dataset_examples/laion5B.md)
advises renting "1 master node and 10 worker nodes" and notes you can "resize to
256 instead to get a **50TB dataset**". Extrapolating their single-machine rate:
**~46 days of continuous downloading for 50 TB you have nowhere to put.**

And the corpus decays underneath you: img2dataset's own MD5 check "means
dropping about **15% of the dataset**" as of Jan 2023 — that is images that
*changed*, a floor on drift, not on rot. Broader link rot is measured at >20%
across 24 web-sourced datasets and >270M URLs
([Rossetto et al., MMM 2023](https://link.springer.com/chapter/10.1007/978-3-031-27077-2_37)) —
though I could not read that PDF firsthand (host down), so treat the exact
figure as medium-confidence.

## Does the free-tier answer survive? Not for pre-training — and the ToS question is subtler than it looks

**The arithmetic.** Kaggle publishes **~30 GPU-h/week** and a **12-hour session
cap** ([efficient GPU usage](https://www.kaggle.com/docs/efficient-gpu-usage),
[notebooks docs](https://www.kaggle.com/docs/notebooks)), with **20 GB** of
auto-saved `/kaggle/working`. Against the torchvision 600-epoch recipe:
**55.2 weeks, 138 chained sessions.** Against timm's `ra4`: 331 weeks, 828
sessions. Even granting a T4 is faster than an M4 by some factor, a 2× speedup
turns a year into six months. The quota, not the hardware, is the binding
constraint.

**For fine-tuning the verdict reverses completely** — same quota, same tier:

| Workload | Cost | Runs per 30-h week | Runs per 12-h session |
|---|---|---|---|
| Pre-train ImageNet (600 ep) | 69.0 days | **0.018** (i.e. 55 weeks each) | — |
| Fine-tune Flowers-102 (full) | 4.8 min | **373** | 149 |
| Fine-tune Flowers-102 (partial) | 2.5 min | 714 | 286 |
| Fine-tune Food-101 (75,750 imgs) | 3.1 h | 9.8 | 3.9 |

A Flowers-102 fine-tune is **0.7% of a single session**. Even Food-101 — the
largest common downstream benchmark — fits nearly 4× over inside one session
and ~10× inside a week's quota, with the 5 GB dataset comfortably under the
20 GB persistent allowance.

**So the free-tier recommendation is use-case-dependent, and that is the whole
point.** The quota that made pre-training absurd is irrelevant to fine-tuning:
the tier flips from *"not a path"* to *"comfortably over-provisioned"*. The
session cap and ephemeral disk — fatal to a 138-session pre-training chain —
never engage when the job is 4.8 minutes and the dataset is 330 MB. **Neither
checkpoint-chaining nor the ToS question below arises at all.**

**Is checkpoint-and-resume across 129 sessions a real path?** Mechanically, on
Kaggle, *partially*: 20 GB persists via Save Version, which is enough for
MobileNet checkpoints — but **not enough for the 144 GB dataset**, which must be
re-attached each session, and the 20 GB is shared with logs and outputs.
Colab is worse: the VM is deleted entirely ("Virtual machines are deleted when
idle for a while, and have a maximum lifetime enforced by the Colab service").

**Does it violate ToS? Here is the honest, non-obvious answer.** Colab's
[FAQ](https://research.google.com/colaboratory/faq.html) restricted-activities
list does **not** prohibit a single user hand-restarting a notebook and resuming
from a checkpoint. What it explicitly prohibits is **every mechanism that would
make it practical at scale**:

> "using multiple accounts to work around access or resource usage
> restrictions… employing techniques such as containerization to circumvent
> anti-abuse policies"

and, for the free tier specifically:

> "remote control such as SSH shells, remote desktops; **bypassing the notebook
> UI to interact primarily via a web UI**… running distributed computing workers"

Combined with "Colab prioritizes users who are actively programming in a
notebook" and "it is not possible to provide more specificity in how our abuse
detection system works", the synthesis is: **a manually-resumed chain is
permitted-but-deprioritized; an automated one is prohibited.** Which settles the
practical question — 129 sessions is not something a human hand-restarts, and
automating it is the prohibited part. Google's own stated remedy is explicit:
"You can fully relax any runtime limits and idle timeouts by **purchasing a
dedicated VM at GCP Marketplace**" — i.e. Google's answer to "I want a long
training run" is "stop using the free tier."

**Unverified, and I am not going to guess**: Kaggle's ToS at
`kaggle.com/terms` is client-rendered and could not be read by any fetch method
tried. **No claim is made here about Kaggle's policy on session chaining.**
Kaggle's *mechanics* (persistent 20 GB, Save Version) support checkpoint-resume
in a way Colab's do not — that is an inference from documented behavior, not
from ToS text. Someone with a browser should read it before relying on this.

Also unverified and therefore **not stated as fact anywhere in this report**:
Colab's disk quota. Google "does not publish these limits, in part because they
can vary over time" — community figures cluster at 35–78 GB, and the widely
repeated "~100 GB" is not sourceable.

## What actually works

### 1. Pretrained weights are the free lunch, and the framing inverts

This is the constructive core. The
[torchvision models table](https://docs.pytorch.org/vision/stable/models.html):

| Weights | Top-1 | Params | GFLOPS |
|---|---|---|---|
| MobileNet_V3_Large **V1** | 74.042 | 5.5 M | 0.22 |
| MobileNet_V3_Large **V2** | **75.274** | 5.5 M | 0.22 |
| MobileNet_V3_Small V1 | 67.668 | 2.5 M | 0.06 |

The paper claims 75.2%. torchvision's *original* weights land at 74.042 —
**1.16 points short**. But torchvision's **V2 weights hit 75.274, beating the
paper**, and timm's `mobilenetv3_large_100.ra4_e3600` reaches **76.31%** — Ross
Wightman beat Google's published number by spending ~6× the epoch budget.
Independently, MMClassification *failed* to reproduce torchvision, getting
[73.39%](https://mmclassification.readthedocs.io/en/dev-1.x/papers/mobilenet_v3.html).

The lesson is not "papers are optimistic." It is: **reproducing a published
top-1 is a research project with a compute budget, and the artifact of that
project is already sitting on a CDN for free.** The `e3600` suffix in timm's
checkpoint name is the receipt for compute you would never rationally spend.

⚠️ **What I could not verify**: any citable statement of what torchvision or
Google actually *spent* to produce these weights. Neither publishes it. The
honest claim is "600 epochs × 8 GPUs, cost undisclosed" — not a dollar figure.

**How many epochs does fine-tuning actually need?** This is what makes the
4.8-minute figure principled rather than lucky, and there are two independent
primary answers:

- **Kornblith et al.** ([arXiv:1805.08974](https://arxiv.org/abs/1805.08974)
  §4.6), 16 networks × 12 datasets: fine-tuning reaches the accuracy bar in
  "an average of **26 epochs/1151 steps**… whereas training from scratch
  required **444 epochs/19531 steps**… corresponding to a **17-fold speedup on
  average**."
- **BiT-HyperRule** ([Kolesnikov et al., arXiv:1912.11370](https://arxiv.org/abs/1912.11370)),
  whose whole thesis is "BiT only needs to be pre-trained once and subsequent
  fine-tuning to downstream tasks is cheap." Its schedule is in
  [`bit_hyperrule.py`](https://github.com/google-research/big_transfer/blob/master/bit_hyperrule.py)
  and prescribes budget purely by dataset size: **<20k examples → 500 steps**;
  <500k → 10k steps; ≥500k → 20k steps. Flowers-102 (2,040) lands in the
  smallest bucket — **500 steps**, against my 620.

The frontier states the ratio outright.
[Smith et al., "ConvNets Match Vision Transformers at Scale" (arXiv:2310.16764)](https://arxiv.org/abs/2310.16764):
their NFNet-F7+ needed "roughly **110k TPU-v4 core hours to pre-train and 1.6k
TPU-v4 core hours to fine-tune**" — **1.45%, or ~1/69th**. Applied to this
report's 69-day pre-training figure that lands, coincidentally, on almost
exactly **one day** — the same order as my measured minutes-to-hours range once
the downstream set is large.

⚠️ **Sources that do *not* support this and are therefore not cited for it**:
[Green AI (arXiv:1907.10597)](https://arxiv.org/abs/1907.10597) does not
quantify pretrain-vs-finetune; [Strubell et al.
(arXiv:1906.02243)](https://arxiv.org/abs/1906.02243) reports no fine-tuning
cost at all — its "tuning" figures are hyperparameter search, a different thing
that is easy to misread as downstream adaptation.

**When to prefer a frozen head.** Kornblith §4.7: "When data is sparse (47-800
total examples), logistic regression is a strong baseline, achieving accuracy
comparable to or better than fine-tuning." The
[PyTorch transfer-learning tutorial](https://docs.pytorch.org/tutorials/beginner/transfer_learning_tutorial.html)
shows the inversion concretely on a ~240-image set: frozen backbone **95.42%**
vs full fine-tune **93.46%**, and faster (28 s vs 36 s on its GPU). At
Flowers-102's 2,040 images I measured the *opposite* ordering (94.65 full vs
90.80 head-only) — consistent with Kornblith, since 2,040 is well past the
sparse regime. **The rule: below ~1k examples freeze; above it, unfreeze at
least the last stage.**

MobileNetV4 checkpoints in
[timm's results CSV](https://github.com/huggingface/pytorch-image-models/blob/main/results/results-imagenet.csv):
`mobilenetv4_conv_aa_large.e230_r448_in12k_ft_in1k` 84.99%,
`mobilenetv4_hybrid_medium.ix_e550_r384_in1k` 83.39%,
`mobilenetv4_conv_medium.e500_r256_in1k` 79.92%. Note timm's card states these
are "the only known MNV4 weights — official weights for Tensorflow models are
unreleased."

### 2. Imagenette is the only benchmark purpose-built for this constraint — but only for from-scratch training

[fastai/imagenette](https://github.com/fastai/imagenette) — sizes HEAD-requested
from the actual S3 objects, not quoted from docs:

| Dataset | Full | 320 px | 160 px | Train | Val |
|---|---|---|---|---|---|
| Imagenette2 | 1.45 GB | 326 MB | **94 MB** | 9,469 | 3,925 |
| Imagewoof2 | 1.25 GB | 313 MB | 88 MB | 9,025 | 3,929 |

Leaderboard: **95.11%** (Imagenette) / 90.38% (Imagewoof) at 256 px, 200 epochs.
The README frames budgets explicitly — "Within a certain budget on AWS or GCP…
$0.05, $0.10, $0.25, $0.50, $1.00, $2.00" — and insists "**Ensure that you start
from random weights - not from pretrained weights.**" Ungated, 94 MB, and
4.1 hours on this laptop. If the goal is *learning to train*, this is the answer.

**That instruction is not a formality — it is the dataset's validity condition.**
Imagenette's classes are ImageNet WNIDs and its images are ImageNet images, so an
ImageNet-pretrained checkpoint has already trained on them with these exact
labels. Random init is the only sound use. **If you want to measure transfer, use
Flowers-102 instead** (0.05% ImageNet duplicate overlap) — that is why the
fine-tuning experiment above does.

**ImageNet-100** is a convention, not an artifact: 100 classes chosen in
[Contrastive Multiview Coding (arXiv:1906.05849)](https://arxiv.org/abs/1906.05849).
There is **no canonical download** — it is a wnid list, so you need gated full
ImageNet to build it. Reproductions exist on HF but inherit the gate's terms.

### 3. Downsampled ImageNet 64×64 is the real sweet spot

[Chrabaszcz, Loshchilov & Hutter (arXiv:1707.08819)](https://arxiv.org/abs/1707.08819)
open with the money quote: "training a strong ImageNet model typically requires
**several GPU months**." Their Table 1 timings are **on a single Titan X** —
literally the "one consumer GPU" conversion:

| Model | Top-1 err | Top-5 err | Params | Days on 1 Titan X |
|---|---|---|---|---|
| WRN-28-2 @32×32 | 56.92% | 30.92% | 1.6 M | **1.5** |
| WRN-28-10 @32×32 | **40.96%** | 18.87% | 37.1 M | 13.8 |
| WRN-36-2 @64×64 | 39.55% | 16.57% | 6.2 M | **6.4** |
| WRN-36-5 @64×64 | **32.34%** | **12.64%** | 37.6 M | 22 |

WRN-28-10 @32×32 "matches the original results by AlexNets (40.7% and 18.2%) on
full-sized ImageNet (which has roughly **50 times more pixels per image**)."

**And it is ungated**: [TFDS](https://www.tensorflow.org/datasets/catalog/downsampled_imagenet)
serves 32×32 at **3.98 GiB** and 64×64 at **11.73 GiB**, full 1,000-class
difficulty, no login. Combined with my measured 982 img/s at 64 px, this is the
one route to *real ImageNet-1k class structure* that fits inside a laptop
overnight.

### 4. Full ImageNet is reachable — but the blocker is the download, not the GPU

[ffcv-imagenet](https://github.com/libffcv/ffcv-imagenet)'s published table
(dollar column is my conversion at [Lambda's $1.99/A100-40GB-hr](https://lambda.ai/pricing),
checked 2026-07-17 — **not** FFCV's figure):

| Arch | Epochs | Top-1 | Setup | Wall | A100-h | ≈$ |
|---|---|---|---|---|---|---|
| ResNet-50 | 88 | 0.784 | 8× A100 | 77.2 min | 10.29 | $20.48 |
| ResNet-50 | 16 | 0.738 | 8× A100 | 14.9 min | 1.99 | $3.95 |
| **ResNet-18** | **88** | **0.724** | **1× A100** | **187.3 min** | **3.12** | **$6.21** |
| ResNet-18 | 16 | 0.669 | 1× A100 | 35.0 min | 0.58 | $1.16 |

**The ResNet-18 rows are the surprise**: 72.4% top-1 on full ImageNet in
**3.1 hours on one GPU**. Note FFCV's headline "Train an ImageNet model on one
GPU in 35 minutes (98¢/model)" sits next to an 8×A100 table — the 35-minute
ResNet-50 row is **8 GPUs**, not one. The framing is loose marketing; the
single-GPU ResNet-18 config is the honest version of the claim.

Which relocates the whole problem: **an overnight run on a single owned
consumer GPU trains ImageNet to respectable accuracy. The 144 GB gated download
and the fast local disk are what actually stop you** — not GPU-hours. That is
the opposite of the intuition.

Corroborating but weaker: [Mosaic ResNet](https://www.databricks.com/blog/mosaic-resnet)
claims "benchmark accuracy on ImageNet in 27 minutes… (ResNet-50 on 8×A100s)"
≈ 3.6 A100-h. ⚠️ The "$15 / 76.6%" figures that circulate for it are **not on
the blog** — unverified, dropped. [fast.ai's DAWNBench](https://www.fast.ai/posts/2018-04-30-dawnbench-fastai.html)
"Imagenet in 3 hours for $25" used 8 GPUs; the famous
["18 minutes for ~$40"](https://www.fast.ai/posts/2018-08-10-fastai-diu-imagenet.html)
used **16 AWS instances × 8 V100 = 128 GPUs** (and 93% is *top-5*) — cheap in
dollars, impossible in access.

### 5. Distillation does not reduce cost — it increases it

Worth stating plainly because the prior report was already suspicious of this
and the evidence now settles it. MobileNetV4 §8: "Training an MNv4-Conv-L
student model for **2000 epochs** yields 85.9% top-1." The augmentation is
"extreme Mixup applied to **1000 ImageNet-1k replicas**." Offline distillation
does remove teacher forward passes from the student loop — but they spend the
savings on 2,000 epochs and then some.

And the headline is worse than expensive, it is **impossible**: MNv4's 87.0%
comes from distillation *plus* pretraining on **JFT-300M resampled to 130M
images**, a **private Google dataset**. That number is not compute-gated, it is
**data-gated — permanently irreproducible at any budget**. Distillation is an
accuracy technique, not a savings technique.

### 6. LAION scale: quantified impossibility

[Cherti et al., "Reproducible scaling laws" (arXiv:2212.07143)](https://arxiv.org/abs/2212.07143),
Appendix Table 18 — trained on JUWELS Booster A100-40GB. Dollar column is my
conversion at $1.99/A100-h:

| Model | Data | #GPUs | Hours | **GPU-h** | MWh | ≈$ | On 1 A100 |
|---|---|---|---|---|---|---|---|
| ViT-B/32 | LAION-2B | 824 | 51 | 42,307 | 14.81 | $84 k | **4.8 yr** |
| ViT-L/14 | LAION-2B | 384 | 319 | 122,509 | 42.88 | $244 k | 14.0 yr |
| **ViT-H/14** | LAION-2B | 824 | 279 | **229,665** | 80.38 | **$457 k** | **26.2 yr** |
| **Total study** | | | | **1,058,318** | **370.41** | **$2.1 M** | 121 yr |

[OpenCLIP's PRETRAINED.md](https://github.com/mlfoundations/open_clip/blob/main/docs/PRETRAINED.md)
corroborates independently: "ViT-B/32 was trained with **128 A100 (40 GB) GPUs
for ~36 hours, 4600 GPU-hours**" → 62.96% zero-shot on LAION-400M; ViT-L/14
"400 A100… **50800 GPU-hours**" → 72.77%.

**ViT-H/14 is 22,319× the GPU-hours of an FFCV ResNet-50.** The *cheapest*
LAION-2B model is 4.8 years on one A100. 370 MWh for the study is roughly 35 US
households' annual electricity. LAION-5B's image embeddings *alone* are 9 TB.

⚠️ **An anomaly I will not paper over.** Within the paper's own Table 18,
B/32 @ LAION-80M / 34B samples = 32,953 GPU-h (344 GPUs × 96 h) but
B/32 @ LAION-400M / **the same 34B samples**, same architecture = 11,177 GPU-h
(128 GPUs × 87 h). Identical samples-seen and architecture should mean identical
FLOPs, yet there is a **3× gap**. The three columns are internally consistent
(#GPUs × hrs ≈ GPU-h for every row), so this is not a transcription error —
it is either real distributed-scaling inefficiency at 344 GPUs, or an error in
the published table. **Real GPU-hours at these scales are configuration-dependent
to within ~3×**, and every dollar figure above inherits that uncertainty. The
conclusion is unaffected: 3× of 26 years is still not a weekend.

## The practices, re-evaluated at this scale

The prior report listed nine levers as uniformly helpful. At ImageNet scale
they stratify sharply, and three of them invert:

| Lever | At CIFAR scale | At ImageNet scale on this laptop |
|---|---|---|
| **Transfer learning** | nice-to-have, 2× | **load-bearing — it IS the answer. Measured: ~20,600×** (69 days → 4.8 min), and +60.1 points over random init at identical cost. The only lever that closes a 46× gap, because it skips the run entirely |
| **Partial unfreeze** (vs full) | rarely worth it | **the best single tuning result here**: 94.91% vs 94.65% at **52% of the wall-clock** |
| **Freezing the trunk entirely** | — | 3.5× faster, −3.9 points at 2k examples; **wins outright below ~1k examples** (Kornblith §4.7) |
| **Resolution reduction** | minor | **load-bearing — 7.6× measured** (64 px vs 224 px). The one knob that moves a rung across the threshold |
| **Efficient architectures** | helpful | **insufficient, and misleading.** MobileNets are cheap per step but trained 4–36× longer; picking MobileNetV3 over ResNet-50 does *not* make ImageNet training affordable |
| **Mixed precision** | 2–3× on NVIDIA | **~6% on MPS — measured.** A rounding error against 46× |
| **Dataloading (FFCV/DALI)** | irrelevant | **irrelevant on this laptop (17.4× headroom, measured)**; headroom erodes to 4.5× at head-only fine-tuning, and would bind first on a free-tier notebook's 2–4 vCPUs |
| **One-cycle LR / early stopping** | 33% | real but bounded — cannot bridge two orders of magnitude |
| **Distillation** | "makes models accurate, not cheap" | **confirmed and stronger: it makes training *more* expensive** (2,000 epochs), and MNv4's headline is JFT-gated |
| **Subset / downsampled datasets** | trivially available | **load-bearing — the practical answer** (Imagenette 94 MB; Downsampled ImageNet 11.7 GB, ungated) |
| **Speedrun bag of tricks** | steal freely | mostly CIFAR-specific; the FFCV single-GPU ResNet-18 config is the ImageNet-scale analogue |

The pattern: **levers that shrink the problem (transfer learning, resolution,
subsets) work. Levers that speed up the run (AMP, dataloading, schedules) do
not**, because no realistic multiplier on a laptop bridges 46×. Transfer
learning wins not by being a *better* 46× — it wins by making the run 20,600×
smaller instead of faster.

## What changed from the previous version of this report

Recorded because the reversal is the finding:

- **"Yes, trivially" → "scale-dependent, with a sharp threshold."** The prior
  conclusion was correct for the evidence it had (MNIST/CIFAR-10) and wrong as
  a general claim. Extending to ImageNet inverted it.
- **AMP was listed as a 2–3× lever.** Measured on MPS: 6%. The 2–3× is real but
  NVIDIA-specific, and the prior report's parenthetical "MPS gains are smaller"
  understated how completely it fails to matter here.
- **"Disk and RAM are non-issues"** was true of MNIST's 12 MB and CIFAR-10's
  170 MB. At ImageNet it is 144 GB behind a non-commercial agreement, and at
  LAION it is 50 TB you download yourself over ~46 days.
- **macOS version corrected** (26.5.1, not 15).
- **Retained unchanged**: the MNIST 21 s / CIFAR-10 8.5 min / MobileNetV3-Small
  fine-tune 92.5% in 4.9 min measurements, and the CIFAR-10 speedrun lineage.
  They were right; they were just the small end of a spectrum, and the 4.9-min
  fine-tune result reads differently now — it is a preview of the only strategy
  that survives at scale.
- **Fine-tuning quantified** (this revision). The first version of this report
  proved the door was locked (69 days, 46× over budget) but left the open door
  as a qualitative "pretrained weights are the free lunch." That was a real gap:
  the alternative deserved the same rigor as the verdict. It now has a measured
  ratio (**~20,600×**), a budget-matched scratch control (**+60.1 points at
  identical cost**), its own place on the threshold axis (**bounded at ~890k
  images**), and a reversed free-tier verdict.
- **An internal inconsistency fixed**: the first version costed the same
  torchvision recipe at both 64.5 and 69.0 days, because two throughput
  measurements (137.9 and 128.9 img/s) were mixed across tables. All
  MobileNetV3-Large figures now use the conservative 128.9, which also moves the
  Kaggle figures from 51.6 weeks/129 sessions to **55.2 weeks/138 sessions**.

## Sources

**First-hand measurements** (Apple M4 MacBook, 16 GB, 10 cores, macOS 26.5.1,
Python 3.14.6, PyTorch 2.13.0, torchvision 0.28.0, timm 1.0.28; 2026-07-17).
Scripts retained in the session scratchpad, methodology fully described above:
`bench_throughput.py` (synthetic-input training throughput, warmup +
`mps.synchronize()`-bracketed timing), `bench_dataloader.py` (real
ImageFolder/DataLoader/transform pipeline over size-calibrated synthetic JPEGs),
**`finetune_flowers.py`** (the four-arm Flowers-102 experiment — real data, real
dataloader, end-to-end wall-clock and test accuracy),
**`bench_finetune_modes.py`** (per-arm pure-compute ceilings),
`ladder.py` + `extrapolate.py` + `finetune_arithmetic.py` (arithmetic from
measured rates × published epoch counts). All extrapolations are stated as such;
the Flowers-102 accuracies are measured, not extrapolated. Flowers-102 splits
(1020/1020/6149, 102 classes) were verified first-hand from `setid.mat`.

**Papers**: [MobileNetV3, arXiv:1905.02244](https://arxiv.org/abs/1905.02244) ·
[Do Better ImageNet Models Transfer Better?, arXiv:1805.08974](https://arxiv.org/abs/1805.08974) ·
[Big Transfer (BiT), arXiv:1912.11370](https://arxiv.org/abs/1912.11370) ·
[ConvNets Match Vision Transformers at Scale, arXiv:2310.16764](https://arxiv.org/abs/2310.16764) ·
[How transferable are features?, arXiv:1411.1792](https://arxiv.org/abs/1411.1792) ·
[MobileNetV4, arXiv:2404.10518](https://arxiv.org/abs/2404.10518) ·
[ResNet Strikes Back, arXiv:2110.00476](https://arxiv.org/abs/2110.00476) ·
[Downsampled ImageNet, arXiv:1707.08819](https://arxiv.org/abs/1707.08819) ·
[FFCV, arXiv:2306.12517](https://arxiv.org/html/2306.12517) ·
[Reproducible scaling laws, arXiv:2212.07143](https://arxiv.org/abs/2212.07143) ·
[LAION-5B, arXiv:2210.08402](https://arxiv.org/abs/2210.08402) ·
[Contrastive Multiview Coding, arXiv:1906.05849](https://arxiv.org/abs/1906.05849) ·
[JPEG decoder benchmark, arXiv:2501.13131](https://arxiv.org/html/2501.13131v1) ·
[Link-Rot in Web-Sourced Multimedia Datasets, MMM 2023](https://link.springer.com/chapter/10.1007/978-3-031-27077-2_37) (medium confidence — PDF host unreachable).

**Code and recipes**: [torchvision references/classification](https://github.com/pytorch/vision/blob/main/references/classification/README.md) ·
[torchvision recipe blog](https://pytorch.org/blog/how-to-train-state-of-the-art-models-using-torchvision-latest-primitives/) ·
[torchvision models table](https://docs.pytorch.org/vision/stable/models.html) ·
[timm](https://github.com/huggingface/pytorch-image-models) ·
[timm results-imagenet.csv](https://github.com/huggingface/pytorch-image-models/blob/main/results/results-imagenet.csv) ·
[rwightman MobileNetV4 hparams gist](https://gist.github.com/rwightman/f6705cb65c03daeebca8aa129b1b94ad) ·
[ffcv](https://github.com/libffcv/ffcv) · [ffcv-imagenet](https://github.com/libffcv/ffcv-imagenet) ·
[FFCV benchmarks](https://docs.ffcv.io/benchmarks.html) ·
[NVIDIA DALI](https://github.com/NVIDIA/DALI) · [webdataset](https://github.com/webdataset/webdataset) ·
[img2dataset](https://github.com/rom1504/img2dataset) + [LAION-5B recipe](https://github.com/rom1504/img2dataset/blob/main/dataset_examples/laion5B.md) ·
[OpenCLIP PRETRAINED.md](https://github.com/mlfoundations/open_clip/blob/main/docs/PRETRAINED.md) ·
[fastai/imagenette](https://github.com/fastai/imagenette) ·
[PyTorch AMP recipe](https://docs.pytorch.org/tutorials/recipes/recipes/amp_recipe.html) ·
[PyTorch transfer-learning tutorial](https://docs.pytorch.org/tutorials/beginner/transfer_learning_tutorial.html) ·
[big_transfer/bit_hyperrule.py](https://github.com/google-research/big_transfer/blob/master/bit_hyperrule.py).

**Datasets and terms**: [Flowers-102](https://www.robots.ox.ac.uk/~vgg/data/flowers/102/)
(`torchvision.datasets.Flowers102`) · [Food-101](https://data.vision.ee.ethz.ch/cvl/datasets_extra/food-101/) ·
[image-net.org/download](https://www.image-net.org/download.php) ·
[HF ILSVRC/imagenet-1k](https://huggingface.co/datasets/ILSVRC/imagenet-1k) ·
[LAION maintenance note](https://laion.ai/notes/laion-maintenance/) ·
[Re-LAION-5B](https://laion.ai/blog/relaion-5b/) · [LAION FAQ](https://laion.ai/faq/) ·
[HF relaion2B-en-research-safe](https://huggingface.co/datasets/laion/relaion2B-en-research-safe) ·
[TFDS downsampled_imagenet](https://www.tensorflow.org/datasets/catalog/downsampled_imagenet).

**Free tiers**: [Kaggle notebooks docs](https://www.kaggle.com/docs/notebooks) ·
[Kaggle efficient GPU usage](https://www.kaggle.com/docs/efficient-gpu-usage) ·
[Kaggle datasets docs](https://www.kaggle.com/docs/datasets) ·
[Colab FAQ](https://research.google.com/colaboratory/faq.html) ·
[Lambda pricing](https://lambda.ai/pricing) (for A100-hour → dollar conversions).
Kaggle's docs pages are client-rendered; their quota figures come from search
snippets of the official pages rather than a firsthand read — flagged above.

**Explicitly unverified, and therefore not asserted**: Kaggle's ToS language on
session chaining (page unreadable by any method tried); Colab's disk quota
(Google declines to publish it); torchvision's/Google's actual training spend
(never published); MosaicML's "$15 / 76.6%"; MobileNetV3's total epoch count;
MobileNetV4's training hardware; **Flowers-102's download size** (no
`Content-Length` served anywhere — the commonly quoted ~330 MB is unconfirmed);
**any MobileNetV3-on-Flowers-102/Food-101 figure** (no primary source exists;
Kornblith predates V3, so MobileNet v1/v2 is used as the proxy); **whether
decode binds at head-only fine-tuning on a free-tier notebook** (direction
argued from measured headroom erosion, but no notebook was benchmarked);
CUB-200-2011's split sizes; Oxford-IIIT Pets' test size (Kornblith's Tables 1
and H.1 disagree: 3,369 vs 3,669).
