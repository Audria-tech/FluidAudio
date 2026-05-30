---
id: sortformer-diarization
name: Sortformer Streaming Diarization
trigger_type: event
trigger: Application pushes audio via `SortformerDiarizer.addAudio(_:)` and calls `process()` to drain ready chunks.
end_state: A `DiarizerTimelineUpdate` is returned with cumulative confirmed + tentative predictions; `spkcache` / FIFO state is rotated for the next chunk.
involves_features: [sortformer-diarizer, sortformer-model-inference, sortformer-state-updater]
linked_flows: [model-download-and-load]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Sortformer Streaming Diarization

## TL;DR
NVIDIA Streaming-Sortformer 4-speaker streaming diarization. Single CoreML "main model" (preprocessor + pre-encoder + head) runs per chunk; a Swift port of NeMo's `StateUpdater` rotates the speaker cache (`spkcache`) and FIFO of recent embeddings to keep speaker identities stable across the stream.

## Trigger & Preconditions
`SortformerDiarizer.initialize(models:)` or `initialize(mainModelPath:)` must have succeeded. The class is **not** an actor — thread safety is hand-rolled with `NSLock` (docstring: "not thread-safe"). Audio fed via `addAudio` overloads (`[Float]`, generic `Collection`, or `sourceSampleRate:` variant with `AudioConverter.resample`).

## Stages
1. **Append audio + preprocess to mel** — `addAudio(_:)` appends to `audioBuffer`, immediately calls `preprocessAudioToFeaturesLocked` which runs `AudioMelSpectrogram.computeFlatTransposed` with `lastAudioSample` for preemphasis continuity. The "samples consumed" math inverts the center-padded frame formula `samplesConsumed = (melLength - 1) * melStride + melWindow - nFFT`: [`Sources/FluidAudio/Diarizer/Sortformer/SortformerDiarizer.swift:740`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerDiarizer.swift#L740).
2. **Drain loop entry** — `process()` enters `while let chunk = getNextChunkFeaturesLocked()`. The "full right context required" gate returns nil when `endFeat + rightContextFrames > featLength`: [`Sources/FluidAudio/Diarizer/Sortformer/SortformerDiarizer.swift:807`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerDiarizer.swift#L807).
3. **Run main model** — `SortformerModels.runMainModel(chunk:chunkLength:spkcache:spkcacheLength:fifo:fifoLength:config:)` loads the six pre-allocated ANE-aligned buffers via `optimizedCopy(pad: true)`, builds an `MLDictionaryFeatureProvider`, calls `prediction`, extracts `speaker_preds` (Float32), `chunk_pre_encoder_lengths` (Int32), `chunk_pre_encoder_embs` (Float32 preferred, Float16 fallback on arm64+macOS15+) by checking both `_out`-suffixed and plain names: [`Sources/FluidAudio/Diarizer/Sortformer/SortformerModelInference.swift:186`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerModelInference.swift#L186).
4. **Streaming state update** — `SortformerStateUpdater.streamingUpdate(state:&:chunk:preds:leftContext:rightContext:)` extracts FIFO predictions, extracts core-frame chunk embeddings (skipping `leftContext` frames), splits the chunk preds into confirmed (`coreFrames`) and tentative (`rightContext`): [`Sources/FluidAudio/Diarizer/Sortformer/SortformerStateUpdater.swift:31`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerStateUpdater.swift#L31). Chunk 0 uses `leftContext = 0`; subsequent chunks use `config.chunkLeftContext`: [`Sources/FluidAudio/Diarizer/Sortformer/SortformerDiarizer.swift:458`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerDiarizer.swift#L458).
5. **FIFO maintenance** — core embeddings appended to FIFO. When FIFO exceeds `fifoCapacity`, `popOutLength = clamp(spkcacheUpdatePeriod, contextLength - fifoCapacity, contextLength)` frames are popped and absorbed: silence frames (`probSum < silenceThreshold`) update the running mean silence embedding via `updateSilenceProfile`, others append to `spkcache`.
6. **Spkcache compression** — when `spkcacheLength > spkcacheCapacity`, `compressSpkcache` runs: compute per-frame log-pred scores `log(p) - log(1-p) + sum_others(log(1-p)) - log(0.5)`, disable non-speech / overlap, boost recent + top-k frames, select via `getTopKIndices`, fill disabled slots with the mean silence embedding: [`Sources/FluidAudio/Diarizer/Sortformer/SortformerStateUpdater.swift:220`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerStateUpdater.swift#L220).
7. **Timeline append** — `_timeline.addChunk(DiarizerChunkResult(...))` appends the chunk's finalized + tentative predictions and returns the cumulative `DiarizerTimelineUpdate`.

## Outputs
- `DiarizerTimelineUpdate` carrying cumulative finalized + tentative `DiarizerPrediction` lists.

## Error Modes
- `SortformerError.notInitialized` from `enrollSpeakerInternal` / `addAudio` / `process` / `finalizeSession` / `processCompleteInternal` if models aren't loaded.
- `SortformerError.inferenceFailed("Missing speaker_preds...")` / `("Missing chunk_pre_encoder_lengths...")` if model outputs are missing under both name conventions.
- `SortformerError.insufficientPredsLength` / `.insufficientChunkLength` when slice math underflows.
- Float16 chunk-embeddings fallback only on arm64 — Intel Macs cannot load a model that produces Float16 chunk embeddings.
- "FIFO predictions nil after update" defensive branch logs `THIS SHOULD NEVER HAPPEN!` and returns current chunk's preds [REVIEW noted].
- `addAudio` for empty samples is a silent no-op.
- Concurrent callers must serialize via `lock.withLock`; new public methods that forget the lock cause silent races.

## Prompts and Models Used

## Usage Metrics

## Changelogs
