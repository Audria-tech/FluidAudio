---
id: mlmultiarray-extensions
name: MLMultiArray Extensions
repo: FluidAudio
status: active
linked_features: []
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# MLMultiArray Extensions

## TL;DR
File-internal `MLMultiArray.reset(to:)` extension that bulk-fills the backing buffer with a single `NSNumber` value for `float32` and `int32` arrays. Used for clearing pooled buffers without going through Swift's element subscript.

## Key Code
- [`Sources/FluidAudio/Shared/MLMultiArray+Extensions.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/MLMultiArray%2BExtensions.swift#L5) — `reset(to:)` — `dataPointer.bindMemory(...).update(repeating:count:)` per dtype

## Motivation — why it exists

## Context

## What It Does
Switch on `dataType`: for `.float32` binds the data pointer to `Float` and calls `update(repeating: value.floatValue, count: count)`; for `.int32` does the same with `Int32`. Any other dtype is silently a no-op.

## Edge Cases & Failure Modes
- Silently no-ops for `.float16`, `.float64`, and any future dtype. Callers expecting a wider dtype reset will see stale data. [REVIEW: `MLMultiArray.reset(to:)` silently no-ops on non-`.float32`/`.int32` dtypes — collides with the similar `resetData(to:)` in the TDT decoder file (loops via subscript and works on every dtype)]
- Has no internal visibility modifier — defaults to `internal`. Not exposed outside the module.
- Distinct from `resetData(to:)` (defined in `TdtDecoderState.swift`) which uses element subscript and works on every dtype; the two methods coexist and serve overlapping purposes.

## Test Coverage
No dedicated tests for this extension; behaviour is exercised by hot-path buffer zeroing.

## Changelogs
