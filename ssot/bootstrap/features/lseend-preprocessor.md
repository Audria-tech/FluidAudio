---
id: lseend-preprocessor
name: LS-EEND Preprocessor
repo: FluidAudio
status: active
linked_features: [lseend-diarizer, lseend-inference, lseend-types]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# LS-EEND Preprocessor

## TL;DR
`LSEENDFeatureProvider` is the streaming preprocessor: audio → resample → STFT → log10-mel → cumulative-mean-normalize → subsample+context chunking. It owns two `StreamingChunkQueue`s (audio + mel) and a per-step `LSEENDInput` carrying the model's recurrent state and pre-allocated MLMultiArrays. Includes snapshot/rollback for speaker enrollment.

## Motivation — why it exists

## Context

## What It Does
The init computes `flushSampleCount` from `(contextMels + convDelay * subsampling) * hopLength + nFFT/2` — the trailing silence required to flush every real frame through STFT + mel ±context + CNN right-lookahead. It also builds a `decoderMask` of length `convDelay + chunkSize` (zeros for the convDelay warmup, ones for real frames), initializes `AudioMelSpectrogram` with `nMels`/`nFFT`/`hopLength`/`winLength`/`preemph=0`/`logFloor=1e-10`/`logFloorMode=.clamped`/`windowPeriodic=true`, and creates a `LSEENDInput` from the metadata.

`enqueueAudio(_:withSampleRate:eagerPreprocessing:)` appends to `audioQueue` (resampling via `AudioConverter.resample` if needed) and optionally calls `processAudioQueue` immediately. `enqueueAudioFile(at:)` reads + resamples via `converter.resampleAudioFile` and always preprocesses. `drainRightContextWithSilence(flush:)` appends `flushSampleCount` zeros plus a chunk-boundary shortfall (`(chunk - overCtx % chunk) % chunk` more zeros) and optionally flushes — this guarantees `popAllChunks` consumes every real sample plus the trailing silence.

`processAudioQueue` pops all ready audio, runs `melSpectrogram.computeFlatTransposed(audio:lastAudioSample:0,paddingMode:.prePadded,expectedFrameCount:nil)`, rescales the natural-log mel features to log10 via `vDSP_vsmul` with `log10Scale = 1/log(10)`, and applies cumulative-mean normalization frame-by-frame using `vDSP_vintb` (interpolated blend at `alpha = 1/cmnCount`) followed by `vDSP_vsub` to subtract the running mean. The CMN math: `µ[k] = µ[k-1] + (mel[k] - µ[k-1]) / k`, then `mel[k] ← mel[k] - µ[k]`.

`emitNextChunk()` advances `decoderMaskEnd` by `chunkFrames`, pops the next mel chunk, and loads the input with the appropriate decoder-mask slice and `warmupFrames = min(decoderMask.count - decoderMaskEnd, chunkFrames)`. Snapshot/rollback exists for enrollment: `takeSnapshot()` deep-copies the state via `state.copy()` plus all four sub-buffers; `rollback(to:keepingState:)` restores everything (optionally keeping the live recurrent state). `reset()` zeros CMN, queues, and recurrent state.

`StreamingChunkQueue` is the underlying ring-ish buffer (lazy-trimming): supports `append`, `popNextChunk` (returns a fixed-width slice including left+right context), `popAllChunks` (returns all available chunks fused together with edge contexts), `readyChunks` count, and `reset`.

## Key Code
- [`LSEENDPreprocessor.swift:46`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDPreprocessor.swift#L46) — `init(from:restoringFrom:)` — sets up flush math, decoder mask, mel spectrogram, and optional snapshot restore.
- [`LSEENDPreprocessor.swift:158`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDPreprocessor.swift#L158) — `drainRightContextWithSilence(flush:)` trailing-silence + chunk-boundary alignment math.
- [`LSEENDPreprocessor.swift:185`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDPreprocessor.swift#L185) — `emitNextChunk()` decoder-mask slice + warmup computation.
- [`LSEENDPreprocessor.swift:249`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDPreprocessor.swift#L249) — `processAudioQueue` log10 rescale + per-frame CMN with `vDSP_vintb`/`vDSP_vsub`.
- [`LSEENDPreprocessor.swift:284`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDPreprocessor.swift#L284) — `StreamingChunkQueue` declaration with lazy trimming (`if buffer.count + newElements.count > buffer.capacity { removeFirst(head); head = 0 }`).

## Edge Cases & Failure Modes
- Snapshot uses `~Copyable Snapshot { let state: LSEENDState; ... }` with `consume`-style moves. `Snapshot` is non-copyable to ensure exactly-once restoration.
- `drainRightContextWithSilence` is safe to call multiple times; each call appends more silence (no idempotency guard). [REVIEW: callers like `finalizeSession` only call once, but a misuse could inflate the audio queue indefinitely.]
- `emitNextChunk` advances `decoderMaskEnd` before checking for a chunk, but then bails via `guard let rawChunk = melQueue.popNextChunk() else { return nil }` — this can leave `decoderMaskEnd` advanced without a corresponding model call. [REVIEW: confirm if this affects later warmup math.] Actually re-reading: `decoderMaskEnd = min(decoderMaskEnd + chunkFrames, decoderMask.count)` runs *after* `popNextChunk` succeeds, so this is correct.
- CMN is sequential by definition; no parallelization opportunity. `cmnCount` grows unboundedly for long streams but `1/Float(cmnCount)` underflows gracefully toward zero (the mean stops updating).
- The streaming chunk queue's lazy trimming amortizes the cost but causes O(n) work on each trim event.

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/LS-EEND/LSEENDQueueTests.swift` — `StreamingChunkQueue` left/right-context boundaries.

## Changelogs
