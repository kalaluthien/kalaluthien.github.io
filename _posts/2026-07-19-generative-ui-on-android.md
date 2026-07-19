---
share: true
title: "Generative UI on Android: Runtime LLM-Rendered Compose and On-Device GenAI Building Blocks"
categories:
  - Report
  - Android
tags:
  - android
  - generative-ui
  - jetpack-compose
  - gemini-nano
  - on-device-ai
author: claude
---

> State-of-the-art Generative UI practice and toolkit for Android. Examples, use cases, and specific design patterns that use GenUI. What is missing for now (limitations) and what can be done easily.

## Short answer

"Generative UI on Android" splits into two things that are at very different levels of maturity, and it matters which one you mean.

1. **Runtime LLM-rendered UI** — an LLM assembling the actual Jetpack Compose UI at runtime (the Android analogue of Vercel AI SDK's `streamUI`, where a tool call streams back a component). **There is no first-party framework for this on Compose.** Google's own generative-UI bet, A2UI, deliberately shipped Flutter/web/Angular/React renderers first and lists Jetpack Compose only on its roadmap. You build it yourself by gluing two pieces that *do* exist: a **server-driven-UI (SDUI) render substrate** (DivKit is the mature open one; AndroidX **Remote Compose** is Google's new, alpha, on-Maven one) plus **Gemini function calling** (via the Firebase AI Logic SDK) as the driver. The pattern is stable and well-understood; the packaging is not.

2. **On-device GenAI building blocks** — Google's **ML Kit GenAI APIs** (Summarization, Rewriting, Proofreading, Image Description, Prompt, Speech Recognition) running on **Gemini Nano** via the **AICore** system service, plus the **Firebase AI Logic** SDK for cloud Gemini and an experimental on-device/cloud **hybrid**. This half is real and callable today — mostly at **beta**, one API a single factory-plus-call away — but gated by hard **device fragmentation**: Nano is effectively a Pixel-9/10-class feature, and the long tail of Android has no on-device model at all and must fall back to cloud.

**Easiest wins today:** the ML Kit GenAI feature APIs (one `getClient()` + `runInference()`), Firebase AI Logic's `generateContentStream()` streaming a Kotlin `Flow` straight into a recomposing `Text`, and Compose's own state/recomposition model, which makes token-streaming UIs fall out for free. **The real gap** is the runtime-generation layer: no standard schema, no first-party `streamUI` equivalent, and an unavoidable security burden (a model may only *select and parameterize* pre-approved Composables from a whitelist, never inject layout).

Everything below is the evidence. Versions and maturity are as of 2026-07-19 — this is a fast-moving area, so each claim carries its source.

## The two centers of gravity

The phrase collapses two distinct ideas that share a slogan:

- **Runtime generation** — the model decides *what UI to show* and the app renders it. This is the "generative UI" the web means (Vercel AI SDK). On Android it is a DIY assembly, not a product.
- **On-device GenAI features** — the model generates *content* (a summary, a rewrite, a caption) and the app shows it in ordinary UI. This is where Google has actually shipped SDKs, and it forces its own interaction patterns (streaming, download/thinking states, attribution, graceful degradation).

Build-time senses of "generative UI" — AI design-to-code tools (v0, Stitch) and plain non-AI SDUI — are out of scope here as centers; SDUI appears below only as the *substrate* runtime generation builds on.

## Center 1 — Runtime LLM-generated UI on Compose

### The web reference point

Vercel open-sourced v0's technique as **AI SDK 3.0** (2024-03-01): an LLM returns **React Server Components** instead of text. The current entry point, **`streamUI`** in `@ai-sdk/rsc`, takes a model plus a set of **tools** (`{ description, inputSchema, generate }`); when the model calls a tool, `generate` is an **async generator** that can `yield <Loading/>` then `return <Final/>`, and the resulting node streams server→client. The mapping is literally **tool call → component**, with partial rendering built in. Status flag worth carrying: AI SDK RSC is now marked **experimental and paused**, with Vercel steering production apps to the plainer `useChat` — the streaming-component paradigm is still unsettled even on the web ([AI SDK 3.0 blog](https://vercel.com/blog/ai-sdk-3-generative-ui); [`streamUI` docs](https://ai-sdk.dev/docs/ai-sdk-rsc/streaming-react-components)).

### There is no first-party Compose analogue

Google's generative-UI effort is **A2UI** ("Agent-to-UI"), an open, framework-agnostic JSON format where an agent describes a component tree and the client renders it against its own native catalog. It was announced 2025-12-15 (v0.8) and reached v0.9 on 2026-04-17. Its renderers are **Lit, Angular, Flutter, Web Components, and (newly landed) React** — **Jetpack Compose is not a renderer; it is explicitly on the roadmap** ("working toward adding official support for React, Jetpack Compose, iOS (SwiftUI), and more"). So A2UI reaches Android today only through **Flutter's GenUI SDK**, not native Compose ([Introducing A2UI](https://developers.googleblog.com/introducing-a2ui-an-open-project-for-agent-driven-interfaces/); [A2UI v0.9](https://developers.googleblog.com/a2ui-v0-9-generative-ui/); [a2ui.org roadmap](https://a2ui.org/roadmap/)).

How I know there is no first-party Compose product: on the Android side Google ships only the *plumbing* — Firebase AI Logic Gemini **function calling** (`FunctionDeclaration`, `Tool.functionDeclarations`), the Gemini Live API, and MediaPipe on-device inference — and none of it has a component-streaming abstraction analogous to `streamUI`. The function-calling docs return **data** to the model, not renderable components ([Firebase function calling](https://firebase.google.com/docs/ai-logic/function-calling)). The absence is the finding.

### The substrate: server-driven UI on Android

Runtime GenUI needs a render layer that turns a *description* into native views. The real options:

| Substrate | What it is | Coordinates / status |
|---|---|---|
| **DivKit** (Yandex) | Mature open-source SDUI; `div-json` schema → native components on Android/iOS/Web; DSLs in Kotlin/TS/Python. | `com.yandex.div:div` (very active, 32.x line); Apache-2.0; ~2.6k★. A newer `com.yandex.div:compose` renderer exists alongside the mature View-based one. |
| **AndroidX Remote Compose** | Google's own. You write Compose-like code on plain JVM; `RemoteComposeWriter` serializes it to a compact **binary** document; `RemoteDocumentPlayer`/`RemoteComposePlayer` renders it **natively** (no WebView, no recompile). Richer than JSON schema — captures shapes/text/images/animations/touch/expressions. | `androidx.compose.remote:remote-core` / `remote-creation` / `remote-player-view` **1.0.0-alpha15** (2026-07-15), **on Google Maven**. **Alpha/experimental.** |
| **compose-remote-layout** (utsmannn) | Community JSON→Compose Multiplatform; custom Composables registered in a registry, driven by a `BindsValue` system. | `io.github.utsmannn:compose-remote-layout:0.2.0-alpha01` (2025-03); alpha, not production. |
| **skydoves/server-driven-compose** | Reference sample: Firebase RTDB streams a layout → `Flow` → each node rendered by a `Consume()` composable, with component versioning. | Sample, not a library; good pattern to copy. |
| **Airbnb Ghost Platform** | Native Kotlin SDUI over a shared GraphQL schema (Sections + Screens). Internal, not open-sourced. | [Airbnb SDUI deep dive](https://medium.com/airbnb-engineering/a-deep-dive-into-airbnbs-server-driven-ui-system-842244c5f5) |

Precedent worth naming: **Jetpack Glance** already remote-renders Compose-style code to `RemoteViews` in the launcher process — Google does out-of-process "remote Compose" for widgets, just not for model-driven UI.

Sources: [AndroidX Remote Compose release page](https://developer.android.com/jetpack/androidx/releases/compose-remote); [DivKit](https://github.com/divkit/divkit); [compose-remote-layout](https://github.com/utsmannn/compose-remote-layout); [server-driven-compose](https://github.com/skydoves/server-driven-compose).

The strongest honest "Compose equivalent of `streamUI`" you can assemble today is **Remote Compose (or DivKit) as the substrate + Gemini function calling as the driver** — nobody has shipped the two glued into a product.

### Design patterns (runtime generation)

- **Tool-call → Composable mapping.** Gemini function calling returns a `FunctionCall` (name + JSON args). Treat the name as a **component key** and the args as **props**, then render via a registry `Map<String, @Composable (Props) -> Unit>`. This is the direct analogue of `streamUI`'s tool→component step — except *you* own the render, because Firebase gives you the tool call, not the component.
- **Whitelist / trusted-Composable registry (the security boundary).** The model never ships code; it ships a descriptor whose `type` **names a Composable the app already compiled**. The renderer walks the tree, looks up each `type` in the registry, and invokes the bound Composable with the props; any unknown `type` is dropped. The model can only **select and parameterize** approved components — this is mandatory, not optional (see limitations).
- **Streaming / partial rendering.** Compose makes this nearly free: collect model output into a `StateFlow`/`MutableState`, expose via `collectAsStateWithLifecycle()`, and each emission triggers recomposition — a growing UI tree renders incrementally with no manual diffing. (MediaPipe's `generateResponseAsync()` and Gemini Live stream tokens the same way.)
- **Optimistic / thinking / error states.** Mirror `streamUI`'s "yield loading → return final": model a sealed `UiState { Thinking, Streaming(partialTree), Complete(tree), Error }` and `when`-branch in Compose. Because latency is seconds not frames, a per-tool-call skeleton is standard.
- **Attribution as a whitelisted component.** Make provenance first-class — source chips, "AI-generated" badges, citation footnotes — as Composables the model can attach via the same registry.

## Center 2 — On-device GenAI building blocks

### ML Kit GenAI APIs

Announced at I/O 2025 (2025-05-20) with four features; the set is now **six**. All run on Gemini Nano through AICore and share one shape — a feature factory returns a client, and one call runs inference (streaming available as a callback overload):

| Feature | Gradle coordinate | Maturity | Notes |
|---|---|---|---|
| Summarization | `com.google.mlkit:genai-summarization:1.0.0-beta1` | Beta | article/chat → bullets; EN/JA/KO |
| Rewriting | `com.google.mlkit:genai-rewriting:1.0.0-beta1` | Beta | styles (ELABORATE/EMOJIFY/SHORTEN/FRIENDLY/PROFESSIONAL/REPHRASE); input **< 256 tokens** |
| Proofreading | `com.google.mlkit:genai-proofreading:1.0.0-beta1` | Beta | grammar/spelling on short text |
| Image Description | `com.google.mlkit:genai-image-description:1.0.0-beta1` | Beta | Bitmap → caption |
| Prompt (custom/multimodal) | `com.google.mlkit:genai-prompt` (**1.0.0-beta3**, 2026-07-14) | Beta (was alpha at 2025-10 launch) | text + image; `Generation.getClient().generateContent(...)` |
| Speech Recognition | genai speech | **Alpha** | Basic (API 31+) + Advanced (Pixel 10) |

API surface (Summarization as the template): `Summarization.getClient(SummarizerOptions)` → `Summarizer`; request/result `SummarizationRequest`/`SummarizationResult`; async as `ListenableFuture` or coroutine `.await()`; **streaming** via `runInference(request) { newText -> ... }`; `summarizer.close()` to release. Each feature = shared Nano base model + a small **feature-specific LoRA adapter** pulled on demand; the base model is managed by AICore and shared across apps.

The UX-critical piece is the **availability + download API**: `checkFeatureStatus()` returns a `FeatureStatus` of `UNAVAILABLE | DOWNLOADABLE | DOWNLOADING | AVAILABLE`, and `downloadFeature(DownloadCallback)` reports `onDownloadStarted/Progress/Completed/Failed`. That four-state enum is what forces the check→download→fallback flow below.

Sources: [ML Kit GenAI overview](https://developers.google.com/ml-kit/genai); [Summarization/Android](https://developers.google.com/ml-kit/genai/summarization/android); [release notes](https://developers.google.com/ml-kit/release-notes); [I/O 2025 announcement](https://android-developers.googleblog.com/2025/05/on-device-gen-ai-apis-ml-kit-gemini-nano.html).

### Gemini Nano + AICore, and the fragmentation wall

**AICore** is the Android system service that hosts on-device foundation models — it manages weights + LoRA adapters, hardware-accelerated inference, safety filters, and automatic model distribution, under Private Compute Core isolation (no direct internet, no input/output retention). Apps reach Nano through **ML Kit GenAI** (recommended) or the lower-level **Google AI Edge SDK** (experimental) ([AICore / Gemini Nano](https://developer.android.com/ai/gemini-nano)).

Published performance (primary): prefill **510 tok/s** on Pixel 9 Pro (nano-v2) and **940 tok/s** on Pixel 10 Pro (nano-v3); decode ~**11 tok/s** on Pixel 9 Pro; image-to-text adds ~0.6–0.8 s of image encoding ([Aug 2025 Nano blog](https://android-developers.googleblog.com/2025/08/the-latest-gemini-nano-with-on-device-ml-kit-genai-apis.html)).

The fragmentation is the defining limitation. Google's own docs name only **Pixel 10 (nano-v3)** and **Pixel 9 / 9 Pro (nano-v2)**, with chipset support stated generically ("MediaTek Dimensity, Qualcomm Snapdragon, Google Tensor"). Broader multi-vendor device lists (Galaxy S24/S25, Xiaomi 15, etc.) exist only in secondary roundups. Hard gates: min **API 26**, and **not supported on devices with an unlocked bootloader**. The rest of the Android install base has **no Nano at all** and must use cloud.

### Firebase AI Logic (cloud, and hybrid)

**Rename lineage:** "Vertex AI in Firebase" → **Firebase AI Logic** (May 2025), because it now supports two backends — the **Gemini Developer API** (free tier) and the **Vertex AI Gemini API** — switchable at init ([migration guide](https://firebase.google.com/docs/ai-logic/migrate-to-latest-sdk)).

Coordinates and surface: `com.google.firebase:firebase-ai` (via Firebase BoM); `Firebase.ai(backend = GenerativeBackend.googleAI()).generativeModel(...)` → `GenerativeModel`; `generateContent()` for single-shot and **`generateContentStream()` returning a Kotlin `Flow`** of chunks — the streaming primitive you collect into Compose state.

**Hybrid on-device + cloud** is now a single API (Android, **experimental**, announced 2026-04-17): pass an `OnDeviceConfig` and choose an `InferenceMode` — **`PREFER_ON_DEVICE`, `ONLY_ON_DEVICE`, `PREFER_IN_CLOUD`, `ONLY_IN_CLOUD`** (verified against the config-options doc). `PREFER_*` fall back automatically; `ONLY_*` throw if their target is unavailable; the response reports which path ran. The on-device leg is constrained to **single-turn** generation over **text or a single Bitmap**. On-device artifact: `com.google.firebase:firebase-ai-ondevice:16.0.0-beta01`. (Note the platform split: on Web, hybrid's on-device leg is Chrome-hosted Nano via the W3C Prompt API, not AICore.)

Sources: [Firebase AI Logic overview](https://firebase.google.com/docs/ai-logic); [hybrid config options](https://firebase.google.com/docs/ai-logic/hybrid/android/configuration-options); [Android hybrid announcement](https://developer.android.com/blog/posts/experimental-hybrid-inference-and-new-gemini-models-for-android).

### Interaction patterns these force

- **Streaming tokens into Compose.** Two mechanisms — ML Kit's `runInference(request){ newText -> }` callback, and Firebase's `generateContentStream()` Flow. Canonical pattern: collect the Flow, accumulate into `mutableStateOf`, render a `Text` that grows as chunks arrive. Forced, because on-device first-token→full-response latency is user-visible.
- **Two distinct waits.** `FeatureStatus.DOWNLOADING` + `onDownloadProgress()` demand a **one-time model/adapter download UI** that is *separate* from a per-request inference spinner. Conflating them is a common mistake.
- **Transparency / attribution — there is real Google guidance.** Google Play policy requires a visible disclosure when content is AI-generated, and the developer owns the generated content. Design guidance lives in the **People + AI Guidebook** (PAIR) — chapters *Explainability + Trust*, *Mental Models*, *Feedback + Control*, *Errors + Graceful Failure* — and the **Responsible Generative AI Toolkit**. Together they back the "AI-generated" label plus "check for mistakes" disclaimer pattern ([PAIR Guidebook](https://pair.withgoogle.com/guidebook/); [Responsible GenAI Toolkit](https://ai.google.dev/responsible/docs/design)).
- **Graceful degradation.** The forced flow is **check → download → fallback**: `checkFeatureStatus()` → if `DOWNLOADABLE`, `downloadFeature()` with progress UI → if `UNAVAILABLE` (unsupported device), fall back to cloud. Firebase hybrid folds this into one line via `InferenceMode.PREFER_ON_DEVICE` — but that path is still experimental.

## What's missing (limitations, and how I know)

- **No first-party runtime-generation framework for Compose.** Confirmed by absence in Android docs and by A2UI's own roadmap listing Compose as not-yet-supported while Flutter/web/Angular/React shipped. The `streamUI` equivalent is DIY.
- **No standard schema.** Every substrate rolls its own: DivKit `div-json`, compose-remote-layout's node/BindsValue JSON, Airbnb's GraphQL Sections, Remote Compose's binary doc, A2UI's format. **Zero interop** — choosing one is a lock-in decision.
- **Security of model-proposed UI is on you.** Rendering arbitrary model output as layout is an injection surface (phishing UIs, unexpected actions, deep-link/`PendingIntent` abuse — Remote Compose exposes `HostAction`/`PendingIntentAction` that an unvalidated remote doc could trigger). A whitelist/registry is **mandatory**; every substrate above enforces "select a known component," never "eval arbitrary code."
- **State and recomposition with dynamic trees.** Streamed node trees lack stable identity, so Compose can't key them well — lost scroll/focus/animation state and wasted recomposition unless you synthesize stable keys and hoist state. The skydoves sample carries explicit component versioning precisely because schema drift breaks clients.
- **Device fragmentation for on-device.** nano-v3 is Pixel-10-class; nano-v2 reaches Pixel 9 and some flagships; the long tail has no Nano. Google's 2026 "Gemini Intelligence" tier (≥12 GB RAM, 2026 flagship chip, nano-v3) shows how narrow the newest frontier is.
- **On-device model constraints (documented).** Rewriting input < 256 tokens; hybrid on-device is single-turn and text-or-single-Bitmap only; features are language-limited; output quality varies with the device's Nano version. Not published in primary docs: Nano's parameter count, model/adapter download size in MB, and first-token latency — those numbers are **absent**, not merely hard to find.
- **Maturity risk across the board.** ML Kit GenAI features are **beta** (Speech Recognition alpha); Firebase hybrid/on-device and Google AI Edge SDK are **experimental**; Remote Compose is **alpha**; even the web's AI SDK RSC is paused. Coordinates move (the Prompt API went alpha→beta in ~9 months). ML Kit GenAI is Android-only (no iOS).

## What can be done easily today

- **ML Kit GenAI = one factory + one call** for summarize / rewrite / proofread / describe, with streaming as a lambda overload and a built-in 4-state availability enum + download callbacks for the UX.
- **Firebase AI Logic = cloud Gemini with Flow streaming into Compose** in a few lines: `generateContentStream()` → collect Flow → incremental `Text`.
- **Token-streaming UIs are essentially free in Compose** — `StateFlow` + `collectAsState` + recomposition give incremental rendering with no diffing code.
- **A working runtime-GenUI loop is assemblable now** from an existing substrate (DivKit for mature, Remote Compose for native-and-Google-published, or the skydoves/compose-remote-layout registry pattern to copy) + Gemini function calling returning `{key, props}` descriptors.
- **One-line on-device-then-cloud fallback** via `InferenceMode.PREFER_ON_DEVICE` (Android, experimental) — accepting the experimental flag.

## Sources

Primary (Google/Vercel/Firebase docs, official blogs, official repos):

- [Vercel AI SDK 3.0: Generative UI](https://vercel.com/blog/ai-sdk-3-generative-ui) — the web reference point; tool → RSC.
- [AI SDK `streamUI` docs](https://ai-sdk.dev/docs/ai-sdk-rsc/streaming-react-components) — generator yield-loading pattern; "experimental, use AI SDK UI."
- [Introducing A2UI](https://developers.googleblog.com/introducing-a2ui-an-open-project-for-agent-driven-interfaces/) and [A2UI v0.9](https://developers.googleblog.com/a2ui-v0-9-generative-ui/) — renderers Lit/Angular/Flutter/React; no Compose.
- [A2UI roadmap](https://a2ui.org/roadmap/) — Jetpack Compose/iOS listed as future support.
- [Firebase AI Logic — function calling](https://firebase.google.com/docs/ai-logic/function-calling) — `FunctionDeclaration`, returns data not components.
- [AndroidX Remote Compose release page](https://developer.android.com/jetpack/androidx/releases/compose-remote) — `androidx.compose.remote:*` 1.0.0-alpha15 (2026-07-15), Google Maven.
- [DivKit](https://github.com/divkit/divkit) — open-source SDUI, div-json, `com.yandex.div:div`.
- [compose-remote-layout](https://github.com/utsmannn/compose-remote-layout) — JSON→Compose registry, `io.github.utsmannn:compose-remote-layout:0.2.0-alpha01`.
- [skydoves/server-driven-compose](https://github.com/skydoves/server-driven-compose) — Firebase-streamed SDUI sample, `Consume()`, versioning.
- [Airbnb SDUI deep dive](https://medium.com/airbnb-engineering/a-deep-dive-into-airbnbs-server-driven-ui-system-842244c5f5) — Ghost Platform (Sections/Screens, GraphQL).
- [ML Kit GenAI overview](https://developers.google.com/ml-kit/genai) and [Summarization/Android](https://developers.google.com/ml-kit/genai/summarization/android) — feature set, classes, FeatureStatus/downloadFeature.
- [ML Kit release notes](https://developers.google.com/ml-kit/release-notes) — coordinates + dates (genai betas 2025-05-14, genai-prompt beta3 2026-07-14).
- [On-device GenAI APIs (I/O 2025)](https://android-developers.googleblog.com/2025/05/on-device-gen-ai-apis-ml-kit-gemini-nano.html) — announcement; base model + LoRA architecture.
- [The latest Gemini Nano (Aug 2025)](https://android-developers.googleblog.com/2025/08/the-latest-gemini-nano-with-on-device-ml-kit-genai-apis.html) — nano-v2/v3; 510/940 tok/s.
- [AICore / Gemini Nano](https://developer.android.com/ai/gemini-nano) — system service, access paths, Private Compute Core.
- [Firebase AI Logic overview](https://firebase.google.com/docs/ai-logic) and [migration guide](https://firebase.google.com/docs/ai-logic/migrate-to-latest-sdk) — rename lineage, two backends, `firebase-ai`, `generateContentStream`.
- [Firebase hybrid config options](https://firebase.google.com/docs/ai-logic/hybrid/android/configuration-options) — `InferenceMode` (4 values), single-turn text/Bitmap constraint.
- [Experimental hybrid inference for Android](https://developer.android.com/blog/posts/experimental-hybrid-inference-and-new-gemini-models-for-android) — 2026-04-17 announcement; `firebase-ai-ondevice:16.0.0-beta01`.
- [People + AI Guidebook (PAIR)](https://pair.withgoogle.com/guidebook/) and [Responsible GenAI Toolkit](https://ai.google.dev/responsible/docs/design) — transparency, attribution, graceful-failure guidance.
- [Androidify case study](https://android-developers.googleblog.com/2025/09/androidify-ai-gemini-android-jetpack-compose-firebase-camerax.html) — Google's reference AI-first Compose app (illustrates the DIY loop).
- [MediaPipe on-device LLM inference (Android)](https://developers.google.com/edge/mediapipe/solutions/genai/llm_inference/android) — `generateResponseAsync()` token streaming.

Secondary (treat as unverified; used only where primary docs are silent): multi-vendor Nano device lists and the 2026 "Gemini Intelligence" hardware requirements ([9to5Google](https://9to5google.com/2026/05/15/gemini-intelligence-android-spec-requirements/)); Google Play AI-content disclosure policy summaries — verify against the Play Console policy center before quoting.
