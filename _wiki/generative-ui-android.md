---
share: true
title: Generative UI on Android
categories:
  - Wiki
tags:
  - generative-ui
  - android
  - jetpack-compose
  - on-device-ai
  - gemini-nano
author: claude
type: concept
created: 2026-07-19
updated: 2026-07-19
sources:
  - "[[2026-07-19-generative-ui-on-android]]"
aliases:
  - genui-android
  - on-device-genai-android
---

# Generative UI on Android

"Generative UI on Android" names two different things at very different maturity
levels. Keeping them apart is the whole point.

1. **Runtime LLM-generated UI** — the model decides *what UI to show* and the
   app renders it (the Android analogue of Vercel AI SDK's `streamUI`). **No
   first-party framework exists for Jetpack Compose.** You assemble it from an
   SDUI substrate + Gemini function calling.
2. **On-device GenAI building blocks** — the model generates *content* (summary,
   rewrite, caption) shown in ordinary UI. Real, shipping SDKs (ML Kit GenAI,
   Gemini Nano, Firebase AI Logic), mostly at beta, gated by device
   fragmentation.

## Center 1 — runtime UI generation (Compose)

- **Web reference point:** Vercel AI SDK `streamUI` (`@ai-sdk/rsc`) — a tool call
  returns a streamed React component; `generate` is an async generator that
  yields a loading component then returns the final one. Now marked
  experimental/paused even on the web.
- **No first-party Compose analogue.** Google's generative-UI bet, **A2UI**
  (agent sends a declarative JSON UI tree; client renders against its native
  catalog), shipped renderers for Lit/Angular/Flutter/Web-Components/React and
  lists **Jetpack Compose only on its roadmap** as of mid-2026. A2UI reaches
  Android today only via Flutter's GenUI SDK. On native Android, Google ships
  the plumbing (Gemini function calling, Gemini Live, MediaPipe inference) but
  no `streamUI`-style component-streaming abstraction — function calling returns
  *data*, not components.
- **SDUI substrates** (the render layer runtime generation builds on):
  - **DivKit** (Yandex) — mature open SDUI, `div-json` schema, `com.yandex.div:div`
    (plus a newer `com.yandex.div:compose` renderer). Apache-2.0.
  - **AndroidX Remote Compose** — Google's own; Compose-like code →
    `RemoteComposeWriter` → compact **binary** doc → `RemoteDocumentPlayer`
    renders natively (no WebView). `androidx.compose.remote:remote-*`
    **1.0.0-alpha15** (2026-07-15), on Google Maven, **alpha**.
  - **compose-remote-layout** (utsmannn) — community JSON→Compose registry
    (`BindsValue`), alpha.
  - **skydoves/server-driven-compose** — reference sample: Firebase-streamed
    layout `Flow` → `Consume()` composable, with component versioning.
  - **Airbnb Ghost Platform** — native Kotlin SDUI over shared GraphQL
    (Sections/Screens); internal, not open.
  - **Jetpack Glance** — precedent: already remote-renders Compose-style code to
    `RemoteViews` in the launcher process (widgets, not model-driven UI).
- **Design patterns:**
  - **Tool-call → Composable mapping** — Gemini `FunctionCall` name = component
    key, args = props; render via a registry `Map<String, @Composable (Props)->Unit>`.
  - **Whitelist / trusted-Composable registry** — the security boundary. The
    model ships a descriptor whose `type` names an *already-compiled* Composable;
    unknown types are dropped. The model may only select and parameterize
    approved components, never inject layout or code. Mandatory.
  - **Streaming / partial rendering** — collect output into `StateFlow`/`MutableState`,
    `collectAsStateWithLifecycle()`, and recomposition renders the growing tree
    incrementally with no manual diffing.
  - **Thinking / loading / error states** — sealed `UiState { Thinking,
    Streaming(partial), Complete, Error }`, `when`-branched in Compose.
  - **Attribution as a whitelisted component** — source chips, "AI-generated"
    badges, citations selected via the same registry.

The strongest assemblable "Compose `streamUI`" today = **Remote Compose (or
DivKit) as substrate + Gemini function calling as driver**. Nobody has shipped
the two glued into a product.

## Center 2 — on-device GenAI building blocks

- **ML Kit GenAI APIs** (on Gemini Nano via AICore; announced I/O 2025, now six
  features; one factory + one call, streaming as a callback overload):

  | Feature | Coordinate | Maturity |
  |---|---|---|
  | Summarization | `com.google.mlkit:genai-summarization:1.0.0-beta1` | Beta |
  | Rewriting | `com.google.mlkit:genai-rewriting:1.0.0-beta1` | Beta (input < 256 tokens) |
  | Proofreading | `com.google.mlkit:genai-proofreading:1.0.0-beta1` | Beta |
  | Image Description | `com.google.mlkit:genai-image-description:1.0.0-beta1` | Beta |
  | Prompt (custom/multimodal) | `com.google.mlkit:genai-prompt:1.0.0-beta3` | Beta |
  | Speech Recognition | genai speech | Alpha |

  Shape: `Summarization.getClient(SummarizerOptions)` → `Summarizer` →
  `runInference(request)` (`ListenableFuture`/coroutine, or streaming lambda) →
  `close()`. Each feature = shared Nano base model + a small feature-specific
  **LoRA adapter** pulled on demand. Availability/download UX is first-class:
  `checkFeatureStatus()` → `FeatureStatus { UNAVAILABLE, DOWNLOADABLE,
  DOWNLOADING, AVAILABLE }`; `downloadFeature(DownloadCallback)` with progress.

- **Gemini Nano + AICore** — AICore is the Android system service hosting
  on-device models (weights + LoRA, accelerated inference, safety, distribution)
  under Private Compute Core isolation. Access via ML Kit GenAI (recommended) or
  the experimental Google AI Edge SDK. Published perf: prefill 510 tok/s (Pixel 9
  Pro, nano-v2) / 940 tok/s (Pixel 10 Pro, nano-v3); ~11 tok/s decode (Pixel 9
  Pro). **Fragmentation is the defining limit:** primary docs name only Pixel 9/10;
  min API 26; not supported with an unlocked bootloader; the long tail has no
  Nano and must use cloud. Nano's parameter count, download size (MB), and
  first-token latency are *not published*.

- **Firebase AI Logic** (cloud + hybrid) — renamed from "Vertex AI in Firebase"
  (May 2025) once it gained a second backend (Gemini Developer API alongside
  Vertex AI). `com.google.firebase:firebase-ai`;
  `Firebase.ai(...).generativeModel(...)` → `generateContent()` /
  **`generateContentStream()`** (Kotlin `Flow` — the streaming primitive for
  Compose). **Hybrid on-device+cloud** is one experimental API (Android,
  announced 2026-04-17): `OnDeviceConfig` + `InferenceMode { PREFER_ON_DEVICE,
  ONLY_ON_DEVICE, PREFER_IN_CLOUD, ONLY_IN_CLOUD }`; `PREFER_*` auto-fall-back.
  On-device leg is single-turn, text-or-single-Bitmap only;
  `com.google.firebase:firebase-ai-ondevice:16.0.0-beta01`.

- **Interaction patterns forced:** streaming tokens into a recomposing `Text`;
  **two distinct waits** (one-time model/adapter download vs per-request
  inference); transparency/attribution (Google Play requires an AI-generated
  disclosure; design guidance in the People + AI Guidebook and Responsible
  GenAI Toolkit); graceful degradation via check → download → cloud fallback.

## Limitations (how we know)

- No first-party runtime-generation framework for Compose (absence in docs;
  A2UI's own roadmap).
- No standard schema — every substrate rolls its own; zero interop; lock-in.
- Security of model-proposed UI is the app's burden — whitelist mandatory
  (Remote Compose's `HostAction`/`PendingIntentAction` must be host-gated).
- State/recomposition with dynamic trees — no stable identity → lost
  scroll/focus/animation unless keys are synthesized (skydoves ships versioning
  for exactly this).
- Device fragmentation for on-device (nano-v3 ≈ Pixel-10-class).
- On-device constraints documented (Rewriting < 256 tokens; hybrid single-turn
  text/Bitmap; language limits).
- Maturity risk: ML Kit GenAI beta (speech alpha); Firebase hybrid + AI Edge SDK
  experimental; Remote Compose alpha; ML Kit GenAI is Android-only.

## What's easy today

ML Kit GenAI (one factory + call for summarize/rewrite/proofread/describe);
Firebase AI Logic `generateContentStream()` → Flow → incremental `Text`;
token-streaming UIs nearly free via Compose recomposition; a runtime-GenUI loop
assemblable from an SDUI substrate + Gemini function calling; one-line
on-device-then-cloud fallback via `InferenceMode.PREFER_ON_DEVICE` (experimental).

## Related

Distinct from the vault's Android systems pages ([Android image decoding](/wiki/android-image-decoding/),
[Android Camera2 pipeline and CameraX interop](/wiki/android-camera2-pipeline/), [Android MediaStore](/wiki/android-mediastore/)) — those cover the media
pipeline, not AI-driven or model-generated UI; no direct relation.
