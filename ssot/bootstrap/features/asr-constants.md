---
id: asr-constants
name: ASR Constants
repo: FluidAudio
status: active
linked_features: [audio-mel-spectrogram]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# ASR Constants

## TL;DR
Public namespace of numeric constants used across the Parakeet ASR pipeline — sample rate, encoder window length, mel hop / subsampling factors, encoder/decoder hidden sizes, frame-vs-time conversions, and a few benchmarking thresholds. Two small helper functions translate between samples and encoder frames.

## Motivation — why it exists

## Context

## What It Does
`sampleRate = 16000`, `maxDurationSeconds = 15.0`, `maxModelSamples = 240_000` define the CoreML encoder window. `minimumAudioDurationSeconds = 0.3` and `minimumRequiredSamples(forSampleRate:)` bound the lower end. `melHopSize = 160`, `encoderSubsampling = 8`, `samplesPerEncoderFrame = 1280` (= 80 ms at 16 kHz), `secondsPerEncoderFrame = 0.08` describe the frame grid. Hidden sizes — `encoderHiddenSize = 1024`, `decoderHiddenSize = 640` — match Parakeet-TDT. `punctuationTokens = [7883, 7952, 7948]` are the period/?/!. `standardOverlapFrames = 25` (2.0 s) is the sliding-window overlap. `highWERThreshold = 0.15`, `minConfidence = 0.1`, `maxConfidence = 1.0` are benchmarking / scoring tunables. `calculateEncoderFrames(from:)` is ceiling-division from samples to encoder frames.

## Key Code
- [`Sources/FluidAudio/Shared/ASRConstants.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ASRConstants.swift#L5) — `sampleRate = 16_000`
- [`Sources/FluidAudio/Shared/ASRConstants.swift:9`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ASRConstants.swift#L9) — encoder window (`maxDurationSeconds = 15.0`, `maxModelSamples = 240_000`)
- [`Sources/FluidAudio/Shared/ASRConstants.swift:18`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ASRConstants.swift#L18) — mel/encoder frame grid (`melHopSize`, `encoderSubsampling`, `samplesPerEncoderFrame`, `secondsPerEncoderFrame`)
- [`Sources/FluidAudio/Shared/ASRConstants.swift:40`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ASRConstants.swift#L40) — `punctuationTokens` — hard-coded vocab IDs for period / `?` / `!`
- [`Sources/FluidAudio/Shared/ASRConstants.swift:54`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ASRConstants.swift#L54) — `calculateEncoderFrames(from:)` — `ceil(samples / samplesPerEncoderFrame)`

## Edge Cases & Failure Modes
- `punctuationTokens` are baked-in vocab IDs for the **Parakeet** vocabulary; using these constants with a different model (e.g. CTC zh-CN, Qwen3) would mis-classify. [REVIEW: `punctuationTokens` are model-specific (Parakeet) but live in a "shared" namespace — risk of cross-model misuse]
- `maxDurationSeconds = 15.0` is the encoder bucket cap for v2/v3 — does **not** apply to Cohere Transcribe (35 s) or Qwen3 ASR. Callers must know which model's constants apply.
- `encoderHiddenSize = 1024` and `decoderHiddenSize = 640` describe Parakeet TDT only; CTC variants and the 110m hybrid have different sizes.
- `standardOverlapFrames = 25` baked-in — non-standard overlap configurations must use their own constant.

## Test Coverage
No dedicated unit tests; constants are referenced throughout the ASR pipeline tests.

## Changelogs
