---
id: sortformer-diarizer
name: Sortformer Diarizer
repo: FluidAudio
status: active
linked_features: [sortformer-model-inference, sortformer-state-updater, sortformer-types]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Sortformer Diarizer

## TL;DR
`SortformerDiarizer` is the streaming diarization pipeline built around NVIDIA's Streaming-Sortformer 4-speaker model. It maintains an audio→mel-feature buffer, a speaker cache (`spkcache`), and a FIFO of recent embeddings; each chunk runs the combined CoreML "main model" (preprocessor + pre-encoder + head) and returns confirmed + tentative predictions via a `DiarizerTimeline`.

## Motivation — why it exists

## Context

## What It Does
The class conforms to `Diarizer` and uses an `NSLock` (the docstring explicitly notes "not thread-safe" but provides a `withLock` helper for the async initializer path). It owns four ingestion-side buffers — `audioBuffer: [Float]`, `featureBuffer: [Float]`, `lastAudioSample: Float`, `startFeat: Int` — plus the `SortformerStreamingState` (containing `spkcache`, `fifo`, silence-profile mean, etc.) and the `DiarizerTimeline`.

`initialize(mainModelPath:)` compiles + loads the model asynchronously via `SortformerModels.load`. `initialize(models:)` accepts a pre-loaded `SortformerModels` value. Both wire up state and reset buffers.

Streaming inputs come through three overloads of `addAudio` (raw `[Float]`, generic `Collection<Float>`, and a `sourceSampleRate` variant that runs `AudioConverter.resample` when needed). Each call appends to `audioBuffer` and immediately calls `preprocessAudioToFeaturesLocked` to produce mel features (128-d) up to the next chunk boundary. The mel call uses `AudioMelSpectrogram.computeFlatTransposed` and carries `lastAudioSample` for preemphasis continuity across calls.

`process()` is the drain step. It loops `getNextChunkFeaturesLocked` → `models.runMainModel(...)` → `stateUpdater.streamingUpdate(...)`, accumulating confirmed and tentative predictions. The actual left context fed to the state updater varies: chunk 0 has `leftContext=0`, subsequent chunks use the configured `chunkLeftContext`. Each loop iteration calls `_timeline.addChunk(DiarizerChunkResult(...))` to append finalized + tentative predictions, returning the cumulative `DiarizerTimelineUpdate`.

`processComplete` is the offline batch path: it resets streaming state (preserving enrolled speakers when the buffers were empty), then uses `SortformerFeatureLoader` to walk through all mel chunks at once, computing per-chunk `leftContext`/`rightContext` from the loader's offsets and calling `_timeline.rebuild(..., isComplete: finalizeOnCompletion)` at the end. There are three overloads (`[Float]`, generic Collection, and `URL`-based file processing).

`finalizeSession()` drains every remaining full-right-context chunk, then absorbs the trailing tentative predictions as finalized — mirroring offline `rebuild(isComplete: true)`. It is idempotent (subsequent calls return nil) and emits a single `DiarizerTimelineUpdate` covering everything new.

`enrollSpeaker(withAudio:sourceSampleRate:named:overwritingAssignedSpeakerName:)` is a one-shot priming flow: it clears audio/feature buffers, processes the enrollment audio with timeline updates enabled, picks the most-spoken speaker slot, optionally renames it, then resets all stream state while keeping the freshly named speaker in the timeline.

## Key Code
- [`SortformerDiarizer.swift:12`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerDiarizer.swift#L12) — `public final class SortformerDiarizer: Diarizer` with `lock = NSLock()`; "not thread-safe" callout in docstring.
- [`SortformerDiarizer.swift:149`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerDiarizer.swift#L149) — `resetBuffersLocked(keepingSpeakers:)` reserves capacity `(chunkMelFrames + coreFrames) * melFeatures` on `featureBuffer`.
- [`SortformerDiarizer.swift:225`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerDiarizer.swift#L225) — `enrollSpeakerInternal` is the priming logic: snapshot occupied slots, process audio, pick best speaker, rename, reset.
- [`SortformerDiarizer.swift:427`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerDiarizer.swift#L427) — `makeStreamingChunkLocked` is the per-chunk inference + state update pipeline.
- [`SortformerDiarizer.swift:458`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerDiarizer.swift#L458) — `leftContext: diarizerChunkIndex > 0 ? config.chunkLeftContext : 0` — the chunk-0 special-case for left context.
- [`SortformerDiarizer.swift:502`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerDiarizer.swift#L502) — `finalizeSession()` drain loop and tentative-as-finalized absorption.
- [`SortformerDiarizer.swift:740`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerDiarizer.swift#L740) — `preprocessAudioToFeaturesLocked` invertss the center-padded frame count formula to compute `samplesConsumed = (melLength - 1) * melStride + melWindow - nFFT`, preserving leftover samples across calls.
- [`SortformerDiarizer.swift:807`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerDiarizer.swift#L807) — `getNextChunkFeaturesLocked` enforces "full right context required before chunk emission" and trims `featureBuffer` to keep `leftContextFrames` of history.

## Edge Cases & Failure Modes
- Throws `SortformerError.notInitialized` from every gated path (`enrollSpeakerInternal`, `addAudio`, `process`, `finalizeSession`, `processCompleteInternal`) if models are not loaded.
- `addAudio` for empty samples is a silent no-op (no error).
- Calling `finalizeSession` after a prior `finalize` returns nil idempotently; the `_finalized` flag gates this.
- `enrollSpeaker` returns nil and warns when (a) audio is empty, (b) not enough audio was provided to fill chunk + right context, (c) the diarizer matched an existing named speaker and `overwriteAssignedSpeakerName=false`.
- `getNextChunkFeaturesLocked` returns nil if `endFeat + rightContextFrames > featLength`, preventing premature emission of chunks without right context.
- `preprocessAudioToFeatureTargetLocked` short-circuits when `audioBuffer.count < config.melWindow` — small initial buffers stall until enough samples arrive.
- The "absorb trailing tentative" branch in `finalizeSession` handles two cases: if we drained any chunk, we use the last drained chunk's tentative; if we didn't drain, we use the timeline's previously-stored tentative. There is no test for the second case at the boundary where exactly one chunk has been drained — [REVIEW: confirm absorb path when `aggTentative` is empty but `_timeline.tentativePredictions` is not.]

## Performance / Concurrency Notes
- The class is **not** an actor; thread safety is hand-rolled with `NSLock`. All "locked" private methods assume the caller holds `lock`. The async `initialize(mainModelPath:)` uses a small `withLock` closure helper to avoid holding the NSLock across `await`. [REVIEW: any external concurrent caller of `addAudio`/`process`/`finalizeSession` is correctly serialized, but the implementation is fragile — if a new public method forgets `lock.withLock { ... }`, races become silent.]
- Throughput tuning: `chunkMelFrames`, `coreFrames`, `chunkLen`, `chunkLeftContext`, `chunkRightContext`, `fifoLen`, `spkcacheLen`, `spkcacheUpdatePeriod` are all configured via `SortformerConfig`. Presets exist for fast/balanced/highContext × v2/v2.1.
- `audioBuffer.removeFirst(samplesConsumed)` is O(n) per call; for sustained streaming this could be a hot spot. The compensating preprocessing buffer reuses capacity (`reserveCapacity` at reset) but rebuilds float arrays on each chunk.

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/Sortformer/SortformerTests.swift` — core streaming behavior + `processComplete`.
- `Tests/FluidAudioTests/Diarizer/Sortformer/SortformerStreamingIntegrationTests.swift` — end-to-end streaming vs. batch parity.
- `Tests/FluidAudioTests/Diarizer/Sortformer/SortformerTimelineTests.swift` — finalize/tentative absorption.
- `Tests/FluidAudioTests/Diarizer/Sortformer/SortformerStateUpdaterTests.swift` — state-updater contract used here.

## Changelogs
