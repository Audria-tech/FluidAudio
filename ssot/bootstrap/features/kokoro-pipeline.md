---
id: kokoro-pipeline
name: Kokoro Pipeline (Preprocess + Synthesize + Postprocess)
repo: FluidAudio
status: active
linked_features: [kokoro-tts-manager, kokoro-custom-lexicon, multilingual-g2p, ssml-processor]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Kokoro Pipeline (Preprocess + Synthesize + Postprocess)

## TL;DR
The internal Kokoro synthesis pipeline split across `Preprocess/`, `Synthesize/`, and `Postprocess/` subfolders. `KokoroSynthesizer` is the static-method orchestrator using `@TaskLocal` to receive model-cache / lexicon-asset / custom-lexicon context from the public manager.

## Motivation — why it exists

## Context

## What It Does
The pipeline runs in three phases:

**Preprocess** — `TtsTextPreprocessor` runs SSML → cleanup → phonetic-override extraction; `PhonemeMapper` walks IPA tokens to model-vocab ids; `KokoroChunker` segments into capacity-aware chunks (sentence-aware via NaturalLanguage); `KokoroModelCache` is an actor that loads variants on demand; `KokoroSynthesizer+LexiconCache` keeps per-asset caches in scope across calls.

**Synthesize** — `KokoroSynthesizer` is a struct of static methods plus `@TaskLocal` storage (`Context.modelCache`, `Context.lexiconAssets`, `Context.customLexicon`). `KokoroSynthesizer+VoiceEmbeddings` resolves voice packs from `.json`. `KokoroSynthesizer+ModelUtils` handles model picking and warm-up shape inference. `KokoroSynthesizer+Memory` includes pools/inputs reused across calls (`MultiArrayPool`).

**Postprocess** — `KokoroSynthesizer+Types` carries the `SynthesisResult` shape (`audio: Data`, framing/timing info).

## Key Code
- [`Sources/FluidAudio/TTS/Kokoro/Pipeline/Synthesize/KokoroSynthesizer.swift:11`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/Pipeline/Synthesize/KokoroSynthesizer.swift#L11) — top of `struct KokoroSynthesizer` + `@TaskLocal` Context.
- [`Sources/FluidAudio/TTS/Kokoro/Pipeline/Preprocess/KokoroChunker.swift:20`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/Pipeline/Preprocess/KokoroChunker.swift#L20) — `enum KokoroChunker`.
- [`Sources/FluidAudio/TTS/Kokoro/Pipeline/Preprocess/KokoroModelCache.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/Pipeline/Preprocess/KokoroModelCache.swift) — variant-keyed lazy model store.
- [`Sources/FluidAudio/TTS/Kokoro/Pipeline/Preprocess/TtsTextPreprocessor.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/Pipeline/Preprocess/TtsTextPreprocessor.swift) — SSML + abbreviation expansion + word indexing.
- [`Sources/FluidAudio/TTS/Kokoro/Pipeline/Preprocess/PhonemeMapper.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/Pipeline/Preprocess/PhonemeMapper.swift) — IPA token → model vocab id.

## Edge Cases & Failure Modes
- All sub-components throw `TTSError.processingFailed` or `TTSError.modelNotFound` upward; manager re-surfaces.
- `@TaskLocal` context must be installed by the manager (`withModelCache`, `withLexiconAssets`, `withCustomLexicon`) — calling synthesizer statics outside that scope fails at runtime.
- [REVIEW: chunker boundary handling — sentence-aware via NaturalLanguage may behave differently on non-en input even though only en is QA'd.]

## Test Coverage
- `Tests/FluidAudioTests/TTS/KokoroChunkerTests.swift`, `KokoroChunkerStemTests.swift`, `TtsTextPreprocessorAbbreviationTests.swift`.

## Changelogs
