---
id: offline-utils
name: Offline Utils
repo: FluidAudio
status: active
linked_features: [offline-diarizer-manager, offline-diarizer-types]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Offline Utils

## TL;DR
Two files: `OfflineReconstruction` builds the final `TimedSpeakerSegment` array from the segmentation output + hard cluster assignments + centroids, applying min-gap merging and optional exclusive-segment trimming. `VDSPOperations` wraps the common vDSP routines (`l2Normalize`, `dotProduct`, `matrixVectorMultiply`, etc.) used across the clustering pipeline.

## Motivation — why it exists

## Context

## What It Does
**`OfflineReconstruction`** uses the soft `speakerWeights` from segmentation, the `hardClusters` matrix (chunk × speaker → cluster), and centroids to accumulate per-cluster activation sums and counts on a uniform time grid (`totalFrames = ceil(maxChunkEndTime / frameDuration)`). For each frame it picks the dominant cluster (highest average activation) above an internal activity threshold, then groups consecutive same-cluster frames into segments. It applies `minGapDuration` (max of `config.minGapDuration` and `config.segmentationMinDurationOff`) to merge same-cluster segments separated by short gaps. When `postProcessing.exclusiveSegments=true`, overlapping segments are trimmed so only one cluster is active at any time. `buildSpeakerDatabase(segments:)` collapses per-cluster embeddings into a `[String: [Float]]` map.

**`VDSPOperations`** provides `l2Normalize` (with `epsilon=1e-12` guard), `dotProduct`, `matrixVectorMultiply` (via `cblas_sgemv`), and other vector helpers used in clustering and centroid math. All routines are static and use `precondition` for shape consistency.

## Key Code
- [`OfflineReconstruction.swift:19`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Utils/OfflineReconstruction.swift#L19) — `buildSegments` — the main reconstruction entry point.
- [`OfflineReconstruction.swift:8`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Utils/OfflineReconstruction.swift#L8) — `Accumulator` struct tracks `start`/`end`/`scoreSum`/`frameCount` per active span.
- [`VDSPOperations.swift:12`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Utils/VDSPOperations.swift#L12) — `l2Normalize` with 1e-12 epsilon (also used by `Speaker.init` and `SpeakerManager` paths).
- [`VDSPOperations.swift:32`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Utils/VDSPOperations.swift#L32) — `matrixVectorMultiply` via `cblas_sgemv`.

## Edge Cases & Failure Modes
- `buildSegments` returns `[]` if `numChunks == 0`, `numFrames == 0`, or `frameDuration <= 0`.
- The `clusterCount = max(centroids.count, 1)` guard ensures at least one cluster is allocated even with empty centroids (degenerate single-cluster fallback).
- `VDSPOperations.dotProduct` traps with `precondition` on length mismatch (no silent failure).
- `matrixVectorMultiply` flattens the row-major matrix on every call — no caching; for repeated calls with the same matrix, the caller pays the allocation cost each time.
- `l2Normalize` returns the input unchanged for empty inputs (avoids division-by-zero by returning early before computing the norm).

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/Offline/VDSPOperationsTests.swift` — vector ops.
- `Tests/FluidAudioTests/Diarizer/Offline/OfflineModuleTests.swift` — reconstruction integrated into the pipeline.

## Changelogs
