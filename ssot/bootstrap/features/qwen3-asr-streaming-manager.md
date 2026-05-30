---
id: qwen3-asr-streaming-manager
name: Qwen3-ASR Streaming Manager
repo: FluidAudio
status: active
linked_features:
  - qwen3-asr-manager
  - qwen3-asr-config
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Qwen3-ASR Streaming Manager

## TL;DR
`Qwen3StreamingManager` is a thin actor that adapts the (offline) `Qwen3AsrManager.transcribe(...)` API into a sliding-window streaming experience: feed audio chunks, get incremental transcripts roughly every `chunkSeconds`, drop the oldest samples when total exceeds `maxAudioSeconds`. The manager re-transcribes the entire accumulated buffer each tick — there is no real KV-cache continuation across chunks.

## Context

## What It Does
`Qwen3StreamingConfig` carries four knobs: `minAudioSeconds` (don't transcribe before this is buffered), `chunkSeconds` (re-transcribe cadence), `maxAudioSeconds` (rolling buffer cap), and an optional `Qwen3AsrConfig.Language` hint. `addAudio(_:)` appends samples, increments `samplesSinceLastTranscribe`, and — once both gates are met (`hasMinAudio && chunkReady`) — trims the buffer to `maxAudioSeconds`, calls `asrManager.transcribe(audioSamples:language:maxNewTokens: 512)`, resets the counter, and returns a `Qwen3StreamingResult(transcript, audioDuration, isFinal: false)`. `finish()` runs a last transcription if buffered audio is unprocessed, returns `isFinal: true`, and `reset()`s the state.

## Key Code
- [`Qwen3StreamingManager.swift:8`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3StreamingManager.swift#L8) — `Qwen3StreamingConfig` struct + defaults.
- [`Qwen3StreamingManager.swift:39`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3StreamingManager.swift#L39) — `Qwen3StreamingResult` shape.
- [`Qwen3StreamingManager.swift:73`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3StreamingManager.swift#L73) — `actor Qwen3StreamingManager`.
- [`Qwen3StreamingManager.swift:105`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3StreamingManager.swift#L105) — `addAudio(_:)` with min-audio + chunk-ready gating.
- [`Qwen3StreamingManager.swift:152`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3StreamingManager.swift#L152) — `finish()` end-of-stream.

## Edge Cases & Failure Modes
- Each tick re-runs the *entire* prompt + decode on the full buffer — wall-time grows roughly linearly with `audioDuration` until `maxAudioSeconds` clamps it.
- When the buffer is trimmed, the dropped audio is gone — there is no continuation cache. Words near the buffer head can disappear from the transcript on the next tick.
- `samplesSinceLastTranscribe` is the *only* trigger for re-transcription; if the caller never calls `addAudio` with enough new samples, no update fires until `finish()`.
- `maxNewTokens: 512` is hard-coded — long utterances may truncate.
- [REVIEW: the comment says "sliding window" but in practice the manager re-transcribes from a single growing buffer rather than maintaining a continuation window — wording can mislead.]

## Test Coverage
- No dedicated test file; exercised through `Tests/FluidAudioTests/ASR/Qwen3/Qwen3AsrConfigTests.swift` indirectly.

## Changelogs
