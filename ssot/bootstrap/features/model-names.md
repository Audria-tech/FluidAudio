---
id: model-names
name: Model Names
repo: FluidAudio
status: active
linked_features: [download-utils]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Model Names

## TL;DR
Single source of truth for every HuggingFace repository identifier, on-disk file name, and per-pipeline "required file set" in FluidAudio. `Repo` is the enum of remote repos; `ModelNames` is the namespaced catalog of compiled `.mlmodelc` filenames and auxiliary JSON/BIN assets each model family ships.

## Motivation — why it exists

## Context

## What It Does
`Repo` enumerates every HuggingFace repo FluidAudio knows about (Parakeet V2/V3, CTC variants, EOU streaming variants at 80/160/320/1280 ms hops, Nemotron streaming, Diarizer, Sortformer, LS-EEND, Kokoro, KokoroAne, PocketTTS, Qwen3 ASR, CharsiuG2P, CosyVoice3, Cohere Transcribe, Magpie TTS, StyleTTS2). The raw value is the HuggingFace path (`FluidInference/<slug>` or `FluidInference/<slug>/<subpath>`); `remotePath`, `name`, `subPath`, and `folderName` derive the API path, slug, in-repo subdirectory, and local cache directory respectively. The four derivations are not always trivially related (e.g. `Repo.kokoro.folderName` is the bare string `"kokoro"`, not `"kokoro-82m-coreml"`) so the per-case switches matter.

`ModelNames` exposes one namespace per model family — `Diarizer`, `OfflineDiarizer`, `ASR`, `CTC`, `CTCZhCn`, `TDTJa`, `VAD`, `ParakeetEOU`, `NemotronStreaming`, `Sortformer`, `LSEEND`, `Qwen3ASR`, `PocketTTS`, `CosyVoice3`, `Magpie`, `StyleTTS2`, `MultilingualG2P`, `CohereTranscribe`, `G2P`, `TTS` (Kokoro v1), `KokoroAne`. Each namespace publishes per-stage bundle names (`preprocessorFile`, `encoderFile`, …) plus a `requiredModels: Set<String>` used by the downloader to decide what to fetch. Some namespaces parametrise the required set: ASR by `ParakeetEncoderPrecision`, PocketTTS by `PocketTtsPrecision`, Sortformer and LS-EEND by enum `Variant` + optional `StepSize`, TTS (Kokoro) by `Variant` (5s vs 15s).

The bottom of the file is the `getRequiredModelNames(for:variant:)` dispatcher. `DownloadUtils.loadModels` calls this with the `Repo` and an optional `variant` string (e.g. `"int8"` for V3 encoder precision, `"offline"` for the VBx diarizer, `"g2p-only"` sentinel for KokoroAne reusing the Kokoro G2P bundles, or a raw mlmodelc filename for single-variant Sortformer/LS-EEND fetches). The result drives both download filtering and post-download verification.

Notable cross-cutting carve-outs: (a) the V3 `JointDecisionv3.mlmodelc` is a different file from `JointDecision.mlmodelc` (V3 exposes top-K outputs for `TokenLanguageFilter`); (b) the 110m hybrid uses the "fused" required set (`requiredModelsFused`) because its preprocessor includes the encoder; (c) the Japanese TDT repo is a hybrid — the preprocessor/encoder come from a CTC-trained model, paired with TDT `Decoderv2`/`Jointerv2`; (d) StyleTTS2 ships precompiled `.mlmodelc` under `compiled/` (not the repo root) to skip the cold-start `anecompilerservice` hit.

## Key Code
- [`Sources/FluidAudio/ModelNames.swift:4`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L4) — `Repo` enum (raw value = HF path); `CaseIterable`, `Sendable`
- [`Sources/FluidAudio/ModelNames.swift:109`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L109) — `remotePath` — collapses sub-pathed cases to their parent repo
- [`Sources/FluidAudio/ModelNames.swift:137`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L137) — `subPath` — in-repo subdirectory (e.g. `160ms`, `optimized/ami`)
- [`Sources/FluidAudio/ModelNames.swift:175`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L175) — `folderName` — local cache directory (often custom, not derived from `name`)
- [`Sources/FluidAudio/ModelNames.swift:229`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L229) — `ParakeetEncoderPrecision` (int8 / int4) — selects V3 encoder file
- [`Sources/FluidAudio/ModelNames.swift:289`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L289) — `ModelNames.ASR` — V2 / V3 / fused / Japanese set splits
- [`Sources/FluidAudio/ModelNames.swift:305`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L305) — `jointV3File` — V3-only top-K joint for `TokenLanguageFilter`
- [`Sources/FluidAudio/ModelNames.swift:330`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L330) — `requiredModelsFused` — 110m hybrid (preprocessor contains encoder)
- [`Sources/FluidAudio/ModelNames.swift:444`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L444) — `NemotronStreaming` — encoder lives in subdir as `encoder_int8.mlmodelc`
- [`Sources/FluidAudio/ModelNames.swift:471`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L471) — `Sortformer.Variant` (6 variants) + `bundle(for:)` + variant-config compatibility
- [`Sources/FluidAudio/ModelNames.swift:552`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L552) — `LSEEND.Variant` × `StepSize` (4×5 = 20 bundles in `requiredModels`)
- [`Sources/FluidAudio/ModelNames.swift:631`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L631) — `Qwen3ASR` — 2-model vs 3-model pipelines (`requiredModels` vs `requiredModelsFull`)
- [`Sources/FluidAudio/ModelNames.swift:684`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L684) — `PocketTTS.flowlmStepFile(precision:)` — only FlowLM has int8 variant
- [`Sources/FluidAudio/ModelNames.swift:796`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L796) — `StyleTTS2` — bucket tables + `compiled/` prefix on every file
- [`Sources/FluidAudio/ModelNames.swift:888`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L888) — `CohereTranscribe` — auto-detect v1/v2 decoder via input shape
- [`Sources/FluidAudio/ModelNames.swift:935`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L935) — `TTS.Variant` — Kokoro `fiveSecond` / `fifteenSecond` (forces v1, v2 has `source_noise` issues)
- [`Sources/FluidAudio/ModelNames.swift:1006`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L1006) — `getRequiredModelNames(for:variant:)` — dispatcher used by `DownloadUtils.loadModels`

## Edge Cases & Failure Modes
- `variant` is a stringly-typed parameter on `getRequiredModelNames` — `parakeetV3` parses it as a `ParakeetEncoderPrecision`, `diarizer` matches `"offline"`, `kokoro` matches `"g2p-only"` or treats it as a single TTS file, `sortformer` / `lseendAmi`-`Dihard3` treat it as a single bundle name. [REVIEW: stringly-typed variant dispatcher — wrong sentinel silently treats variant as a single model filename]
- `Repo.kokoro` always fetches Kokoro v1 (5s + 15s) **plus** G2P **plus** MultilingualG2P bundles, even when a single variant is requested.
- `LSEEND.Variant` × `StepSize.requiredModels` enumerates 20 bundles per repo — fetching the full required set pulls every step size, not just the chosen one.
- `Sortformer.bundle(for:)` returns `nil` if the config has no `modelVariant`, and a debug `assert` (not a release-time check) verifies compatibility. [REVIEW: Sortformer compatibility check is `assert` only — release builds silently load incompatible variant]
- StyleTTS2 `requiredModels` does **not** include the `.mlpackage` files at the repo root (intentional — only the precompiled `compiled/*.mlmodelc` artifacts are loaded).
- `CTCZhCn.requiredModels` ships both `Encoder-v2-int8.mlmodelc` and `Encoder-v1-fp32.mlmodelc` so callers can pick at runtime; that doubles the download payload.
- `parakeetJa` reuses the CTC-trained preprocessor + encoder paired with TDT `Decoderv2`/`Jointerv2`. The required set lives in `ModelNames.TDTJa`, not `ASR`.
- `Qwen3ASR.requiredModels` (3-model) is named but the dispatcher actually returns `requiredModelsFull` (2-model). Callers wanting the 3-model pipeline must opt in explicitly. [REVIEW: dispatcher routes both `qwen3Asr` cases to `requiredModelsFull` — `requiredModels` is unreachable via `loadModels` default]
- `Repo.cohereTranscribeCoreml` raw value embeds `/q8` but `remotePath` strips it back to the repo root — easy to mis-extract if someone reads `rawValue` directly.

## Performance / Concurrency Notes
Pure-value enum, no runtime cost beyond the switch lookups. All static lets are computed at first access.

## Test Coverage
Indirectly covered by every download/load test (e.g. `Tests/FluidAudioTests/CI/BasicInitializationTests.swift`); there is no dedicated unit test file for `ModelNames` itself.

## Changelogs
