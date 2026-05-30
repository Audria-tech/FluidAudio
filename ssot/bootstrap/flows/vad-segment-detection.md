---
id: vad-segment-detection
name: VAD Speech Segment Detection
trigger_type: user-action
trigger: Caller invokes `VadManager.segmentSpeech(_:config:)` (or the `from:totalSamples:config:` overload) with a complete audio buffer.
end_state: An array of `(startSample, endSample)` speech ranges is returned; manager state is unchanged.
involves_features: [vad-manager, vad-types, audio-converter]
linked_flows: [model-download-and-load]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# VAD Speech Segment Detection

## TL;DR
Silero VAD batch-style segmentation. Runs the unified CoreML VAD model over 4096-sample chunks, threading the 128-dim LSTM hidden/cell state, then applies the Silero `detectSpeechSampleRanges` state machine over per-chunk probabilities to produce a list of speech intervals with `maxSpeechDuration` splitting and adjacent-segment padding sharing.

## Trigger & Preconditions
`VadManager.segmentSpeech(_:config:)`. Model loaded via `model-download-and-load.md` (`DownloadUtils.loadModels(.vad, ...)`). Audio is `[Float]` at 16 kHz (other rates require pre-resampling via `AudioConverter`).

## Stages
1. **Normalize audio** — `processAudioSamples` accepts `URL` / `AVAudioPCMBuffer` / `[Float]`; non-Float paths route through `AudioConverter`: [`Sources/FluidAudio/Shared/AudioConverter.swift:14`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L14).
2. **Chunk the audio** — split into 4096-sample (256 ms @ 16 kHz) chunks, no overlap (the LSTM context is implicit in `hiddenState`/`cellState`).
3. **Per-chunk inference** — `processChunk(_:inputState:)` pads short tails by "repeat-last-sample" (not zero) to avoid energy distortion, captures the trailing 64 samples for the next call's context: [`Sources/FluidAudio/VAD/VadManager.swift:162`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/VAD/VadManager.swift#L162).
4. **Unified model call** — `processUnifiedModel` uses three pooled ANE-aligned buffers (`vad_audio_input`, `vad_hidden_state`, `vad_cell_state`) via `memoryOptimizer.getPooledBuffer`, clears with `vDSP_vclr`, copies inputs, runs `prediction` inside `autoreleasepool`, extracts outputs by exact + case-insensitive substring match (`vad_output`, `new_hidden_state`, `new_cell_state`): [`Sources/FluidAudio/VAD/VadManager.swift:208`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/VAD/VadManager.swift#L208).
5. **Thread state** — `VadState` (hidden + cell + last 64 samples) is updated and passed into the next chunk's `processChunk`.
6. **Run segmentation state machine** — `detectSpeechSampleRanges` walks per-chunk probabilities tracking `tempEnd` and a candidate-silence list (`possibleEnds`) for max-duration splitting. When current speech exceeds `maxSpeechSamples`, picks the longest candidate silence below `silenceThresholdForSplit` (or the longest overall, controlled by `useMaxPossibleSilenceAtMaxSpeech`): [`Sources/FluidAudio/VAD/VadManager+SpeechSegmentation.swift:71`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/VAD/VadManager+SpeechSegmentation.swift#L71).
7. **Filter + post-process** — drop segments shorter than `minSpeechSamples`; adjacent segments separated by < `2 * speechPad` share padding evenly (integer-division can cause a 1-sample overlap).

## Outputs
- `[VadSegment]` with `(startSample: Int, endSample: Int)` ranges (or seconds via `returnSeconds`).

## Error Modes
- `VadError.notInitialized` from `processChunk` if `vadModel == nil`.
- `VadError.modelLoadingFailed` if download didn't yield `sileroVadFile`.
- `VadError.modelProcessingFailed` if any of `vad_output` / `new_hidden_state` / `new_cell_state` is missing from the output.
- Empty audio → `[]`, no model call.
- Chunks longer than 4096 are silently truncated (`processedChunk.prefix(chunkSize)`).
- Integer-division on padding sharing may produce 1-sample segment overlaps.

## Prompts and Models Used

## Usage Metrics

## Changelogs
