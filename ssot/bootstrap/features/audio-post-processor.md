---
id: audio-post-processor
name: Audio Post-Processor (De-Esser)
repo: FluidAudio
status: active
linked_features: [kokoro-tts-manager, pocket-tts-manager]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Audio Post-Processor (De-Esser)

## TL;DR
Shared `AudioPostProcessor` enum exposing audio post-processing utilities. Currently provides `deEss(...)` — a biquad high-shelf filter that reduces sibilants above a cutoff (default 6 kHz, -3 dB). Used by Kokoro and PocketTTS managers when `deEss: true` (default).

## Motivation — why it exists

## Context

## What It Does
`AudioPostProcessor.deEss(_:sampleRate:cutoffHz:reductionDb:)` applies a biquad high-shelf filter in place to the input `[Float]` buffer. Coefficients are computed from the analog prototype `H(s) = A * (s² + sqrt(A)/Q * s + A) / (A*s² + sqrt(A)/Q * s + 1)` with Butterworth `Q = 0.707`. Pre-normalizes by `a0`, then runs direct-form II transposed in a single forward pass. Defaults: `sampleRate = 24000` (matches Kokoro/PocketTTS output), `cutoffHz = 6000`, `reductionDb = -3.0`.

## Key Code
- [`Sources/FluidAudio/TTS/Shared/AudioPostProcessor.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Shared/AudioPostProcessor.swift#L5) — `public enum AudioPostProcessor`.
- [`Sources/FluidAudio/TTS/Shared/AudioPostProcessor.swift:15`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Shared/AudioPostProcessor.swift#L15) — `deEss(_:sampleRate:cutoffHz:reductionDb:)`.
- [`Sources/FluidAudio/TTS/Shared/AudioPostProcessor.swift:24`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Shared/AudioPostProcessor.swift#L24) — biquad high-shelf coefficient derivation.
- [`Sources/FluidAudio/TTS/Shared/AudioPostProcessor.swift:53`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Shared/AudioPostProcessor.swift#L53) — direct-form II transposed forward pass.

## Edge Cases & Failure Modes
- Returns immediately if `samples.count <= 2` (filter state would dominate output).
- Mutates the buffer in place; callers are responsible for any necessary copy.
- Filter is mono — multichannel callers must process each channel independently.
- [REVIEW: filter does not import Accelerate vDSP for the forward pass — single scalar loop. For long buffers (multi-second), a vDSP biquad would be cheaper.]

## Test Coverage
- `Tests/FluidAudioTests/TTS/AudioPostProcessorTests.swift`

## Changelogs
