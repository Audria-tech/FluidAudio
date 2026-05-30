---
id: online-diarization
name: Online Diarization (DiarizerManager)
trigger_type: user-action
trigger: Caller invokes `DiarizerManager.performCompleteDiarization(_:sampleRate:atTime:)` with a buffer of 16 kHz audio.
end_state: A `DiarizationResult` carrying per-speaker `TimedSpeakerSegment`s is returned; the `SpeakerManager` actor's in-memory database has been updated with EMA embedding updates for matched speakers and new speaker IDs for previously-unseen voices.
involves_features: [diarizer-manager, segmentation-processor, segmentation-sliding-window, embedding-extractor, speaker-manager, audio-validation]
linked_flows: [model-download-and-load, model-warmup]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Online Diarization (DiarizerManager)

## TL;DR
Chunked pyannote-style diarization: walk the input in 10-second chunks, run the segmentation CoreML model (powerset → 3 speakers × ~592 frames per chunk), mask per-speaker, compute speaker embeddings via the WeSpeaker model, assign consistent identities through the actor-backed `SpeakerManager`, and reconstruct timed segments.

## Trigger & Preconditions
`DiarizerManager.performCompleteDiarization(_:sampleRate:atTime:)`. `initialize(models:)` must have consumed a `DiarizerModels` value before any diarization call. Audio is any `Collection<Float>` at 16 kHz; non-16 kHz audio is the caller's responsibility to resample.

## Stages
1. **Validate** — `AudioValidation.validateAudio(_:)` checks duration ≥ 1 s, RMS ≥ 0.01; issues are logged but the call proceeds: [`Sources/FluidAudio/Diarizer/Segmentation/AudioValidation.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Segmentation/AudioValidation.swift#L5).
2. **Chunk loop** — `performCompleteDiarization` strides by `chunkDuration - chunkOverlap` seconds (defaults: 10 s window, 0 s overlap), reusing a single `chunkBuffer: [Float]` via `inout` between iterations: [`Sources/FluidAudio/Diarizer/Core/DiarizerManager.swift:153`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerManager.swift#L153).
3. **Pad chunk** — `vDSP_vclr` clears the buffer then `vDSP_mmov` copies `min(chunkSize, available)` samples in (fast path uses `withContiguousStorageIfAvailable`; non-contiguous Collections fall back to per-element loop).
4. **Segmentation** — `SegmentationProcessor.getSegments(audioChunk:segmentationModel:chunkSize:)` allocates an ANE-aligned `[1, 1, 160_000]` MLMultiArray, calls `prefetchToNeuralEngine`, runs the model, decodes the `[1, frames, 7]` powerset output via per-frame `vDSP_maxvi` argmax + lookup against `[[], [0], [1], [2], [0,1], [0,2], [1,2]]` → binarized `[batch][frame][speaker(0..2)]`: [`Sources/FluidAudio/Diarizer/Segmentation/SegmentationProcessor.swift:24`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Segmentation/SegmentationProcessor.swift#L24).
5. **Build sliding window feature** — `createSlidingWindowFeature` wraps the binarized tensor with `SlidingWindow(start: chunkOffset, duration: 0.0619375, step: 0.016875)`: [`Sources/FluidAudio/Diarizer/Segmentation/SegmentationProcessor.swift:224`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Segmentation/SegmentationProcessor.swift#L224).
6. **Per-speaker mask** — `processChunkWithSpeakerTracking` builds a "clean" mask per speaker, silencing frames where more than one speaker is active so the embedding model sees only that speaker's audio: [`Sources/FluidAudio/Diarizer/Core/DiarizerManager.swift:247`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerManager.swift#L247).
7. **Extract embeddings** — `EmbeddingExtractor.getEmbeddings(audio:masks:minActivityThreshold:)` runs the 256-dim WeSpeaker model with ANE-aligned buffers, repeat-padding the audio (vDSP doubling) to 160k samples and zero-vector-emitting for speakers below `minActivityThreshold = 10`: [`Sources/FluidAudio/Diarizer/Extraction/EmbeddingExtractor.swift:27`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Extraction/EmbeddingExtractor.swift#L27).
8. **Validate embedding** — `AudioValidation.validateEmbedding(_:)` rejects non-finite or sub-magnitude-0.1 vectors: [`Sources/FluidAudio/Diarizer/Segmentation/AudioValidation.swift:31`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Segmentation/AudioValidation.swift#L31).
9. **Assign speaker** — `SpeakerManager.assignSpeaker(_:speechDuration:confidence:speakerThreshold:newName:)` L2-normalizes via `VDSPOperations`, finds the nearest existing speaker by cosine distance, and either updates (EMA `alpha=0.9` when distance < `embeddingThreshold`), creates a new ID when `speechDuration >= minSpeechDuration`, or drops the segment: [`Sources/FluidAudio/Diarizer/Clustering/SpeakerManager.swift:128`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerManager.swift#L128).
10. **Build timed segments** — `createTimedSegments` walks binarized frames per speaker with a dynamic activation threshold (0.3 normally, 0.15 when another speaker is also active in the same frame to detect overlap), drops runs shorter than `minSpeechDuration`, sorts by start time: [`Sources/FluidAudio/Diarizer/Core/DiarizerManager.swift:407`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Core/DiarizerManager.swift#L407).
11. **Return result** — `DiarizationResult` carries the segment list; in `config.debugMode` also includes a flattened speaker database (id → current 256D embedding) and `PipelineTimings`.

## Outputs
- `DiarizationResult { segments: [TimedSpeakerSegment], speakerDatabase?, timings? }`.
- Side effect: `SpeakerManager` actor's `speakerDatabase` updated with EMA embedding updates and possibly new speaker IDs.

## Error Modes
- `DiarizerError.notInitialized` if `initialize(models:)` was not called.
- `DiarizerError.processingFailed` if the segmentation model can't be queried for its output shape (rank < 2 or missing `segments` output).
- Empty audio → empty `DiarizationResult` (no error).
- Audio shorter than `chunkSize`: padded with zeros; segmentation sees a partly-silent window.
- Speakers below `minActiveFramesCount` (default 10) get empty IDs and produce no segment.
- `DiarizerManager` is **not** actor-isolated — concurrent calls race on `models`/`embeddingExtractor` mutable refs [REVIEW].
- The 7-entry powerset is hard-coded — any new community-N weights with > 3 speakers would silently mis-decode.
- `qualityScore` magnitude divisor (10.0) is hard-coded and clips quickly for unit-normalized embeddings.

## Prompts and Models Used

## Usage Metrics

## Changelogs
