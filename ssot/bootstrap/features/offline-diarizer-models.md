---
id: offline-diarizer-models
name: Offline Diarizer Models
repo: FluidAudio
status: active
linked_features: [offline-diarizer-manager, offline-extraction]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Offline Diarizer Models

## TL;DR
`OfflineDiarizerModels` bundles the four CoreML models (segmentation, FBANK, embedding, PLDA-rho) plus the PLDA `psi` parameter array required by the offline pyannote/community-1 pipeline. `load(from:)` downloads, compiles, and loads everything from disk via `DownloadUtils`.

## Motivation — why it exists

## Context

## What It Does
Stores four `MLModel`s and a `pldaPsi: [Double]` array, plus `compilationDuration`. Available on `macOS 14.0+` / `iOS 17.0+`. The PLDA `psi` is loaded from a `plda-parameters.json` sidecar (tried in four candidate paths under the model directory): the JSON is parsed for `tensors.psi.data_base64`, base64-decoded, reinterpreted as Float32 via `withUnsafeMutableBytes`, then cast to `[Double]`.

`load(from:configuration:progressHandler:)` runs two `DownloadUtils.loadModels` calls: the first batch (`segmentation`, `embedding`, `pldaRho`) uses `.cpuAndNeuralEngine`; the second (`fbank`) uses `.cpuOnly` (FBANK is fast on CPU and ANE compilation isn't worth the cost). After both loads, it calls `loadPLDAPsi` and reports `compilationDuration`.

## Key Code
- [`OfflineDiarizerModels.swift:17`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerModels.swift#L17) — `loadPLDAPsi` candidate-path resolution + base64 decode.
- [`OfflineDiarizerModels.swift:78`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerModels.swift#L78) — `load(from:configuration:progressHandler:)` two-pass loader with mixed compute units.
- [`OfflineDiarizerModels.swift:146`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerModels.swift#L146) — `MLComputeUnits.label` debug helper.

## Edge Cases & Failure Modes
- Throws `OfflineDiarizationError.processingFailed("PLDA parameters file not found...")` if none of the four candidate paths contains `plda-parameters.json`.
- Throws `OfflineDiarizationError.processingFailed("Failed to decode PLDA psi parameters")` if the JSON shape is wrong.
- Throws `OfflineDiarizationError.processingFailed("PLDA psi tensor is empty")` if the base64 decodes to zero floats.
- Throws `OfflineDiarizationError.modelNotLoaded(name)` for each missing model in either download batch.
- The compute-unit split is hard-coded — overriding the `configuration` parameter does not change FBANK's `.cpuOnly` selection.

## Test Coverage
- Indirectly via `Tests/FluidAudioTests/Diarizer/Offline/OfflineModuleTests.swift`.

## Changelogs
