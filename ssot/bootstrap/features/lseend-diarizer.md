---
id: lseend-diarizer
name: LS-EEND Diarizer
repo: FluidAudio
status: active
linked_features: [lseend-inference, lseend-preprocessor, lseend-types]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# LS-EEND Diarizer

## TL;DR
`LSEENDDiarizer` is the streaming pipeline for Long-form Streaming End-to-End Neural Diarization. It owns an `LSEENDModel` (CoreML T-block) and an `LSEENDFeatureProvider` (STFT → log10-mel → CMN → subsample+context queue), and conforms to the `Diarizer` protocol with streaming, complete-file, finalize, and speaker-enrollment paths.

## Motivation — why it exists

## Context

## What It Does
The class is a `public final class` (not actor-isolated) but the underlying model and feature provider each carry their own `NSLock`s, so single-threaded use is safe and most operations are reentrant per-method.

On `init(model:timelineConfig:)`, the diarizer pulls `sampleRate`, `frameDurationSeconds`, and `maxSpeakers` from the model metadata, configures a `LSEENDFeatureProvider`, and instantiates a `DiarizerTimeline`. Convenience initializers exist for downloading from HuggingFace (`init(variant:stepSize:timelineConfig:)`) and for replacing the model on a live instance (`initialize(model:)` or `initialize(variant:stepSize:cacheDirectory:computeUnits:progressHandler:)`) — the replacement path rebuilds metadata-derived properties and resets streaming state.

Streaming API: `addAudio(samples:sourceSampleRate:)` simply enqueues into the feature provider (which handles resampling, STFT, log10 conversion, cumulative-mean normalization, and chunk queuing internally). `process()` calls the shared `flush(progressCallback:)` helper, which drains every ready mel chunk through `model.predict(from:)` (skipping warmup frames inside `model.predict`) and appends the resulting predictions to the timeline via `timeline.addPredictions(finalizedPredictions:tentativePredictions:)`. There is no separate "tentative" path here — LS-EEND emits finalized frames directly once the warmup quota is satisfied.

`finalizeSession()` calls `session.drainRightContextWithSilence()` (pushing silence to flush real frames out of the right context), runs one more `process()`, marks the timeline finalized, and sets the `finalized` flag.

Offline API: `processComplete(_:sourceSampleRate:keepingEnrolledSpeakers:finalizeOnCompletion:progressCallback:)` resets streaming state, enqueues all audio, then flushes with optional finalization. The `audioFileURL` overload uses `session.enqueueAudioFile(at:)` for memory-mapped reading.

`enrollSpeaker` is the priming flow: takes a feature-provider snapshot + timeline snapshot, drains pending audio, processes the enrollment audio with frames *not* recorded against `framesFedToModel`, picks the best speaker slot (preferring unnamed, with most speech activity), renames it, and rolls back to the snapshots on failure.

## Key Code
- [`LSEENDDiarizer.swift:17`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDDiarizer.swift#L17) — `public final class LSEENDDiarizer: Diarizer` and per-instance state declarations.
- [`LSEENDDiarizer.swift:79`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDDiarizer.swift#L79) — `initialize(model:)` is the model-hot-swap path; it rebuilds timeline + feature provider from the new metadata and calls `resetStreamingState`.
- [`LSEENDDiarizer.swift:117`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDDiarizer.swift#L117) — `addAudio<C: Collection>` delegates to `session.enqueueAudio` (which handles resampling internally).
- [`LSEENDDiarizer.swift:206`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDDiarizer.swift#L206) — `flush(recordFrames:finalizeOnCompletion:progressCallback:)` is the shared drain path used by both streaming and complete processing.
- [`LSEENDDiarizer.swift:250`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDDiarizer.swift#L250) — `input.warmupFrames = max(min(rightContext - framesFedToModel, chunkSize), 0)` — the warmup countdown that LS-EEND uses instead of a tentative/confirmed split.
- [`LSEENDDiarizer.swift:287`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDDiarizer.swift#L287) — `enrollSpeaker` snapshot/rollback flow with `oldSlots` set-difference to disambiguate fresh vs. existing speakers.

## Edge Cases & Failure Modes
- Throws `LSEENDError.notInitialized` from every gated path if `model == nil` or `session == nil`.
- `addAudio` for empty samples is a silent no-op (guarded by `samples.isEmpty`).
- `finalizeSession` is idempotent via the `finalized` flag.
- `enrollSpeaker` rolls back both the session (`session.rollback(to:)`) and timeline (`timeline.rollback(to:)`) snapshots if (a) flush produces no segments, or (b) the matched speaker slot was already named and `requireNewSpeaker` is set.
- `framesFedToModel` is only incremented when `recordFrames=true`; enrollment intentionally bypasses this so the warmup quota is not consumed by training audio.
- `processComplete(audioFileURL:)` reads + resamples the entire file synchronously via `session.enqueueAudioFile(at:)`; for very large files this can hold a lot in memory before draining. [REVIEW: contrast with `OfflineDiarizerManager.process(_:URL)` which uses a memory-mapped disk-backed source.]
- `cleanup()` clears the model, session, and metadata, but `timeline` is *not* reset — callers will still see the old segments until they replace the diarizer.

## Performance / Concurrency Notes
- The class is **not** `Sendable` or actor-isolated. Concurrent calls to `addAudio` and `process` from different threads will race on `framesFedToModel`, `finalized`, and the `timeline`. Callers must serialize.
- The underlying `LSEENDModel` holds its own `NSLock` around `model.prediction`, and `LSEENDFeatureProvider` has its own `NSLock` around queue mutation, so single-threaded use is correct, but cross-method serialization is the caller's responsibility.
- Compute units default to `.cpuOnly` per the model loader; the model docstring notes CPU is "fastest for this model" — CoreML's ANE path adds latency for this particular T-block topology.
- Warmup frames are stripped per-chunk inside `model.predict` (using `vDSP_mmov` over the probs tensor stride), so the predictions appended to the timeline are 1:1 with real audio time.

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/LS-EEND/LSEENDQueueTests.swift` — covers `StreamingChunkQueue` (used inside the feature provider) for left/right-context boundaries.
- `Tests/FluidAudioTests/Diarizer/LS-EEND/LSEENDFeatureProvider.swift` is itself a test helper (not a test target file) used by other LS-EEND tests.

## Changelogs
