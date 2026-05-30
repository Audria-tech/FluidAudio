---
id: offline-diarizer-manager
name: Offline Diarizer Manager
repo: FluidAudio
status: active
linked_features: [offline-diarizer-models, offline-diarizer-types, offline-segmentation, offline-extraction, offline-utils, ahc-clustering, vbx-clustering, speaker-count-constraints, fastcluster-bridge]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Offline Diarizer Manager

## TL;DR
`OfflineDiarizerManager` is the batch diarization pipeline ported from pyannote/community-1: segmentation → ECAPA embeddings → PLDA → AHC + VBx clustering → segment reconstruction. It produces a `DiarizationResult` with a fully assigned timeline and per-stage timings, and supports speaker-count constraints, embedding export, and disk-backed audio sources.

## Motivation — why it exists

## Context

## What It Does
`process(audioSource:audioLoadingSeconds:)` orchestrates the pipeline as two concurrent `Task`s feeding through an `AsyncThrowingStream<SegmentationChunk, Error>`:

1. **Segmentation task** uses `OfflineSegmentationProcessor` with the `segmentationModel`, producing a `SegmentationOutput` (per-chunk powerset log-probs + speaker weights) and yielding `SegmentationChunk`s to the stream.
2. **Embedding task** uses `OfflineEmbeddingExtractor` (`fbankModel` + `embeddingModel` + `PLDATransform`) to consume the stream and emit `[TimedEmbedding]`. Each timed embedding carries `embedding256`, `rho128`, frame range, and chunk/speaker index.

Once both tasks complete, the manager:
3. Filters NaN/Inf embeddings out of training (`selectTrainingEmbeddings`).
4. Runs `AHCClustering` with `clusteringThreshold` to seed cluster IDs.
5. Optionally runs `VBxClustering.refineWithConstraints` when the user has set any of `numSpeakers`/`minSpeakers`/`maxSpeakers` via `SpeakerCountConstraints.resolve`.
6. Computes per-cluster centroids: either directly from VBx K-Means (if `vbxOutput.wasAdjusted`), or weighted by `gamma`/`pi` (filtering speakers with `pi <= 1e-7`), or fallback to averaged hard-cluster means.
7. Assigns every (full, not just training) embedding to the nearest centroid by normalized dot product (`vDSP_dotprD`).
8. Builds a `[chunkIdx][speakerIdx] → cluster` matrix and hands it to `OfflineReconstruction.buildSegments` along with the segmentation output and centroids.
9. Builds the per-cluster speaker database via `OfflineReconstruction.buildSpeakerDatabase`.
10. Optionally exports embeddings + assignments to a JSON file (`config.embeddingExportPath`).

Model lifecycle: `initialize(models:)` accepts pre-loaded models; `prepareModels(directory:configuration:forceRedownload:)` will download + compile from HuggingFace, retrying once on initial failure (purging the cache before the retry). After loading, `prewarmSegmentationModel` runs an async dummy prediction on a zero-filled audio buffer, and `prewarmEmbeddingStack` extracts embeddings from a single-frame dummy segmentation to JIT the FBANK + embedding + PLDA path.

## Key Code
- [`OfflineDiarizerManager.swift:7`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerManager.swift#L7) — `@available(macOS 14.0, iOS 17.0, *)` `public final class`; `nonisolated(unsafe) private var models` is "managed safely by only writing during initialization".
- [`OfflineDiarizerManager.swift:29`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerManager.swift#L29) — `prepareModels(directory:configuration:forceRedownload:)` retry-with-purge logic on initial load failure.
- [`OfflineDiarizerManager.swift:102`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerManager.swift#L102) — `process(_:URL)` builds a disk-backed `AudioSampleSource` via `AudioSourceFactory` for memory-mapped streaming.
- [`OfflineDiarizerManager.swift:131`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerManager.swift#L131) — `AsyncThrowingStream` fan-out: segmentation feeds → embedding consumes.
- [`OfflineDiarizerManager.swift:234`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerManager.swift#L234) — VBx + constraint resolution: `SpeakerCountConstraints.resolve` only runs when at least one constraint is set.
- [`OfflineDiarizerManager.swift:442`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerManager.swift#L442) — `computeCentroids` — three modes: K-Means centroids (if `wasAdjusted`), gamma-weighted (VBx active speakers), or hard-cluster averages.
- [`OfflineDiarizerManager.swift:617`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerManager.swift#L617) — `assignEmbeddings` uses normalized dot product (`vDSP_dotprD`) over all embeddings, not just training set.

## Edge Cases & Failure Modes
- Throws `OfflineDiarizationError.modelNotLoaded("offline-diarizer")` if `prepareModels` was never called and the lazy `prepareModels` inside `process` also fails.
- Throws `.noSpeechDetected` when `timedEmbeddings.isEmpty` (no valid embeddings extracted).
- `config.validate()` runs first; throws `.invalidConfiguration` for any out-of-range parameter (clustering threshold > √2, batch size > 32, etc.).
- All embeddings being NaN/Inf → `selectTrainingEmbeddings` returns all indices anyway (defensive fallback) so AHC still runs on garbage; could produce single-cluster output silently. [REVIEW: consider raising vs. falling through to potentially unstable VBx.]
- If AHC returns < 2 training points, `initialClusters = [0]` and VBx is skipped, producing a single-speaker assignment.
- Cancellation: both inner tasks are cancelled on any error, and `chunkContinuation.finish(throwing:)` is called to release the embedding task waiting on the stream.
- `forceRedownload=true` purges before *and* on retry; logs `"Failed to purge..."` warnings but doesn't fail.

## Performance / Concurrency Notes
- Segmentation and embedding run concurrently via `async let awaitedSegmentation` / `awaitedEmbeddings`; both use `Task(priority: .userInitiated)`.
- `prewarmSegmentationModel` uses iOS 17+ `model.prediction(from:options:) async throws` to avoid the synchronous prediction's risk of indefinitely blocking on a contended ANE — the docstring explicitly cites this rationale.
- The segmentation model uses `.cpuAndNeuralEngine` while the FBANK model uses `.cpuOnly` (FBANK is fast enough on CPU and avoids ANE compilation cost).
- `nonisolated(unsafe) private var models` is the only mutable property, written only during `initialize`/`prepareModels` and read concurrently from tasks; this is safe by construction but bypasses the actor isolation checker. [REVIEW: documented but worth tracking — concurrent `prepareModels` calls could race.]
- Centroid computation uses `cblas_daxpy` for vector accumulation and `vDSP_vsmulD` for normalization.

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/Offline/OfflineModuleTests.swift` — end-to-end pipeline behaviour.
- `Tests/FluidAudioTests/Diarizer/Offline/OfflineConfigTests.swift` — `config.validate()` boundary cases.
- `Tests/FluidAudioTests/Diarizer/Offline/DiarizerMemoryTests.swift` — memory profile under sustained processing.
- `Tests/FluidAudioTests/Diarizer/Offline/AHCClusteringTests.swift`, `SpeakerCountConstraintsTests.swift`, `VBxConstraintTests.swift` cover the clustering substeps invoked here.

## Changelogs
