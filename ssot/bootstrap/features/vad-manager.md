---
id: vad-manager
name: VAD Manager
repo: FluidAudio
status: active
linked_features: [vad-types]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# VAD Manager

## TL;DR
`VadManager` is the Silero-VAD-based voice-activity-detection actor. It runs a unified CoreML model that takes a 4096-sample chunk (256 ms at 16 kHz) plus 64-sample context and 128-dim LSTM hidden/cell state, and produces a probability. Three extension files add streaming (start/end event emission with hysteresis) and Silero-style speech-segmentation timestamps.

## Motivation — why it exists

## Context

## What It Does
The base type is a `public actor VadManager`. Constants: `chunkSize=4096`, `contextSize=64`, `stateSize=128`, `modelInputSize=chunkSize+contextSize=4160`, `sampleRate=16000`. The actor holds an optional `MLModel` (lazy-loaded), an `AudioConverter` for resampling, and an `ANEMemoryOptimizer` for buffer pooling.

`process(_:URL)` / `process(_:AVAudioPCMBuffer)` / `process(_:[Float])` all funnel through `processAudioSamples`, which splits audio into `chunkSize`-stride chunks (no overlap — the LSTM context is implicit in `hiddenState`/`cellState`) and calls `processChunk` sequentially, threading the `VadState` (hidden + cell + last 64 samples of context) chunk-to-chunk.

`processChunk(_:inputState:)` pads short chunks via "repeat-last-sample" (not zero) to avoid energy distortion, captures the trailing 64 samples for the next call's context, and runs `processUnifiedModel`. The model call uses `memoryOptimizer.getPooledBuffer` with three keys (`vad_audio_input`, `vad_hidden_state`, `vad_cell_state`) so consecutive calls reuse ANE-aligned `MLMultiArray`s. Inputs are cleared with `vDSP_vclr`, populated via `memoryOptimizer.optimizedCopy`, fed through an `MLDictionaryFeatureProvider`, and `featureValue(in:matchingSubstrings:)` extracts outputs by both exact match and case-insensitive substring (the model's actual output names may carry suffixes).

**Streaming extension** (`VadManager+Streaming.swift`): `processStreamingChunk(_:state:config:returnSeconds:timeResolution:)` runs one chunk through `processChunk`, then a Silero-style state machine over `(probability, threshold, negativeThreshold)`:
- On entry (`probability >= threshold`): emit `speechStart` with sample index `max(0, processedSamples - speechPad - chunkSampleCount)`.
- On exit (`probability < negativeThreshold` after `minSilenceSamples` elapsed): emit `speechEnd` with sample index `max(0, silenceStart + speechPad - chunkSampleCount)`.
- Thresholds: if `config.negativeThreshold` is set, the working positive threshold is `min(1.0, negativeThreshold + negativeThresholdOffset)`; otherwise the manager's default threshold (0.85) is used.

**Speech-segmentation extension** (`VadManager+SpeechSegmentation.swift`): `segmentSpeech(_:config:)` and `segmentSpeech(from:totalSamples:config:)` run the full audio through the model (or accept pre-computed `VadResult`s) and apply the Silero `detectSpeechSampleRanges` algorithm:
- Walks per-chunk probabilities; tracks `tempEnd` and a candidate-silence list (`possibleEnds`) for max-duration splitting.
- When the current speech exceeds `maxSpeechSamples`, picks the longest candidate silence below `silenceThresholdForSplit` (or falls back to the longest overall, controlled by `useMaxPossibleSilenceAtMaxSpeech`).
- Filters segments shorter than `minSpeechSamples`, then post-processes adjacent segments to share `speechPad` evenly when the gap is < `2*speechPad`.

## Key Code
- [`VadManager.swift:14`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/VAD/VadManager.swift#L14) — `public actor VadManager` declaration; chunk/context/state/sample-rate constants.
- [`VadManager.swift:162`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/VAD/VadManager.swift#L162) — `processChunk(_:inputState:)` is the per-chunk inference. Repeat-last padding for short chunks, vDSP/ANE-aligned buffer reuse, no normalization (preserves original amplitude info).
- [`VadManager.swift:208`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/VAD/VadManager.swift#L208) — `processUnifiedModel` — `autoreleasepool` wrapping for ANE buffer lifecycle, three pooled buffers, flexible output name resolution.
- [`VadManager+Streaming.swift:11`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/VAD/VadManager+Streaming.swift#L11) — `processStreamingChunk` event emission with `speechPadding` adjustment subtracting the chunk size to align with chunk-boundary timestamps.
- [`VadManager+SpeechSegmentation.swift:71`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/VAD/VadManager+SpeechSegmentation.swift#L71) — `detectSpeechSampleRanges` — the Silero-port state machine for offline segmentation with `maxSpeechDuration` splitting and adjacent-segment padding sharing.

## Edge Cases & Failure Modes
- Throws `VadError.notInitialized` from `processChunk` if `vadModel == nil`.
- Throws `VadError.modelLoadingFailed` when downloading from the registry fails (no `sileroVadFile` in the loaded models dict).
- Throws `VadError.modelProcessingFailed` if any required output (`vad_output`, `new_hidden_state`, `new_cell_state`) is missing from the model output.
- Empty audio → `processAudioSamples` returns `[]` without any model call.
- Chunks longer than `chunkSize` are silently truncated (taken via `processedChunk.prefix(chunkSize)`).
- The internal logic-only initializer (`init(skipModelLoading:config:)`) leaves `vadModel = nil`; any `processChunk` call will throw `.notInitialized`. Used for unit tests of the segmentation state machine.
- `streamingStateMachine` clamps sample indices to 0 to handle the case where `processedSamples - speechPad - chunkSampleCount < 0` (very-early speech start).
- Silero post-processing: when the gap between two adjacent segments is < `2 * speechPad`, each side gets `silence / 2` instead of full padding — segments may overlap by 1 sample due to integer division.
- `effectiveNegativeThreshold` clamps to `max(baseThreshold - offset, 0.01)` so the negative threshold can never collapse below 0.01 even with extreme offsets.
- The status note in the docstring: "**Beta Status**: This VAD implementation is currently in beta."

## Performance / Concurrency Notes
- `public actor VadManager` provides Swift 6 strict-concurrency safety automatically; all methods are isolated.
- `processChunk` calls `model.prediction` synchronously inside an `autoreleasepool` — this is fine on iOS 17+/macOS 14+ since the actor serializes calls anyway. [REVIEW: contrast with `OfflineDiarizerManager.prewarmSegmentationModel` which uses async prediction explicitly.]
- The three pooled ANE buffers (`vad_audio_input`, `vad_hidden_state`, `vad_cell_state`) survive across calls; only the buffer contents are rewritten. This is the primary perf win for streaming.
- `processAudioSamples` is sequential by design (state must thread through chunks); there is no batched fast path.
- The model is loaded via `DownloadUtils.loadModels(.vad, ...)` with `config.computeUnits` — default is `.cpuAndNeuralEngine`.
- Streaming/segmentation extensions are pure logic with no model calls per state-machine step, so they are CPU-cheap.

## Test Coverage
- `Tests/FluidAudioTests/VAD/VadTests.swift` — `processChunk`, `processAudioSamples`, padding behavior.
- `Tests/FluidAudioTests/VAD/VadStreamingTests.swift` — `processStreamingChunk` event emission, hysteresis boundaries.
- `Tests/FluidAudioTests/VAD/VadSegmentationTests.swift` — `segmentSpeech` and `detectSpeechSampleRanges` corner cases including `maxSpeechDuration` splitting.
- `Tests/FluidAudioTests/TestHelpers/VadTestHelpers.swift` — shared fixtures.

## Changelogs
