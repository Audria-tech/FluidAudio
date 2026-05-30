---
id: pocket-tts-pipeline
name: PocketTTS Pipeline
repo: FluidAudio
status: active
linked_features: [pocket-tts-manager, pocket-tts-tokenizer]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# PocketTTS Pipeline

## TL;DR
PocketTTS internal pipeline files: `PocketTtsSynthesizer` (+Flow, +KVCache, +Mimi, +Types extensions), `PocketTtsModelStore`, `PocketTtsSession`, `PocketTtsVoiceCloner`, and key-name tables for flow-LM and Mimi layers. Implements flow-matching language-model AR generation at 80 ms / 1920-sample / 24 kHz granularity.

## Motivation — why it exists

## Context

## What It Does
`PocketTtsSynthesizer` is a struct with static methods plus a `@TaskLocal` model-store context (`withModelStore`/`currentModelStore`). Each public manager call wraps work in the task-local scope. Pipeline per chunk: text → tokenize → embed → prefill KV → AR generate → flow decode → Mimi decode → WAV/frames. Each step produces an 80 ms (1920-sample) frame at 24 kHz. Long text is split into sentence-based chunks (≤50 tokens each) to stay within the 512-position KV cache.

`PocketTtsSynthesizer+Flow` runs the flow-matching transformer step. `+KVCache` keeps decoder state across AR steps. `+Mimi` invokes the Mimi neural codec decoder (24 kHz output). `+Types` holds `SynthesisResult`, `AudioFrame`, etc.

`PocketTtsModelStore` is an actor that downloads + caches Mimi encoder/decoder, flow-LM step model (`.fp16` or `.int8` via `flowlm_stepv2`), voice constants, and Mimi key tables.

`PocketTtsSession` keeps the voice prefill warm so subsequent `enqueue` calls only pay text prefill cost. Mimi decoder state persists across utterances.

`PocketTtsVoiceCloner` handles serialization of `PocketTtsVoiceData` (cloned voice conditioning) to/from disk.

`PocketTtsLayerKeys` and `PocketTtsMimiKeys` are key-name lookup tables for binding CoreML model inputs.

## Key Code
- [`Sources/FluidAudio/TTS/PocketTTS/Pipeline/PocketTtsSynthesizer.swift:13`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/Pipeline/PocketTtsSynthesizer.swift#L13) — `struct PocketTtsSynthesizer` + `@TaskLocal` context.
- [`Sources/FluidAudio/TTS/PocketTTS/Pipeline/PocketTtsSession.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/Pipeline/PocketTtsSession.swift) — persistent warm-cache session.
- [`Sources/FluidAudio/TTS/PocketTTS/Pipeline/PocketTtsVoiceCloner.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/Pipeline/PocketTtsVoiceCloner.swift) — voice data serialization.
- [`Sources/FluidAudio/TTS/PocketTTS/Pipeline/PocketTtsSynthesizer+Mimi.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/Pipeline/PocketTtsSynthesizer%2BMimi.swift) — Mimi codec decode.
- [`Sources/FluidAudio/TTS/PocketTTS/Pipeline/PocketTtsSynthesizer+KVCache.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/Pipeline/PocketTtsSynthesizer%2BKVCache.swift) — decoder KV cache.

## Edge Cases & Failure Modes
- KV cache capped at 512 positions; chunker splits to fit.
- Statics throw `PocketTTSError.processingFailed` if invoked outside the `withModelStore(...)` scope.
- `mimi_encoder` model is optional — voice cloning gated on its presence.
- [REVIEW: session continuity — Mimi state persists across utterances; verify behavior when a session is cancelled mid-utterance.]

## Test Coverage
- `Tests/FluidAudioTests/TTS/PocketTTS/PocketTtsSessionTests.swift`, `PocketTtsStreamingTests.swift`.

## Changelogs
