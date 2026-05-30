---
id: tts-compute-unit-preset
name: TTS Compute Unit Preset
repo: FluidAudio
status: active
linked_features: [kokoro-ane, tts-backend-protocol, fluidaudio-cli-bundle]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# TTS Compute Unit Preset

## TL;DR
`TtsComputeUnitPreset` is the uniform compute-unit knob flipped by the CLI / benchmark harness. Maps the four canonical values (`default`, `all-ane`, `cpu-and-gpu`, `cpu-only`) to either a concrete `MLComputeUnits` (uniform overrides) or `nil` for "let the backend keep its empirical per-stage mapping". Used because each backend has its own per-stage struct (`KokoroAneComputeUnits`, etc.) but the CLI only takes one flag.

## Motivation — why it exists

## Context

## What It Does
`TtsComputeUnitPreset` is a `public enum: String, Sendable, CaseIterable` with four cases:
- `.default` — empirically-tuned per-stage mapping; `uniformUnits` returns `nil`.
- `.allAne` — force every stage to `.cpuAndNeuralEngine`.
- `.cpuAndGpu` — force every stage to `.cpuAndGPU`. Useful as a latency baseline when ANE compile cache is cold.
- `.cpuOnly` — force every stage to `.cpuOnly`. Fallback / debugging baseline.

`init?(cliValue:)` accepts the kebab-case CLI form plus aliases (`ane`/`neural-engine` → `.allAne`, `cpuandgpu`/`gpu` → `.cpuAndGpu`, `cpu`/`cpuonly` → `.cpuOnly`). `cliValue` returns the canonical kebab-case form for round-tripping through logs and JSON reports.

Backends opt in by adding `init(preset: TtsComputeUnitPreset)` to their per-stage compute-units struct — `KokoroAneComputeUnits` is the reference implementation cited in the docstring.

## Key Code
- [`Sources/FluidAudio/TTS/Shared/TtsComputeUnitPreset.swift:16`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Shared/TtsComputeUnitPreset.swift#L16) — `public enum TtsComputeUnitPreset`.
- [`Sources/FluidAudio/TTS/Shared/TtsComputeUnitPreset.swift:39`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Shared/TtsComputeUnitPreset.swift#L39) — `uniformUnits: MLComputeUnits?`.
- [`Sources/FluidAudio/TTS/Shared/TtsComputeUnitPreset.swift:51`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Shared/TtsComputeUnitPreset.swift#L51) — `init?(cliValue:)` with aliases.
- [`Sources/FluidAudio/TTS/Shared/TtsComputeUnitPreset.swift:64`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Shared/TtsComputeUnitPreset.swift#L64) — `cliValue` canonical form.

## Edge Cases & Failure Modes
- `init?(cliValue:)` returns `nil` for unrecognized strings so callers can surface a usage error.
- `.default` does not return a concrete `MLComputeUnits` — backends must explicitly check `if let units = preset.uniformUnits` and fall through to their internal default otherwise.
- [REVIEW: not all backends adopt this preset — `KokoroTtsManager` and the experimental Magpie/CosyVoice3 paths take `MLComputeUnits` directly. Adoption is gradual.]

## Test Coverage
- `Tests/FluidAudioTests/TTS/TtsComputeUnitPresetTests.swift`

## Changelogs
