---
id: segmentation-sliding-window
name: Segmentation Sliding Window
repo: FluidAudio
status: active
linked_features: [segmentation-processor, diarizer-manager]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Segmentation Sliding Window

## TL;DR
Internal value types describing the pyannote segmentation model's frame-to-time mapping. `SlidingWindow` (start/duration/step), `Segment` (frame interval), and `SlidingWindowFeature` (data tensor + window metadata) live in this file. All internal to the module.

## Motivation — why it exists

## Context

## What It Does
`Segment` is a `Hashable` pair `(start: Double, end: Double)`. `SlidingWindow` stores `start`, `duration`, `step` (all `Double` seconds) and exposes two helpers: `time(forFrame:)` returns `start + index * step`, and `segment(forFrame:)` returns the `Segment(start: s, end: s + duration)`. `SlidingWindowFeature` is the carrier: `data: [[[Float]]]` (batch × frame × speaker) plus the associated `SlidingWindow`.

## Key Code
- [`SlidingWindow.swift:3`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Segmentation/SlidingWindow.swift#L3) — `internal struct Segment: Hashable`.
- [`SlidingWindow.swift:8`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Segmentation/SlidingWindow.swift#L8) — `SlidingWindow.time(forFrame:)` and `segment(forFrame:)`.
- [`SlidingWindow.swift:23`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Segmentation/SlidingWindow.swift#L23) — `SlidingWindowFeature` carrier.

## Edge Cases & Failure Modes
- All three types are `internal`, so they cannot be constructed by external callers (`SegmentationProcessor` is the only producer).
- `time(forFrame:)` accepts any Int including negatives; negative frames yield `start - |index| * step`, which is meaningful for left-context calculations but undefined for sliced data tensors.
- `Segment` has no validation that `start <= end`.

## Test Coverage
- No dedicated tests; covered transitively through `SegmentationProcessorTests.swift` (offline) and `DiarizerManager` integration tests.

## Changelogs
