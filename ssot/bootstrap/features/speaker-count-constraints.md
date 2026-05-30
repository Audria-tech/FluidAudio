---
id: speaker-count-constraints
name: Speaker Count Constraints
repo: FluidAudio
status: active
linked_features: [offline-diarizer-manager, vbx-clustering, kmeans-clustering]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Speaker Count Constraints

## TL;DR
`SpeakerCountConstraints` is a small `Sendable` value type that resolves and validates user-supplied speaker count bounds (`numSpeakers`/`minSpeakers`/`maxSpeakers`) against the available embedding count. Used by `OfflineDiarizerManager` + `VBxClustering.refineWithConstraints`.

## Motivation — why it exists

## Context

## What It Does
Stores `numSpeakers: Int?`, `minSpeakers: Int`, `maxSpeakers: Int`. The static `resolve(numEmbeddings:numSpeakers:minSpeakers:maxSpeakers:)` applies pyannote-style precedence: if `numSpeakers` is set, it overrides both bounds (`resolvedMin = resolvedMax = numSpeakers`). Otherwise `min` defaults to 1, `max` defaults to `numEmbeddings`. All values are clamped to `[1, numEmbeddings]`. If `min > max`, `min` is silently clamped down to `max` (logged at warning level). The final `numSpeakers` is set iff `min == max`. Two helpers: `needsAdjustment(detectedCount:)` returns true when `detectedCount < min || detectedCount > max`; `targetCount(detectedCount:)` clamps the detected count to the valid range.

## Key Code
- [`SpeakerCountConstraints.swift:6`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Clustering/SpeakerCountConstraints.swift#L6) — `struct SpeakerCountConstraints: Sendable` declaration.
- [`SpeakerCountConstraints.swift:25`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Clustering/SpeakerCountConstraints.swift#L25) — `resolve(...)` — precedence + clamping logic.
- [`SpeakerCountConstraints.swift:63`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Clustering/SpeakerCountConstraints.swift#L63) — `needsAdjustment` / `targetCount` helpers.

## Edge Cases & Failure Modes
- `min > max` clamps `min` silently (logs warning); the caller's intent may be silently lost.
- Negative or zero `numSpeakers`/`minSpeakers` are clamped to 1.
- `numEmbeddings == 0` is not explicitly handled — `resolvedMin` becomes `min(numEmbeddings, 1) = 0`, then clamped via `max(1, ...) = 1`, and `resolvedMax` becomes `max(1, min(0, max ?? 0)) = 1`. Result: `min=max=1`, even though no embeddings exist. [REVIEW: empty-embedding case is mathematically degenerate but doesn't crash.]
- `numSpeakers` is preserved in the output only when `min == max` after clamping; otherwise it's set to the input value (which is `nil` if the caller didn't pass it).

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/Offline/SpeakerCountConstraintsTests.swift` — exhaustive coverage of `resolve`, `needsAdjustment`, `targetCount`.

## Changelogs
