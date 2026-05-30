---
id: kokoro-tts-manager
name: Kokoro TTS Manager
repo: FluidAudio
status: active
linked_features: [kokoro-pipeline, kokoro-custom-lexicon, kokoro-ane, tts-backend-protocol, ssml-processor, multilingual-g2p]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Kokoro TTS Manager

## TL;DR
Public entry point for Kokoro 82M text-to-speech as a single CoreML graph (CPU+GPU). Default Kokoro path that ships under `FluidInference/kokoro-82m-coreml` and supports multi-voice packs, the long-text chunker, and custom lexicon overrides. Marked beta — only American English is QA'd.

## Motivation — why it exists

## Context

## What It Does
`KokoroTtsManager` is a `public final class` (non-actor) that owns one `TtsModels` bundle plus a `KokoroModelCache`, a `LexiconAssetManager`, and an optional `TtsCustomLexicon`. Construction is lightweight; callers must invoke `initialize()` (which downloads/compiles the model bundle via `TtsModels.download` and warms each variant) or `initialize(models:)` (when models are externally produced) before synthesizing. Initialization also calls `KokoroSynthesizer.loadSimplePhonemeDictionary()` and preloads requested voice embeddings via `TtsResourceDownloader.ensureVoiceEmbedding`.

Synthesis happens through `synthesize`, `synthesizeDetailed`, or `synthesizeToFile`. Each call runs the input through `TtsTextPreprocessor.preprocessDetailed` (handles SSML, expands numbers, etc.), sanitizes via `KokoroSynthesizer.sanitizeInput`, resolves the voice (falling back to a voice indexed by `speakerId % availableVoices.count`), ensures the voice's embedding is on disk, then enters three `@TaskLocal` scopes (`withLexiconAssets`, `withModelCache`, `withCustomLexicon`) so the underlying `KokoroSynthesizer` static API can pick up per-call dependencies without manager references. A `deEss` flag toggles AudioPostProcessor de-essing on the output.

Defaults: voice is `TtsConstants.recommendedVoice` ("af_heart"), `speakerId` 0, `computeUnits` `.all`. Docstring calls out using `.cpuAndGPU` on iOS 26+ to work around an ANE compiler regression ("Cannot retrieve vector from IRValue format int32"). Variant choice (5s vs 15s) is forwarded to `KokoroSynthesizer` via `variantPreference`, with the model cache lazy-loading whichever variants the bundle exposes.

Mutating helpers (`setDefaultVoice`, `setCustomLexicon`, `cleanup`) are synchronous since the class is not an actor; voice state and the `ensuredVoices` set are guarded only by the assumption that callers serialize their own usage. `cleanup()` drops the model bundle and resets init flags but does not affect the on-disk cache.

## Key Code
- [`Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift:38`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift#L38) — class declaration + stored deps (modelCache, lexiconAssets, customLexicon, voice state).
- [`Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift:109`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift#L109) — `initialize(models:preloadVoices:)`: registers preloaded models, prepares lexicon assets, preloads voice embeddings, loads phoneme dictionary, loads variants.
- [`Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift:124`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift#L124) — `initialize(preloadVoices:)` convenience: triggers `TtsModels.download`.
- [`Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift:148`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift#L148) — `synthesizeDetailed`: full pipeline with `@TaskLocal` injection of lexicon/model/custom-lexicon context.
- [`Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift:209`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift#L209) — `setDefaultVoice(_:speakerId:)`: re-ensures voice embedding before swapping.
- [`Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift:246`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift#L246) — `voiceName(for:)` fallback: indexes `TtsConstants.availableVoices` by `abs(speakerId) % count`.
- [`Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift:278`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift#L278) — `normalizeVoice`: trims whitespace, falls back to `recommendedVoice` when empty.

## Edge Cases & Failure Modes
- Calls to `synthesize` before `initialize` throw `TTSError.modelNotFound("Kokoro model not initialized")`.
- Empty / whitespace `voice` argument falls back to the speaker-indexed default voice.
- Voice embeddings are downloaded lazily on first use; network failure surfaces from `TtsResourceDownloader.ensureVoiceEmbedding`.
- `synthesizeToFile` deletes any existing file at `outputURL` before writing — destructive by design.
- `cleanup()` does not clear `customLexicon` or `defaultVoice`; only init state and the ensured-voice cache.
- [REVIEW: thread-safety — the class is not an actor; `ensuredVoices`, `defaultVoice`, `defaultSpeakerId`, `customLexicon`, `assetsReady`, `ttsModels`, `isInitialized` are all stored properties on a class with no documented serialization contract. Concurrent calls from multiple tasks could race.]
- [REVIEW: beta scope — only American English voices are QA'd; non-en voices are present in `availableVoices` but flagged "experimental, not tested" in `TtsConstants`.]

## Performance / Concurrency Notes
- Single-graph CPU+GPU path; for ANE-resident performance see `KokoroAneManager` (~3-11× RTFx tradeoff: single voice af_heart, ≤512 IPA tokens, no custom lexicon).
- `TtsModels.download` runs a per-variant warm-up prediction after compile so user-visible latency excludes first-call dispatch cost.
- Per-call `@TaskLocal` context injection (`KokoroSynthesizer.withModelCache`/`withLexiconAssets`/`withCustomLexicon`) lets the synthesizer remain a struct with static methods while dependencies flow through async boundaries.
- [REVIEW: docstring contradiction — class doc says "Use this when you want… mature, shipping default" while elsewhere TTS overall is documented as beta. Confirm intended positioning.]

## Test Coverage
- `Tests/FluidAudioTests/TTS/TTSManagerTests.swift` — manager-level smoke tests (init, synth, file output).
- `Tests/FluidAudioTests/TTS/TtsCustomLexiconTests.swift` — covers lexicon overrides through this manager.
- `Tests/FluidAudioTests/TTS/KokoroChunkerTests.swift`, `KokoroChunkerStemTests.swift`, `TtsTextPreprocessorAbbreviationTests.swift` — preprocessing path validated independently.

## Changelogs
