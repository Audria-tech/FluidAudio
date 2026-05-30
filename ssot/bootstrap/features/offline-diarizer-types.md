---
id: offline-diarizer-types
name: Offline Diarizer Types
repo: FluidAudio
status: active
linked_features: [offline-diarizer-manager, offline-segmentation, offline-extraction, ahc-clustering, vbx-clustering, speaker-count-constraints]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Offline Diarizer Types

## TL;DR
The offline-pipeline value types: `OfflineDiarizationError`, `OfflineDiarizerConfig` (segmentation + embedding + clustering + VBx + post-processing + export sub-configs with `.community` presets), `SegmentationLogits`/`SegmentationOutput`/`SegmentationChunk` (model output carriers), `VBxOutput`, `TimedEmbedding`, and the `EmbeddingSkipStrategy` enum.

## Motivation — why it exists

## Context

## What It Does
`OfflineDiarizerConfig` has six sub-structs each with a `.community` preset (pyannote/community-1 defaults). `Segmentation` (10s windows, 16kHz, stepRatio 0.2, on/off thresholds 0.5). `Embedding` (batchSize 32, excludeOverlap=true, minSegmentDuration 1.0s, `skipStrategy: EmbeddingSkipStrategy`). `Clustering` (threshold 0.6, warmStartFa 0.07, warmStartFb 0.8, optional `numSpeakers`/`minSpeakers`/`maxSpeakers`). `VBx` (maxIterations 20, convergenceTolerance 1e-4). `PostProcessing` (minGapDurationSeconds 0.1, exclusiveSegments=true). `Export` (optional `embeddingsPath`). The struct also offers a legacy flat init that mirrors all leaf parameters and a long list of pass-through computed properties (e.g. `clusteringThreshold` for `clustering.threshold`). `samplesPerWindow = sampleRate * windowDurationSeconds`, `samplesPerStep = max(1, samplesPerWindow * stepRatio)`. `validate()` enforces ranges (threshold ≤ √2, batch size ≤ 32, both Fa and Fb > 0, etc.). Two convenience builders: `withSpeakers(min:max:)` and `withSpeakers(exactly:)` apply the pyannote-style precedence (`numSpeakers` overrides min/max).

`EmbeddingSkipStrategy` enum: `.none` (every chunk) or `.maskSimilarity(threshold:)` (skip when current mask is ≥ threshold cosine-similar to the mask that produced the cached embedding). Recommended threshold per docstring: 0.95.

`SegmentationOutput` carries `logProbs: [[[Float]]]` (chunks × frames × classes), `speakerWeights: [[[Float]]]` (chunks × frames × speakers, 0..1 soft values), counts, `chunkOffsets`, and `frameDuration`. `SegmentationChunk` is the streaming variant emitted via the segmentation→embedding channel. `SegmentationChunkContinuation` / `SegmentationChunkHandler` are the typedefs for the chunk-handler callback.

`VBxOutput` has `gamma: [[Double]]` (frame × speaker soft assignment), `pi: [Double]` (mixture weights), `hardClusters: [[Int]]`, `centroids: [[Double]]`, `numClusters`, `elbos: [Double]`, plus `wasAdjusted: Bool` and `originalClusterCount: Int?` for tracking constraint-driven K-Means fallback.

`TimedEmbedding` is internal: `(chunkIndex, speakerIndex, startFrame, endFrame, frameWeights, startTime, endTime, embedding256, rho128)`.

## Key Code
- [`OfflineDiarizerTypes.swift:82`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerTypes.swift#L82) — `EmbeddingSkipStrategy` enum with `.maskSimilarity(threshold:)` case.
- [`OfflineDiarizerTypes.swift:299`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerTypes.swift#L299) — `validate()` — all the bounds checks.
- [`OfflineDiarizerTypes.swift:564`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerTypes.swift#L564) — `VBxOutput` with `wasAdjusted`/`originalClusterCount` markers.
- [`OfflineDiarizerTypes.swift:626`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerTypes.swift#L626) — `withSpeakers(min:max:)` / `withSpeakers(exactly:)` convenience builders.

## Edge Cases & Failure Modes
- `validate()` throws `.invalidBatchSize` for `embeddingBatchSize > 32` (PLDA batch limit) and `.invalidConfiguration` for all other range failures.
- `speechOffsetThreshold` must be `<= speechOnsetThreshold` — enforced by `validate()`.
- `withSpeakers(exactly:)` clears `min`/`max`; `withSpeakers(min:max:)` clears `numSpeakers`. The methods are not commutative if called in succession.
- `EmbeddingSkipStrategy.maskSimilarity` does not validate that `threshold` is in `[0, 1]` — caller responsibility.
- `SegmentationLogits` is `struct` `Sendable` but `internal` (not exported); it's used by the streaming path internally only.

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/Offline/OfflineConfigTests.swift` — validate boundary cases and the constructor.

## Changelogs
