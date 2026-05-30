---
id: streaming-asr-utils
name: Streaming ASR Utilities
repo: FluidAudio
status: active
linked_features:
  - eou-detector
  - nemotron-streaming
  - parakeet-tokenizer
  - encoder-cache-manager
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Streaming ASR Utilities

## TL;DR
`StreamingAsrUtils` is a static collection of small helpers shared by the EOU and Nemotron streaming managers: end-of-stream padding into a final chunk, tokenizer-based detokenization, shared-state reset, and audio buffer append-with-resample. Pure functions designed to be unit-tested independently of the actor managers.

## Context

## What It Does
`processRemainingAudio(audioBuffer:chunkSamples:processChunk:)` zero-pads the residual buffer to `chunkSamples` (if needed), passes the first `chunkSamples` samples through the caller's async closure, clears the buffer, and returns whether work was done. `decodeTokens(_:using:)` forwards to `Tokenizer.decode(ids:)`. `resetSharedState(audioBuffer:accumulatedTokenIds:processedChunks:)` zeros the three pieces of state that every streaming manager needs to clear on `reset()`. `appendAudio(_:using:to:)` runs `AudioConverter.resampleBuffer(_:)` and appends the float samples into the destination buffer.

## Key Code
- [`StreamingAsrUtils.swift:14`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrUtils.swift#L14) — `processRemainingAudio` end-of-stream padding helper.
- [`StreamingAsrUtils.swift:40`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrUtils.swift#L40) — `decodeTokens` thin wrapper around `Tokenizer.decode`.
- [`StreamingAsrUtils.swift:49`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrUtils.swift#L49) — `resetSharedState` clears buffer / tokens / chunk counter.
- [`StreamingAsrUtils.swift:64`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrUtils.swift#L64) — `appendAudio` resample-and-append.

## Edge Cases & Failure Modes
- `processRemainingAudio` returns `false` (without throwing) when the buffer is empty.
- The padding strategy is right-padding with zeros — fine for the final chunk because trailing silence does not move the decoder state.
- `processRemainingAudio` uses `prefix(chunkSamples)` so if the buffer is somehow longer than `chunkSamples`, the trailing tail is silently dropped — callers are expected to call this only after consuming complete chunks.

## Test Coverage
- Indirectly via streaming manager tests; the helpers themselves are small enough that no dedicated test file exists.

## Changelogs
