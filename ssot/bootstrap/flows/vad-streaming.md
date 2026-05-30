---
id: vad-streaming
name: VAD Streaming (Event Emission)
trigger_type: event
trigger: Application feeds one 4096-sample chunk at a time via `VadManager.processStreamingChunk(_:state:config:returnSeconds:timeResolution:)`.
end_state: A `VadStreamingResult` is returned per chunk with optional `speechStart` / `speechEnd` events; the `VadState` is updated for the next call.
involves_features: [vad-manager]
linked_flows: [vad-segment-detection, model-download-and-load]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# VAD Streaming (Event Emission)

## TL;DR
Per-chunk Silero VAD with a hysteresis state machine. Each chunk yields a probability; entry into speech is gated by `threshold`, exit by `negativeThreshold` after `minSilenceSamples` elapsed. `speechStart` / `speechEnd` sample indices are corrected for `speechPadding` to align with chunk-boundary timestamps.

## Trigger & Preconditions
`VadManager.processStreamingChunk(_:state:config:returnSeconds:timeResolution:)`. Model loaded via `model-download-and-load.md`. Caller owns the `VadState` and threads it across calls.

## Stages
1. **Receive 4096-sample chunk** — caller pushes one chunk plus the prior `VadState`.
2. **Per-chunk inference** — `processChunk(_:inputState:)` runs the model with the threaded LSTM state, returns the new state and a probability: [`Sources/FluidAudio/VAD/VadManager.swift:162`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/VAD/VadManager.swift#L162).
3. **Compute working thresholds** — if `config.negativeThreshold` is set, the working positive threshold becomes `min(1.0, negativeThreshold + negativeThresholdOffset)`; otherwise default `threshold = 0.85`. Effective negative threshold clamped to `max(baseThreshold - offset, 0.01)`.
4. **State machine — entry** — when `probability >= threshold` and not currently in speech, emit `speechStart` with sample index `max(0, processedSamples - speechPad - chunkSampleCount)` so the timestamp aligns to the start of the chunk minus padding: [`Sources/FluidAudio/VAD/VadManager+Streaming.swift:11`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/VAD/VadManager+Streaming.swift#L11).
5. **State machine — exit** — when `probability < negativeThreshold` and in speech, record `silenceStart`; once `processedSamples - silenceStart >= minSilenceSamples`, emit `speechEnd` with sample index `max(0, silenceStart + speechPad - chunkSampleCount)`.
6. **Return result** — `VadStreamingResult` carries the optional `speechStart` and `speechEnd` events, the new `VadState`, and the raw probability.

## Outputs
- `VadStreamingResult { speechStart?, speechEnd?, state: VadState, probability: Float }`.

## Error Modes
- Inherits all `vad-segment-detection.md` error modes.
- Very-early speech (before `processedSamples - speechPad - chunkSampleCount >= 0`) clamps the start index to 0.
- Misuse of an old `VadState` across reset boundaries corrupts the entry/exit hysteresis.
- Model still in beta per docstring callout.

## Prompts and Models Used

## Usage Metrics

## Changelogs
