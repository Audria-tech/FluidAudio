---
id: offline-batch-diarization
name: Offline Batch Diarization
trigger_type: user-action
trigger: Caller invokes `OfflineDiarizerManager.process(audioSource:audioLoadingSeconds:)` or the `URL` overload.
end_state: A `DiarizationResult` is returned with a fully assigned speaker timeline + per-stage `PipelineTimings`, optionally with the per-cluster speaker database; an embedding-export JSON may be written to disk.
involves_features: [offline-diarizer-manager, offline-segmentation, offline-extraction, ahc-clustering, kmeans-clustering, vbx-clustering, speaker-count-constraints, fastcluster-bridge]
linked_flows: [model-download-and-load, model-warmup]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Offline Batch Diarization

## TL;DR
Batch diarization ported from pyannote/community-1: segmentation → ECAPA embeddings → PLDA → AHC + VBx clustering → segment reconstruction. Segmentation and embedding tasks run concurrently via an `AsyncThrowingStream`; clustering is sequential after both finish.

## Trigger & Preconditions
`OfflineDiarizerManager.process(...)`. `initialize(models:)` or lazy `prepareModels` must succeed (with retry-after-purge on the first failure). `prewarmSegmentationModel` + `prewarmEmbeddingStack` run after load to JIT the ANE path. URL variant uses `AudioSourceFactory` for memory-mapped streaming.

## Stages
1. **Validate config** — `config.validate()` throws `.invalidConfiguration` for any out-of-range parameter (clustering threshold > √2, batch size > 32, etc.).
2. **Spawn concurrent tasks** — `async let awaitedSegmentation` and `awaitedEmbeddings` both run at `.userInitiated`; an `AsyncThrowingStream<SegmentationChunk, Error>` joins them: [`Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerManager.swift:131`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerManager.swift#L131).
3. **Segmentation task** — `OfflineSegmentationProcessor.process(audioSource:segmentationModel:config:chunkHandler:)` slides a 10-second window with overlap-reuse via `memmove` + `vDSP_vclr` for trailing pad, batches up to 32 windows per CoreML call, decodes the 8-class powerset `[[], [0], [1], [2], [0,1], [0,2], [1,2], [0,1,2]]` into `[chunk][frame][speaker(0..2)]` log-probs + soft weights, yields `SegmentationChunk`s through the stream: [`Sources/FluidAudio/Diarizer/Offline/Segmentation/OfflineSegmentationProcessor.swift:44`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Segmentation/OfflineSegmentationProcessor.swift#L44).
4. **Embedding task** — `OfflineEmbeddingExtractor` consumes the stream: per-chunk overlap-aware speaker masks → batched FBANK → batched 256-dim ECAPA embedding → batched `PLDATransform` to 128-dim `rho`. Optional skip strategy (`.maskSimilarity(threshold:)`) reuses the cached embedding when current-vs-cached mask cosine similarity ≥ threshold: [`Sources/FluidAudio/Diarizer/Offline/Extraction/OfflineEmbeddingExtractor.swift:81`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Extraction/OfflineEmbeddingExtractor.swift#L81).
5. **Filter training embeddings** — `selectTrainingEmbeddings` drops NaN/Inf embeddings (defensive fallback returns all indices if every embedding is invalid).
6. **AHC seed clustering** — `AHCClustering.cluster(embeddingFeatures:threshold:)` L2-normalizes per-row via `vDSP_dotprD` + `vDSP_vsmulD`, calls `fastcluster_compute_centroid_linkage` over the FFI bridge, cuts the SciPy-format dendrogram at the converted threshold, renumbers cluster IDs densely: [`Sources/FluidAudio/Diarizer/Offline/Clustering/AHCClustering.swift:20`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Clustering/AHCClustering.swift#L20).
7. **Optional VBx refinement** — when any of `numSpeakers`/`minSpeakers`/`maxSpeakers` is set, `SpeakerCountConstraints.resolve(...)` runs and `VBxClustering.refineWithConstraints` adjusts cluster assignments (using K-Means or gamma/pi-weighted centroids): [`Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerManager.swift:234`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerManager.swift#L234).
8. **Compute centroids** — three modes: K-Means centroids when `vbxOutput.wasAdjusted`, gamma-weighted (`pi <= 1e-7` filter) for VBx active speakers, or hard-cluster averages as fallback. Uses `cblas_daxpy` + `vDSP_vsmulD`: [`Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerManager.swift:442`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerManager.swift#L442).
9. **Assign all embeddings** — `assignEmbeddings` walks every (not just training) embedding and picks the nearest centroid via normalized dot product (`vDSP_dotprD`): [`Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerManager.swift:617`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Core/OfflineDiarizerManager.swift#L617).
10. **Reconstruct segments** — `OfflineReconstruction.buildSegments(...)` consumes the `[chunkIdx][speakerIdx] → cluster` matrix plus the segmentation output and centroids to produce a timeline.
11. **Optional export** — `config.embeddingExportPath` writes embeddings + assignments to JSON.

## Outputs
- `DiarizationResult` with `segments`, optional `speakerDatabase`, and `PipelineTimings`.
- Optional disk JSON export of embeddings.

## Error Modes
- `OfflineDiarizationError.modelNotLoaded("offline-diarizer")` if `prepareModels` fails and lazy prepare inside `process` also fails.
- `.noSpeechDetected` if `timedEmbeddings.isEmpty`.
- `.invalidConfiguration` from `config.validate()`.
- All-NaN/Inf embeddings → `selectTrainingEmbeddings` returns all anyway → AHC runs on garbage → potentially single-cluster output [REVIEW].
- AHC < 2 training points → VBx skipped, single-speaker assignment.
- FastCluster FFI returning non-success → defensive fallback `Array(0..<count)` (every embedding its own cluster).
- `nonisolated(unsafe) private var models` — bypasses isolation checker; concurrent `prepareModels` could race [REVIEW].

## Prompts and Models Used

## Usage Metrics

## Changelogs
