---
id: encoder-cache-manager
name: Encoder Cache Manager
repo: FluidAudio
status: active
linked_features:
  - eou-detector
  - nemotron-streaming
  - streaming-asr-utils
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Encoder Cache Manager

## TL;DR
`EncoderCacheManager` consolidates the boilerplate for streaming encoder cache state — creating zero-initialized `cache_channel`, `cache_time`, and `cache_len` `MLMultiArray`s and extracting the matching `_out` arrays from a model prediction. Used by both EOU and Nemotron streaming managers so the cache-init / cache-rotate code lives in one place.

## Context

## What It Does
Three static helpers: `createInitialCaches(config:)` allocates the three caches with caller-supplied shapes (float32 for channel/time, int32 for len) and zero-fills them; `extractCachesFromOutput(_:channelKey:timeKey:lenKey:)` reads them back from an `MLFeatureProvider` using configurable feature names (default `cache_channel_out` / `cache_time_out` / `cache_len_out`); `createZeroArray(shape:dataType:)` is a generic zero-array helper used for the mel cache and decoder states.

## Key Code
- [`EncoderCacheManager.swift:23`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/EncoderCacheManager.swift#L23) — `createInitialCaches(config:)` returns the channel/time/len tuple.
- [`EncoderCacheManager.swift:58`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/EncoderCacheManager.swift#L58) — `extractCachesFromOutput(_:)` pulls updated caches from a CoreML output.
- [`EncoderCacheManager.swift:72`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/EncoderCacheManager.swift#L72) — `createZeroArray(shape:dataType:)` general-purpose zero allocator.

## Edge Cases & Failure Modes
- All allocation paths throw if `MLMultiArray.init(shape:dataType:)` fails (e.g. shape contains a 0).
- `extractCachesFromOutput` returns `nil` per slot when the model output does not expose that key — callers must defensively handle this.
- `reset(to: 0)` is an `MLMultiArray` extension defined elsewhere in the module.

## Test Coverage
- Exercised indirectly via `Tests/FluidAudioTests/ASR/Parakeet/Streaming/StreamingNemotronAsrManagerTests.swift` and `Tests/FluidAudioTests/ASR/Parakeet/Streaming/StreamingAsrManagerTests.swift`.

## Changelogs
