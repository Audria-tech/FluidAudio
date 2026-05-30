---
id: cosyvoice3-models
name: CosyVoice3 Models Bundle
repo: FluidAudio
status: active
linked_features: [cosyvoice3-tts-manager, cosyvoice3-pipeline-synthesize]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# CosyVoice3 Models Bundle

## TL;DR
`CosyVoice3Models` is the `Sendable` struct holding the four CoreML graphs of the CosyVoice3 inference pipeline: `prefill`, `decode`, `flow`, `hift`. Sendable conformance leans on `@preconcurrency import CoreML` — same pattern as `TtsModels`.

## Motivation — why it exists

## Context

## What It Does
`CosyVoice3Models` is a `public struct Sendable` with four `MLModel` stored properties: `prefill` (Qwen2 LM prefill), `decode` (Qwen2 LM stateless decode step), `flow` (Flow CFM, fp32 CPU/GPU only), `hift` (HiFT vocoder). All four are passed by reference into the `CosyVoice3Synthesizer`. The docstring documents the rationale for `Sendable` despite `MLModel` being reference-typed: predict surfaces are internally synchronized, and these instances are only handed to actors that own them for their lifetime, so crossing actor isolation is safe in practice.

## Key Code
- [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3Models.swift:11`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3Models.swift#L11) — `public struct CosyVoice3Models: Sendable`.
- [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3Models.swift:17`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3Models.swift#L17) — `init(prefill:decode:flow:hift:)`.
- [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3Constants.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3Constants.swift) — paired constants (embedDim, sample rate, Flow token cap).
- [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3Error.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3Error.swift) — `notInitialized`, `invalidShape`, etc.

## Edge Cases & Failure Modes
- Flow must be loaded with `.cpuAndGPU` (or `.cpuOnly`) — `.cpuAndNeuralEngine` triggers ANE+fp16 NaNs through fused `layer_norm`.
- HiFT falls back to CPU for ~12 sinegen / windowing ops regardless of requested compute units.
- All four models are required — there's no partial-bundle init path.
- [REVIEW: `Sendable` is technically unsafe for `MLModel` — the docstring acknowledges this and justifies via lifetime ownership; surfaces should not be invoked from multiple actors concurrently in case future CoreML versions tighten thread-safety contracts.]

## Test Coverage
- `Tests/FluidAudioTests/TTS/CosyVoice3ModelNameTests.swift`

## Changelogs
