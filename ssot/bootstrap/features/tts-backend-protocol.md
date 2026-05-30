---
id: tts-backend-protocol
name: TTS Backend Enum and Models
repo: FluidAudio
status: active
linked_features: [kokoro-tts-manager, magpie-tts-manager, pocket-tts-manager, cosyvoice3-tts-manager, kokoro-ane, style-tts2]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# TTS Backend Enum and Models

## TL;DR
`TtsBackend` is the public enum that names every shipped TTS backend (`kokoro`, `pocketTts`, `cosyvoice3`, `kokoroAne`, `styleTts2`). `TtsModels` is the Kokoro-specific CoreML model bundle (a dict keyed by variant) with the canonical download + warm-up path. `TtsConstants` holds Kokoro voice ids, sample rate, repo defaults, and the recommended voice.

## Motivation — why it exists

## Context

## What It Does
`TtsBackend` is a thin `Sendable` enum with one case per shipped backend manager. There is no protocol; each manager class is independent. The enum is used by the CLI and benchmark harness to switch between backends without conditionally importing every manager class. The docs inline on `.cosyvoice3` and `.styleTts2` mark them experimental/beta and explain the Flow CFM fp32 constraint and the StyleTTS2 fp32 HiFi-GAN decoder.

`TtsModels` is a struct holding a `[ModelNames.TTS.Variant: MLModel]` dictionary — strictly Kokoro models (single-graph). `init(models:)` is the manual injection path; `download(...)` is the canonical async path that selects target variants (5s vs 15s, or both), downloads via `DownloadUtils.loadModels(.kokoro, ...)` into `~/.cache/fluidaudio/Models/kokoro/`, then runs `warmUpModel(...)` on each variant. Warmup constructs zero `input_ids` + zero `ref_s` + zero `random_phases` and (for v2 macOS-only models with `source_noise`) a `[1, sampleRate*maxSeconds, 9]` Float16 noise tensor produced via vImage's `vImageConvert_PlanarFtoPlanar16F` (the code avoids touching Float16 directly because it isn't available in all build configs). Warm-up errors are logged but non-fatal. `optimizedPredictionOptions()` returns an `MLPredictionOptions` with empty `outputBackings` (reuse output buffers).

`TTSError` enumerates `downloadFailed`, `corruptedModel`, `modelNotFound`, `processingFailed`.

`TtsConstants` carries Kokoro-flavored constants: `recommendedVoice = "af_heart"`, an `availableVoices` list grouped by language with a callout that only American English (`af_*`, `am_*`) voices are QA'd, `delimiterCharacters` to strip from input, the whitespace-collapse regex, `audioSampleRate = 24_000`, the 5s-model tuning knobs (`kokoroFrameSamples`, `shortVariantGuardThresholdSeconds`, `shortVariantGuardFrameCount`, `shortSentenceMergeTokenThreshold`), and `defaultRepository = "FluidInference/kokoro-82m-coreml"`.

## Key Code
- [`Sources/FluidAudio/TTS/TtsBackend.swift:4`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/TtsBackend.swift#L4) — `public enum TtsBackend: Sendable` with `kokoro`, `pocketTts`, `cosyvoice3`, `kokoroAne`, `styleTts2`.
- [`Sources/FluidAudio/TTS/TtsModels.swift:6`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/TtsModels.swift#L6) — `public struct TtsModels: Sendable` storing `[Variant: MLModel]`.
- [`Sources/FluidAudio/TTS/TtsModels.swift:36`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/TtsModels.swift#L36) — `download(...)` canonical path: variant selection → `DownloadUtils.loadModels` → warm-up.
- [`Sources/FluidAudio/TTS/TtsModels.swift:90`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/TtsModels.swift#L90) — `getCacheDirectory()` platform-specific `~/.cache/fluidaudio` (macOS) or iOS caches dir.
- [`Sources/FluidAudio/TTS/TtsModels.swift:126`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/TtsModels.swift#L126) — `warmUpModel(...)`: zero-tensor prediction including vImage-based Float16 conversion for `source_noise`.
- [`Sources/FluidAudio/TTS/TtsModels.swift:235`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/TtsModels.swift#L235) — `TTSError` enum.
- [`Sources/FluidAudio/TTS/TtsConstants.swift:12`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/TtsConstants.swift#L12) — `recommendedVoice = "af_heart"`.
- [`Sources/FluidAudio/TTS/TtsConstants.swift:19`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/TtsConstants.swift#L19) — `availableVoices` table with per-language annotations.
- [`Sources/FluidAudio/TTS/TtsConstants.swift:50`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/TtsConstants.swift#L50) — `audioSampleRate = 24_000`.
- [`Sources/FluidAudio/TTS/TtsConstants.swift:59`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/TtsConstants.swift#L59) — `defaultRepository = "FluidInference/kokoro-82m-coreml"`.

## Edge Cases & Failure Modes
- `TtsModels.download` throws `TTSError.modelNotFound(name)` if any requested variant fails to materialize from the download dict.
- `getCacheDirectory()` throws `processingFailed("Failed to locate caches directory")` on iOS if `FileManager.default.urls(for: .cachesDirectory, ...)` returns empty.
- Warm-up failures are caught and logged via `logger.warning(...)`; the model is still returned to the caller.
- The v1/v2 split (presence of `source_noise` input) is detected at runtime via the model's input descriptions — there's no compile-time flag.
- Non-American-English voices in `availableVoices` are flagged "experimental, not tested" and are not covered by QA.
- [REVIEW: `TtsModels` is Kokoro-only despite the generic name; other backends use their own `*ModelStore` types. Confirm naming intent — should this be renamed `KokoroTtsModels` or kept as the Kokoro-default contract?]
- [REVIEW: warm-up fills a `[1, 24000*maxSeconds, 9]` Float16 noise tensor with `Float.random(in: -1...1)` — for 15s variant this is 24000×15×9 = 3.24M Float16 samples per warmup; check whether this dominates first-call latency.]
- [REVIEW: no `TtsBackend` → `ProtocolManager` mapping is documented in source — callers must `switch` over the enum themselves.]

## Performance / Concurrency Notes
- `TtsModels` is `Sendable` and the dict is read-only after construction.
- Warm-up runs in parallel across variants only if the caller awaits each in sequence (current code awaits in a `for` loop, so warm-up is serial).
- `optimizedPredictionOptions()` reuses output buffers via empty `outputBackings`.

## Test Coverage
- `Tests/FluidAudioTests/TTS/TTSManagerTests.swift` exercises `TtsModels` end-to-end via the manager.
- No dedicated `TtsBackendTests`/`TtsConstantsTests` in the read scope.

## Changelogs
