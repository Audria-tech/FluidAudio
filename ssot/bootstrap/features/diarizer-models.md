---
id: diarizer-models
name: Diarizer Models
repo: FluidAudio
status: active
linked_features: [diarizer-manager, diarizer-types]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Diarizer Models

## TL;DR
`DiarizerModels` is the Sendable value type that bundles the pyannote-segmentation `MLModel` and the WeSpeaker-embedding `MLModel` used by `DiarizerManager`, along with the compilation/load duration.

## Motivation — why it exists

## Context

## What It Does
Holds two CoreML model references (`segmentationModel`, `embeddingModel`) plus a `compilationDuration: TimeInterval`. Provides three loading paths: `download(to:configuration:progressHandler:)` and `downloadIfNeeded(...)` both delegate to `DownloadUtils.loadModels(.diarizer, ...)` with the required model names from `ModelNames.Diarizer`; `load(localSegmentationModel:localEmbeddingModel:configuration:)` loads pre-downloaded `.mlmodelc` URLs without any download attempt. `defaultModelsDirectory()` returns `MLModelConfigurationUtils.defaultModelsDirectory(for: .diarizer)`. `defaultConfiguration()` chooses `.cpuAndNeuralEngine` under CI (`CI` env var) and `.all` otherwise. `CoreMLDiarizer` is a public typealias namespace (`SegmentationModel`, `EmbeddingModel` both `= MLModel`).

## Key Code
- [`DiarizerModels.swift:10`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerModels.swift#L10) — `public struct DiarizerModels: Sendable` declaration with required-models tie-in via `ModelNames.Diarizer.requiredModels`.
- [`DiarizerModels.swift:40`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerModels.swift#L40) — `download(to:configuration:progressHandler:)` — runs `DownloadUtils.loadModels(.diarizer, ...)` and throws `DiarizerError.modelDownloadFailed` if either model is missing from the result dictionary.
- [`DiarizerModels.swift:120`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerModels.swift#L120) — `load(localSegmentationModel:localEmbeddingModel:configuration:)` for pre-compiled mlmodelc loading without fallback.

## Edge Cases & Failure Modes
- Throws `DiarizerError.modelDownloadFailed` when `DownloadUtils.loadModels` returns a dict without both `segmentationFile` and `embeddingFile` entries.
- The local-load path throws whatever `MLModel(contentsOf:configuration:)` throws — no recovery.
- `compilationDuration` is the wall-clock time of the entire load (download + compile), not strictly compile time.

## Test Coverage
- Indirectly via `Tests/FluidAudioTests/Diarizer/SpeakerEnrollmentTests.swift` and `OfflineModuleTests.swift` which load real or stub models.

## Changelogs
