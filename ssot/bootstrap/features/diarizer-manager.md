---
id: diarizer-manager
name: Diarizer Manager
repo: FluidAudio
status: active
linked_features: [diarizer-models, diarizer-types, speaker-manager, embedding-extractor, segmentation-processor, segmentation-sliding-window, audio-validation, speaker-types]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Diarizer Manager

## TL;DR
`DiarizerManager` is the chunked pyannote-style diarization pipeline. It runs the segmentation CoreML model over fixed-size audio chunks, extracts per-speaker embeddings via `EmbeddingExtractor`, and tracks consistent speaker identities across chunks through an actor-backed `SpeakerManager`. It is the public entry point for "online-style" diarization on top of pyannote community-1 weights.

## Motivation — why it exists

## Context

## What It Does
The manager owns three collaborators: a `SegmentationProcessor`, an `EmbeddingExtractor`, and a `SpeakerManager` actor whose thresholds are derived from the `DiarizerConfig.clusteringThreshold` (speaker assignment uses `threshold * 1.2`; embedding-update uses `threshold * 0.8`). `initialize(models:)` consumes a `DiarizerModels` value, wires the embedding model into the extractor, and stores the segmentation model for later reuse.

`performCompleteDiarization(_:sampleRate:atTime:)` walks the input collection in `chunkDuration`-second chunks stepping by `chunkDuration - chunkOverlap` seconds (defaults: 10s window, 0s overlap). Each chunk is padded to `chunkSize` via vDSP into a reusable buffer, segmented (pyannote powerset output → 3 speakers × ~592 frames), masked per-speaker (silencing frames where more than one speaker is active), embedded, and validated. Valid embeddings with activity above `minActiveFramesCount` are sent to `speakerManager.assignSpeaker` along with a quality score (`magnitude/10` clamped, scaled by frame activity ratio). The returned per-speaker IDs feed back into `createTimedSegments`, which uses a dynamic activation threshold (0.3 normally, 0.15 when another speaker is also active in the same frame) to detect overlap, then filters runs shorter than `minSpeechDuration` and sorts by start time.

`extractSpeakerEmbedding(from:)` is a single-clip helper for building enrollment embeddings: it queries the segmentation model's `segments` output shape to determine `numFrames`, fills an all-ones mask, and runs the embedding model once. `validateEmbedding`, `validateAudio`, and `initializeKnownSpeakers` are thin pass-throughs to `AudioValidation` and `SpeakerManager`.

When `config.debugMode` is set, the returned `DiarizationResult` includes a flattened speaker database (id → current 256D embedding) and a populated `PipelineTimings`; otherwise only the segments are emitted.

## Key Code
- [`DiarizerManager.swift:42`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerManager.swift#L42) — `initialize(models: consuming DiarizerModels)` wires the embedding model into a new `EmbeddingExtractor` and consumes the models value.
- [`DiarizerManager.swift:92`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerManager.swift#L92) — `extractSpeakerEmbedding(from:)` reads the segmentation model's output shape to size the mask, then runs the embedding model with an all-ones mask.
- [`DiarizerManager.swift:153`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerManager.swift#L153) — `performCompleteDiarization(_:sampleRate:atTime:)` is the main chunk-loop entry point; computes chunk/step sizes and accumulates `TimedSpeakerSegment`s.
- [`DiarizerManager.swift:247`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerManager.swift#L247) — `processChunkWithSpeakerTracking` pads with `vDSP_mmov`, runs segmentation + embedding, computes per-speaker "clean" masks, and assigns IDs via `SpeakerManager`.
- [`DiarizerManager.swift:407`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerManager.swift#L407) — `createTimedSegments` walks the binarized frames per speaker, applying the 0.3/0.15 overlap-aware threshold.
- [`DiarizerManager.swift:483`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerManager.swift#L483) — `createSegmentIfValid` enforces `minSpeechDuration` and computes `qualityScore = magnitude/10 * (activity/frameCount)`.

## Edge Cases & Failure Modes
- Throws `DiarizerError.notInitialized` from both `extractSpeakerEmbedding` and `performCompleteDiarization` if `initialize` has not been called.
- `extractSpeakerEmbedding` throws `DiarizerError.processingFailed` if the segmentation model output shape cannot be queried (rank < 2 or missing `segments` output).
- Empty audio: the stride loop runs zero iterations, returning an empty `DiarizationResult` with no error.
- Audio shorter than `chunkSize`: the chunk buffer is zero-cleared before partial copy, so the segmentation model always sees a full-sized window padded with zeros.
- Speakers with activity below `config.minActiveFramesCount` (default 10 frames) get an empty ID and produce no segment.
- Embeddings failing `validateEmbedding` (magnitude < 0.1 or non-finite) produce an empty ID; the speaker counter logs are tracked but discarded.
- `qualityScore` formula in `createSegmentIfValid` divides activity by `endFrame - startFrame` (segment length), not total frames — this differs from the inline calculation in `processChunkWithSpeakerTracking` which divides by `numFrames`. [REVIEW: intentional asymmetry between assignment-time quality and segment-time quality; not currently documented.]
- `qualityScore` magnitude divisor (`10.0`) is hard-coded and produces values that quickly clip to 1.0 for unit-normalized embeddings (the embedding model output is not L2-normalized prior to this — normalization happens inside `SpeakerManager`).

## Performance / Concurrency Notes
- `DiarizerManager` is a `public final class` and is **not** `Sendable`/`actor`-isolated; callers must serialize concurrent calls themselves. `SpeakerManager` is an actor and is safe to share. [REVIEW: the class is mutated (`models`, `embeddingExtractor`) outside any isolation; concurrent `initialize`/`performCompleteDiarization` calls could race.]
- The chunk buffer (`chunkBuffer`) is allocated once and reused across all chunks of a single `performCompleteDiarization` call via `inout`, but is local to the call so concurrent calls would not collide.
- vDSP (`vDSP_vclr`, `vDSP_mmov`) is used for buffer init and Collection-to-buffer copy. The fast-path uses `withContiguousStorageIfAvailable`; non-contiguous collections fall back to a per-element loop.
- Memory: `ANEMemoryOptimizer` is held but only used for `embeddingExtractor`'s shared buffers (segmentation processor has its own).

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/SpeakerEnrollmentTests.swift` exercises `extractSpeakerEmbedding` and `initializeKnownSpeakers` end-to-end with stub models.
- `Tests/FluidAudioTests/Diarizer/SpeakerManagerTests.swift` covers the speaker assignment behavior used by this class.
- `Tests/FluidAudioTests/Diarizer/Extraction/EmbeddingExtractorOverflowTests.swift` covers the embedding-extractor failure modes that surface through the manager.

## Changelogs
