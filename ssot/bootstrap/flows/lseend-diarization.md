---
id: lseend-diarization
name: LS-EEND Streaming Diarization
trigger_type: event
trigger: Application pushes audio via `LSEENDDiarizer.addAudio(samples:sourceSampleRate:)` and calls `process()` to drain.
end_state: A `DiarizerTimelineUpdate` is returned with finalized speaker predictions (no tentative split); recurrent state is updated in place.
involves_features: [lseend-diarizer, lseend-inference, lseend-preprocessor]
linked_flows: [model-download-and-load]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# LS-EEND Streaming Diarization

## TL;DR
Long-form Streaming End-to-End Neural Diarization. Single CoreML T-block per chunk; the preprocessor handles STFT → log10-mel → cumulative-mean normalization → subsample+context queue. Recurrent state (`probs` + 6 enc/dec/cnn cache tensors) is updated in place across calls. Default compute units are `.cpuOnly` (model docstring: CPU fastest for this T-block topology).

## Trigger & Preconditions
`LSEENDDiarizer.initialize(model:)` or `initialize(variant:stepSize:...)` must have loaded a model. The class is **not** actor-isolated; underlying `LSEENDModel` and `LSEENDFeatureProvider` each carry an `NSLock` so single-threaded use is safe. Audio fed via `addAudio` (resampling happens inside the feature provider).

## Stages
1. **Enqueue audio** — `addAudio<C: Collection>` delegates to `session.enqueueAudio(...)` which resamples via `AudioConverter.resample` if needed, appends to the audio queue, optionally runs eager preprocessing: [`Sources/FluidAudio/Diarizer/LS-EEND/LSEENDDiarizer.swift:117`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDDiarizer.swift#L117).
2. **Preprocess to mel** — `LSEENDFeatureProvider.processAudioQueue` pops ready audio, runs `AudioMelSpectrogram.computeFlatTransposed(...paddingMode: .prePadded)`, rescales natural log → log10 via `vDSP_vsmul` with `log10Scale = 1/log(10)`: [`Sources/FluidAudio/Diarizer/LS-EEND/LSEENDPreprocessor.swift:249`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDPreprocessor.swift#L249).
3. **Cumulative mean normalize** — per-frame CMN: `µ[k] = µ[k-1] + (mel[k] - µ[k-1]) / k` via `vDSP_vintb` (blend at `alpha = 1/cmnCount`); then `mel[k] ← mel[k] - µ[k]` via `vDSP_vsub`. `cmnCount` grows unboundedly; `1/Float(cmnCount)` underflows toward zero gracefully so the mean stops updating in very long streams.
4. **Drain loop entry** — `process()` calls `flush(...)`. Each iteration emits one mel chunk via `emitNextChunk()`, advances `decoderMaskEnd` by `chunkFrames`, populates `LSEENDInput` with the next decoder-mask slice and computes `warmupFrames = min(decoderMask.count - decoderMaskEnd, chunkFrames)`: [`Sources/FluidAudio/Diarizer/LS-EEND/LSEENDPreprocessor.swift:185`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDPreprocessor.swift#L185).
5. **Inference** — `LSEENDModel.predict(from:)` runs inside `autoreleasepool` + per-instance `NSLock`. Extracts 7 outputs (probs + 6 recurrent caches), updates `LSEENDState` in place, and copies the probs tensor (skipping `warmupFrames` rows) into a flat `[outputFrames * outputSpeakers]` array via stride-aware `vDSP_mmov`. Throws if the probs tensor's innermost stride isn't 1: [`Sources/FluidAudio/Diarizer/LS-EEND/LSEENDInference.swift:85`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDInference.swift#L85).
6. **Append predictions** — `flush` records `framesFedToModel` (only when `recordFrames=true` — enrollment bypasses this to preserve the warmup quota), then appends the predictions to the timeline via `timeline.addPredictions(finalizedPredictions:tentativePredictions:)`. LS-EEND emits finalized frames directly — no tentative split: [`Sources/FluidAudio/Diarizer/LS-EEND/LSEENDDiarizer.swift:206`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDDiarizer.swift#L206).
7. **Finalize** — `finalizeSession()` calls `session.drainRightContextWithSilence()` (pushes `flushSampleCount` zeros plus chunk-boundary shortfall) to flush every real frame, runs one more `process()`, marks timeline finalized.

## Outputs
- `DiarizerTimelineUpdate` carrying cumulative finalized `DiarizerPrediction` lists.

## Error Modes
- `LSEENDError.notInitialized` from every gated path if `model == nil` or `session == nil`.
- `LSEENDError.initializationFailed("No `config` found in model metadata")` if `creatorDefinedKey.config` is malformed.
- `LSEENDError.inferenceFailed("Probs innermost stride must be 1...")` if CoreML produces a non-contiguous probs tensor.
- `LSEENDError.invalidInputSize` if a buffer's count doesn't match the destination's.
- `processComplete(audioFileURL:)` reads + resamples the entire file synchronously — large files held in memory.
- Concurrent callers must serialize externally; `framesFedToModel`/`finalized`/`timeline` race silently otherwise.
- `cleanup()` clears model + session but **not** the timeline — old segments still visible until manager replaced.

## Prompts and Models Used

## Usage Metrics

## Changelogs
