---
id: magpie-tts-manager
name: Magpie TTS Manager
repo: FluidAudio
status: active
linked_features: [magpie-local-transformer, magpie-pipeline, magpie-types, magpie-constants, tts-backend-protocol]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Magpie TTS Manager

## TL;DR
Public actor facade for NVIDIA Magpie TTS Multilingual 357M — an encoder-decoder transformer with NanoCodec vocoder producing 22 kHz audio. Supports 8 built-in languages, 5 built-in speakers, and a `synthesizeStream(...)` chunked variant for time-to-first-audio. Currently experimental and slow on Apple Silicon.

## Motivation — why it exists

## Context

## What It Does
`MagpieTtsManager` is a `public actor` that wraps three lazily-loaded components: a `MagpieModelStore` (downloads + holds the four CoreML graphs `text_encoder`, `decoder_prefill`, `decoder_step`, `nanocodec_decoder`), a `MagpieTokenizer` (per-language id mapping rooted at `repoDir/tokenizer_data/<lang>/`), and a `MagpieSynthesizer` (orchestrates the AR loop). `init(...)` only captures config; `downloadAndCreate(...)` is the convenience factory that returns a fully initialized manager. Default `computeUnits` is `.cpuAndNeuralEngine`.

`initialize()` runs the model download, materializes the tokenizer for `preferredLanguages`, then invokes `synthesizer.warmup()` to pay first-dispatch cost on `text_encoder` / `decoder_step` / `nanocodec_decoder` before the user-visible synthesize call. Warmup failures are logged as non-fatal. `prepareLanguage(_:)` fetches tokenizer data for a language not declared at init time.

`synthesize(text:speaker:language:options:)` runs the full pipeline: tokenize → text_encoder → prefill → 8-codebook AR loop (≤ 500 steps, gated by `minFrames`) → NanoCodec decode → optional peak normalize. `synthesizeStream(...)` yields one `MagpieAudioChunk` per chunker chunk as soon as its NanoCodec decode finishes; first chunk is intentionally short (~50 codec frames ≈ 2.3 s) to minimize time-to-first-audio, later chunks pack at normal capacity. `peakNormalize` is force-disabled in streaming mode. There is also a phoneme-mode `synthesize(phonemes:speaker:options:)` that bypasses the text frontend.

When `options.allowIpaOverride` is true (default), `|...|` regions in the text are tokenized directly as space-separated IPA against the language's token2id map (see `MagpieIpaOverride`); outside those regions, normal language tokenization runs.

## Key Code
- [`Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift:34`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift#L34) — `public actor MagpieTtsManager` declaration + private store/tokenizer/synthesizer.
- [`Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift:61`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift#L61) — `downloadAndCreate(...)` factory.
- [`Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift:75`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift#L75) — `initialize()`: download → store → tokenizer → synthesizer → warmup.
- [`Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift:118`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift#L118) — `prepareLanguage(_:)` on-demand tokenizer fetch.
- [`Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift:133`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift#L133) — `synthesize(text:speaker:language:options:)` — single-shot.
- [`Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift:160`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift#L160) — `synthesizeStream(...)` returns `AsyncThrowingStream<MagpieAudioChunk, Error>`.
- [`Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift:174`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift#L174) — phoneme-mode synthesis bypassing the text frontend.
- [`Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift:186`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieTtsManager.swift#L186) — `cleanup()` unloads the model store.

## Edge Cases & Failure Modes
- Calls to any `synthesize*` before `initialize()` throw `MagpieError.notInitialized`.
- Japanese (`ja`) is intentionally omitted in this Swift port; it requires OpenJTalk + MeCab dictionary (~102 MB) — see `MagpieTypes.swift`.
- Speaker 0 has a single trailing-word artifact attributable to fp16 sampler-trajectory drift (not a structural bug, per docstring).
- Cancelling the consuming task on `synthesizeStream` cancels in-flight synthesis cleanly.
- Streaming variant force-disables `peakNormalize` (cannot be applied without buffering the full utterance).
- [REVIEW: performance — Magpie at 0.04 RTFx, 25× slower than realtime. Docstring quotes "RTFx ≈ 0.04" warm, with ~30s first-synth cold; do not use in latency-sensitive paths. Per README, recommended alternatives: Kokoro (~20× RTFx, parallel) or PocketTTS (~1.5–2× RTFx, streaming Mimi).]
- [REVIEW: doc inconsistency — class doc cites "RTFx ≈ 0.04" but `initialize()` log line says "agg-RTFx ~0.41× on M2 for the MiniMax-English corpus". Two different numbers in the same file.]

## Performance / Concurrency Notes
- First synth on a fresh process is dominated by CoreML model load + first-call ANE compile (~30 s); warm synths run at ~96 s wall for an 8-word English sentence on M-series.
- Actor isolation provides serialization; the model store is also actor-isolated.
- Warmup forces `topK=1`, sets `minFrames > maxSteps` so EOS cannot fire — forces all decoder_step calls to specialize.
- The local transformer is implemented in Swift (BLAS) rather than CoreML so the 8-codebook AR sampling stays cache-resident; see `magpie-local-transformer.md`.

## Test Coverage
- `Tests/FluidAudioTests/TTS/Magpie/MagpieConstantsTests.swift`
- `Tests/FluidAudioTests/TTS/Magpie/MagpieIpaOverrideTests.swift`
- `Tests/FluidAudioTests/TTS/Magpie/MagpieKvCacheTests.swift`
- `Tests/FluidAudioTests/TTS/Magpie/MagpieMT19937Tests.swift`
- `Tests/FluidAudioTests/TTS/Magpie/MagpieNpyReaderTests.swift`

## Changelogs
