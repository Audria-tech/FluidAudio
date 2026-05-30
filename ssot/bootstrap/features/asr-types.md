---
id: asr-types
name: ASR Types (ASRConfig, ASRResult, TokenTiming, ASRError)
repo: FluidAudio
status: active
linked_features:
  - sliding-window-asr-manager
  - tdt-decoder
  - custom-vocabulary-rescorer
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# ASR Types (ASRConfig, ASRResult, TokenTiming, ASRError)

## TL;DR
`AsrTypes.swift` defines the public configuration / result / error types that surface from the TDT Parakeet path: `ASRConfig` (sample rate, TDT config, encoder hidden size, parallel chunk concurrency, streaming threshold), `ASRResult` (text + confidence + duration + timings + CTC metadata), `TokenTiming` (per-token start/end/confidence), and `ASRError` (the public error cases used across the manager).

## Context

## What It Does
`ASRConfig` parameterises the long-form / streaming pipeline: `parallelChunkConcurrency` (default 4) caps concurrent CoreML inferences for offline long-audio chunking; `streamingEnabled` + `streamingThreshold` (default 480 000 samples ≈ 30 s) decide when the long-form path switches to chunked streaming for memory bounding. `ASRResult` carries `rtfx` (real-time factor) as a computed property and exposes `withRescoring(text:detected:applied:)` which returns a copy with rescored text and CTC term lists — used by `SlidingWindowAsrManager` when vocabulary boosting rewrites a confirmed chunk. `TokenTiming` is `Codable + Sendable` so transcripts can be persisted. `ASRError.fileAccessFailed(URL, Error)` carries the failing URL so CLI tools can surface it to the user.

## Key Code
- [`AsrTypes.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/AsrTypes.swift#L5) — `ASRConfig` struct + `streamingThreshold` default.
- [`AsrTypes.swift:47`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/AsrTypes.swift#L47) — `ASRResult` `Codable, Sendable` with `ctcDetectedTerms` / `ctcAppliedTerms`.
- [`AsrTypes.swift:86`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/AsrTypes.swift#L86) — `withRescoring(text:detected:applied:)` clones with updated CTC metadata.
- [`AsrTypes.swift:100`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/AsrTypes.swift#L100) — `TokenTiming` struct.
- [`AsrTypes.swift:121`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/AsrTypes.swift#L121) — `ASRError` cases with `errorDescription`.

## Edge Cases & Failure Modes
- `parallelChunkConcurrency` is clamped to `max(1, _)` — zero or negative values are silently corrected.
- `rtfx` divides by `processingTime` — if `processingTime == 0` the result is `inf` / `NaN` (no guard).
- `ASRError.invalidAudioData` description hard-codes "300ms of 16kHz audio" — when the lower bound is changed via `ASRConstants.minimumRequiredSamples`, the human message can drift.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Parakeet/ASRConfigTests.swift`.

## Changelogs
