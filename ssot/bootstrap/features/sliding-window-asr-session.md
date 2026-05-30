---
id: sliding-window-asr-session
name: Sliding Window ASR Session
repo: FluidAudio
status: active
linked_features:
  - sliding-window-asr-manager
  - streaming-asr-manager
  - parakeet-model-variant
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Sliding Window ASR Session

## TL;DR
`SlidingWindowAsrSession` is the actor that owns shared model state for multiple concurrent ASR streams, keyed by `AudioSource` (microphone / system). It deduplicates model loads, distributes pre-loaded `AsrModels` to each spawned `SlidingWindowAsrManager`, and also exposes a separate `createEngine(variant:source:)` path for true streaming engines (`StreamingAsrManager`) like EOU / Nemotron.

## Context

## What It Does
The session lazily loads `AsrModels` on first use (via `AsrModels.downloadAndLoad()`) and caches the result. `createStream(source:config:)` returns an existing manager if one is already attached to that `AudioSource`, otherwise constructs a `SlidingWindowAsrManager`, calls `loadModels(_:)` with the shared models, and starts it. The companion `createEngine(variant:source:)` path uses `StreamingModelVariant.createManager()` so the same session can supervise mixed engine types per source. `cleanup()` cancels TDT streams, resets streaming engines, drops all references, and clears the cached models. `SlidingWindowAsrError` (defined at the bottom) carries the structured error cases used by both the session and `SlidingWindowAsrManager` (`modelsNotLoaded`, `audioBufferProcessingFailed`, `modelProcessingFailed`, `bufferOverflow`, etc.).

## Key Code
- [`SlidingWindowAsrSession.swift:6`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrSession.swift#L6) — actor declaration with `[AudioSource: SlidingWindowAsrManager]` and `[AudioSource: any StreamingAsrManager]` dictionaries.
- [`SlidingWindowAsrSession.swift:34`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrSession.swift#L34) — `createStream(source:config:)` deduplicated TDT stream factory.
- [`SlidingWindowAsrSession.swift:101`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrSession.swift#L101) — `createEngine(variant:source:)` dispatches to `StreamingModelVariant.createManager()`.
- [`SlidingWindowAsrSession.swift:134`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrSession.swift#L134) — `cleanup()` cancels TDT, resets engines, drops models.
- [`SlidingWindowAsrSession.swift:159`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrSession.swift#L159) — `SlidingWindowAsrError` cases.

## Edge Cases & Failure Modes
- Asking for a stream/engine that already exists for the same `AudioSource` returns the existing instance with a warning — does not throw.
- `cleanup()` ignores errors thrown by `engine.reset()` (uses `try?`).
- Models are dropped on `cleanup()`; a subsequent `createStream` triggers a fresh `downloadAndLoad()` cycle.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Parakeet/SlidingWindow/SlidingWindowAsrSessionTests.swift`.

## Changelogs
