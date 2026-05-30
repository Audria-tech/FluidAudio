---
id: vad-types
name: VAD Types
repo: FluidAudio
status: active
linked_features: [vad-manager]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# VAD Types

## TL;DR
Value types for the VAD pipeline: `VadConfig` (model threshold + compute units), `VadSegmentationConfig` (Silero hysteresis + segmentation knobs with explicit precondition/assert validation), `VadState`/`VadStreamState` (LSTM hidden+cell+context), `VadResult`, `VadSegment`, `VadStreamEvent`, `VadStreamResult`, `VadError`.

## Motivation — why it exists

## Context

## What It Does
`VadConfig` is `Sendable`: `defaultThreshold` (0.85), `debugMode`, `computeUnits` (default `.cpuAndNeuralEngine`). `VadSegmentationConfig` is `Sendable` with explicit `precondition` checks on non-negativity (will trap at runtime in both debug and release) and `assert` checks for logical consistency (debug-only). Fields: `minSpeechDuration` (0.15s), `minSilenceDuration` (0.75s), `maxSpeechDuration` (14.0s — tuned for ASR's typical 15s max), `speechPadding` (0.1s), `silenceThresholdForSplit` (0.3), `negativeThreshold` (optional override), `negativeThresholdOffset` (0.15), `minSilenceAtMaxSpeech` (0.098s), `useMaxPossibleSilenceAtMaxSpeech` (true). `effectiveNegativeThreshold(baseThreshold:)` returns the override if set, otherwise `max(baseThreshold - negativeThresholdOffset, 0.01)`.

`VadState` carries `hiddenState: [Float]` (128-dim), `cellState: [Float]` (128-dim), `context: [Float]` (64-dim trailing audio); `static contextLength = 64`. `.initial()` returns zero-init state. `VadResult` carries `probability`, `isVoiceActive`, `processingTime`, and the new `outputState`. `VadSegment` carries `startTime`/`endTime` plus sample-conversion helpers (`startSample(sampleRate:)`, `endSample`, `sampleCount`).

`VadStreamState` adds `triggered: Bool`, `tempEndSample: Int?`, `processedSamples: Int` on top of `modelState`. `VadStreamEvent` has `Kind.{speechStart,speechEnd}` plus `sampleIndex` and optional `time`. `VadStreamResult` carries `state` + optional `event` + `probability`.

`VadError`: `.notInitialized`, `.modelLoadingFailed`, `.modelProcessingFailed(String)`.

## Key Code
- [`VadTypes.swift:4`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/VAD/VadTypes.swift#L4) — `VadConfig` declaration; `defaultThreshold=0.85`.
- [`VadTypes.swift:48`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/VAD/VadTypes.swift#L48) — `VadSegmentationConfig.init` — `precondition` (release-fatal) + `assert` (debug-only) validation block.
- [`VadTypes.swift:85`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/VAD/VadTypes.swift#L85) — `effectiveNegativeThreshold` with 0.01 lower bound.
- [`VadTypes.swift:93`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/VAD/VadTypes.swift#L93) — `VadState` with 64-sample context constant.

## Edge Cases & Failure Modes
- `VadSegmentationConfig.init` will **trap** (release-fatal `precondition`) for: negative `minSpeechDuration`/`minSilenceDuration`/`speechPadding`/`negativeThresholdOffset`/`minSilenceAtMaxSpeech`; non-positive `maxSpeechDuration`; `silenceThresholdForSplit` outside `[0, 1]`; `negativeThreshold` outside `[0, 1]`.
- Debug `assert`s warn (but don't trap in release) if `minSpeechDuration > maxSpeechDuration`, `minSilenceDuration > maxSpeechDuration`, `speechPadding > minSpeechDuration`, or `negativeThreshold > silenceThresholdForSplit`.
- `VadState.initial()` returns 128-dim zero hidden + cell + 64-dim zero context (assumes Silero's standard model dimensions).
- `VadSegment.sampleCount(sampleRate:)` can return a negative value if `endTime < startTime` (no validation).
- `VadStreamEvent.time` is optional; segmentation extension fills it when `returnSeconds=true`.

## Test Coverage
- `Tests/FluidAudioTests/VAD/VadTests.swift`, `VadStreamingTests.swift`, `VadSegmentationTests.swift` — types exercised across all VAD code paths.

## Changelogs
