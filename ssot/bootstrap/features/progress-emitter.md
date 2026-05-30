---
id: progress-emitter
name: Progress Emitter
repo: FluidAudio
status: active
linked_features: []
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Progress Emitter

## TL;DR
`actor`-isolated bridge from imperative progress reporting to a single `AsyncThrowingStream<Double, Error>`. Used by long-running async operations (downloads, model compilation, batched inference) that want to expose a Swift Concurrency stream of `0.0…1.0` to consumers.

## Motivation — why it exists

## Context

## What It Does
`ensureSession()` returns the current stream or starts a new one. `report(progress:)` clamps to `[0,1]` and yields if a session is active. `finishSession()` yields `1.0` then finishes the stream. `failSession(error:)` finishes the stream with the error. Internally tracks `continuation`, `streamStorage`, and `isActive`; `reset()` wipes all three after a finish/fail. The initial `startSession()` also yields `0.0` so consumers always see a leading tick.

## Key Code
- [`Sources/FluidAudio/Shared/ProgressEmitter.swift:3`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ProgressEmitter.swift#L3) — `actor ProgressEmitter`
- [`Sources/FluidAudio/Shared/ProgressEmitter.swift:10`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ProgressEmitter.swift#L10) — `ensureSession` — idempotent getter
- [`Sources/FluidAudio/Shared/ProgressEmitter.swift:17`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ProgressEmitter.swift#L17) — `report(progress:)` — clamped yield
- [`Sources/FluidAudio/Shared/ProgressEmitter.swift:36`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ProgressEmitter.swift#L36) — `startSession` — `AsyncThrowingStream.makeStream()` + initial `0.0` yield

## Edge Cases & Failure Modes
- `report` is a silent no-op when `isActive == false` — calling before `ensureSession()` drops the report.
- Re-starting after finish requires calling `ensureSession()` again; the actor doesn't auto-restart.
- Only one consumer at a time — `AsyncThrowingStream` is single-consumer. A second `for await` would not receive any values until the first finishes.
- `failSession` doesn't check `isActive` — calling it twice is safe (the second call has no continuation) but the order is significant.
- No buffering policy is set on `makeStream()` — defaults apply; very high-rate `report` calls may be dropped silently.

## Test Coverage
No dedicated unit test; consumed by pipelines that wire it into UI progress.

## Changelogs
