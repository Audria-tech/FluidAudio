---
id: diarizer-types
name: Diarizer Types
repo: FluidAudio
status: active
linked_features: [diarizer-manager, diarizer-models]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Diarizer Types

## TL;DR
Sendable value types for the chunked diarization pipeline: `DiarizerConfig` (knobs), `PipelineTimings` (per-stage durations + bottleneck helpers), `DiarizationResult` (segments + optional speaker db + timings), `TimedSpeakerSegment`, `ModelPaths`, `AudioValidationResult`, and the `DiarizerError` enum.

## Motivation — why it exists

## Context

## What It Does
`DiarizerConfig` holds: `clusteringThreshold` (default 0.7 in field, 0.5 in `init`), `minSpeechDuration` (1.0s), `minEmbeddingUpdateDuration` (2.0s), `minSilenceGap` (0.5s), `numClusters` (-1 for auto), `minActiveFramesCount` (10.0), `debugMode`, `chunkDuration` (10.0s), `chunkOverlap` (0.0s). The static `.default` uses the field defaults (0.7 clustering threshold). [REVIEW: `default` and `init()` disagree on `clusteringThreshold` — field default is 0.7, init default is 0.5; `DiarizerConfig.default` produces 0.7 but `DiarizerConfig()` produces 0.5.]

`PipelineTimings` is `Sendable & Codable`. It auto-computes `totalInferenceSeconds` (segmentation + embedding + clustering) and `totalProcessingSeconds` (everything including compilation + audio loading + post-processing) in `init`. Exposes `stagePercentages: [String: Double]` and `bottleneckStage: String` for diagnostics.

`DiarizationResult` carries `segments: [TimedSpeakerSegment]` and optionals `speakerDatabase: [String: [Float]]?` and `timings: PipelineTimings?` — the latter two only populated when `debugMode=true` is configured.

`TimedSpeakerSegment` is `Sendable & Identifiable` with a UUID `id`, `speakerId`, `embedding`, `startTimeSeconds`, `endTimeSeconds`, `qualityScore`, and a computed `durationSeconds`.

`DiarizerError` covers: `notInitialized`, `modelDownloadFailed`, `modelCompilationFailed`, `embeddingExtractionFailed`, `invalidAudioData`, `processingFailed(String)`, `memoryAllocationFailed`, `invalidArrayBounds` — each with a localized description.

## Key Code
- [`DiarizerTypes.swift:7`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerTypes.swift#L7) — `DiarizerConfig` declaration; note the field/init default mismatch on `clusteringThreshold`.
- [`DiarizerTypes.swift:60`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerTypes.swift#L60) — `PipelineTimings` with auto-computed totals and `bottleneckStage`/`stagePercentages` introspection.
- [`DiarizerTypes.swift:144`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerTypes.swift#L144) — `TimedSpeakerSegment` with UUID id (newly generated on each `init`, so `==` cannot be derived purely from value equality).
- [`DiarizerTypes.swift:190`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerTypes.swift#L190) — `DiarizerError` enum + `errorDescription` cases.

## Edge Cases & Failure Modes
- The init/field default discrepancy on `clusteringThreshold`: callers using `DiarizerConfig.default` get 0.7, callers using `DiarizerConfig()` (no args) get 0.5. [REVIEW]
- `TimedSpeakerSegment.id` is regenerated on each init; segments are not equal even if all fields match.
- `PipelineTimings.bottleneckStage` returns `"Unknown"` if all durations are 0 (e.g., empty result).
- `stagePercentages` returns `[:]` when `totalProcessingSeconds == 0`.

## Test Coverage
- Used throughout `Tests/FluidAudioTests/Diarizer/` — no dedicated tests for the types themselves.

## Changelogs
