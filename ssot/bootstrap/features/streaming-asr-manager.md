---
id: streaming-asr-manager
name: Streaming ASR Manager Protocol
repo: FluidAudio
status: active
linked_features:
  - eou-detector
  - nemotron-streaming
  - parakeet-model-variant
  - sliding-window-asr-manager
  - sliding-window-asr-session
  - punctuation-commit-layer
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Streaming ASR Manager Protocol

## TL;DR
`StreamingAsrManager` is the actor-constrained protocol that unifies all *true* streaming Parakeet engines — those with cache-aware encoders that hold state across chunks — behind one polymorphic interface. `StreamingEouAsrManager` and `StreamingNemotronAsrManager` both conform to it so callers can instantiate either through `StreamingModelVariant.createManager()` and treat them identically. The TDT sliding-window manager intentionally does **not** conform, because its encoder is offline and its windowing model is structurally different.

## Motivation — why it exists

## Context

## What It Does
Although the file is only ~60 lines long, it is one of the most load-bearing abstractions in the ASR layer. The protocol enforces an `Actor` constraint so the conforming types' mutable buffers (audio buffer, encoder cache, decoder cache, accumulated tokens, callbacks) are automatically isolated. The required surface is intentionally narrow: `loadModels()` (idempotent, downloads + loads), `appendAudio(_:)` (synchronous push into the engine's internal buffer with implicit 16 kHz resampling), `processBufferedAudio()` (advances complete chunks through the encoder + decoder), `finish()` (flush + return final transcript), `reset()` (keep models, drop session state), `cleanup()` (release models entirely), `setPartialTranscriptCallback(_:)` (live partial updates), and `getPartialTranscript()` (snapshot).

The split between `appendAudio` and `processBufferedAudio` is deliberate: callers can decouple capture cadence from inference cadence — feed many small buffers, then drive inference at the engine's natural chunk boundary (160ms / 320ms / 560ms / 1120ms / 1280ms depending on variant). The protocol also documents that `displayName` is human-readable for logging/telemetry.

The companion factory lives in `ParakeetModelVariant.swift` (`StreamingModelVariant.createManager()`) which dispatches by `engineFamily` to either `StreamingEouAsrManager(configuration:chunkSize:)` or `StreamingNemotronAsrManager(configuration:requestedChunkSize:)`, returning `any StreamingAsrManager`. Higher-level orchestration (e.g. `SlidingWindowAsrSession.createEngine`) holds engines as `any StreamingAsrManager` to remain engine-agnostic.

## Key Code
- [`StreamingAsrManager.swift:20`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrManager.swift#L20) — protocol declaration with `Actor` constraint and the explanatory note that `SlidingWindowAsrManager` deliberately does not conform.
- [`StreamingAsrManager.swift:27`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrManager.swift#L27) — `loadModels()` idempotency contract.
- [`StreamingAsrManager.swift:34`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrManager.swift#L34) — `appendAudio(_:)` resampling contract.
- [`StreamingAsrManager.swift:40`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrManager.swift#L40) — `processBufferedAudio()` chunk-based execution.
- [`StreamingAsrManager.swift:43`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrManager.swift#L43) — `finish()` end-of-stream contract.
- [`StreamingAsrManager.swift:55`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrManager.swift#L55) — `setPartialTranscriptCallback(_:)` — `@Sendable (String) -> Void` for live UI updates.
- [`ParakeetModelVariant.swift:88`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/ParakeetModelVariant.swift#L88) — factory dispatch into conforming engines.
- [`SlidingWindowAsrSession.swift:101`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrSession.swift#L101) — `createEngine(variant:source:)` consumes the abstraction.

## Edge Cases & Failure Modes
- The protocol does not surface backpressure: `appendAudio` is non-async/throws-only and the engine's internal buffer can grow if `processBufferedAudio()` is not called. Conforming actors should bound their buffers.
- `setPartialTranscriptCallback` replaces (does not add to) the callback; conforming types may invoke it from inside their actor — callers must avoid re-entrant calls back into the manager.
- `cleanup()` makes the manager unusable until `loadModels()` is called again — there is no soft-cleanup that keeps weights.
- `finish()` returns the full transcript as a single `String`; callers needing token-level structure must use the partial callback during streaming.
- Conforming types are free to interpret `reset()` differently — both current implementations clear audio buffer, accumulated tokens, encoder cache, and decoder LSTM state.

## Performance / Concurrency Notes
- The `Actor` constraint means all conforming types implicitly serialize calls; consumers using `any StreamingAsrManager` cross suspension points at every method call.
- Polymorphic dispatch through `any StreamingAsrManager` adds existential overhead — negligible vs. CoreML inference, but worth knowing for tight benchmarks.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Parakeet/Streaming/StreamingAsrManagerTests.swift` — exercises the protocol via both conforming engines.
- `Tests/FluidAudioTests/ASR/Parakeet/Streaming/StreamingNemotronAsrManagerTests.swift` — Nemotron path.
- `Tests/FluidAudioTests/ASR/Parakeet/Streaming/EouChunkSizeFrameCountTests.swift` — EOU path chunk math.

## Changelogs
