---
id: style-tts2
name: StyleTTS2 Backend
repo: FluidAudio
status: active
linked_features: [tts-backend-protocol, audio-post-processor]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# StyleTTS2 Backend

## TL;DR
StyleTTS2 LibriTTS multi-speaker checkpoint as a 4-stage diffusion pipeline: `text_predictor` (fp16, ANE) → `diffusion_step_512` (fp16 CPU+GPU, ADPM2 + Karras + CFG, 5× per utterance) → `f0n_energy` (fp16, ANE) → `decoder` (fp32 CPU+GPU, HiFi-GAN). The Swift host handles espeak-ng phonemization, the diffusion sampler loop, cumsum→one-hot→matmul hard alignment, and bucket selection.

## Motivation — why it exists

## Context

## What It Does
`StyleTTS2Manager` is a `public actor` wrapping `StyleTTS2ModelStore` (downloads + holds the four CoreML graphs and config + vocab) and `StyleTTS2Synthesizer`. `initialize(progressHandler:)` ensures assets are downloaded, decodes and validates `config.json` against `StyleTTS2Constants` (so wrong-bundle / partial-download / version-mismatch errors surface here rather than as cryptic CoreML shape errors later). Models load lazily on first synthesis.

Pipeline responsibilities:
- `StyleTTS2Phonemizer` runs espeak-ng IPA phonemization + vocab lookup.
- The synthesizer drives the ADPM2 + Karras sampler around `diffusion_step` (5 steps × CFG → 10 forward passes), then computes hard alignment via cumsum-of-durations → one-hot → matmul.
- Bucket selection chooses the round-up token-length variant for `text_predictor` and round-up mel-frame variant for `decoder`.
- `StyleTTS2Sampler` is the standalone ADPM2 sampler.
- `StyleTTS2VoiceStyle` carries the per-voice style vectors.
- `StyleTTS2BundleConfig` is the `config.json` validation type.

Logger emits a beta warning at init: WER on long English phrases is elevated on the MiniMax corpus (~44% vs Kokoro 1.3%) — see `Documentation/TTS/Benchmarks.md`.

## Key Code
- [`Sources/FluidAudio/TTS/StyleTTS2/StyleTTS2Manager.swift:18`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/StyleTTS2/StyleTTS2Manager.swift#L18) — `public actor StyleTTS2Manager`.
- [`Sources/FluidAudio/TTS/StyleTTS2/Pipeline/StyleTTS2Synthesizer.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/StyleTTS2/Pipeline/StyleTTS2Synthesizer.swift) — sampler loop + alignment.
- [`Sources/FluidAudio/TTS/StyleTTS2/Pipeline/StyleTTS2Sampler.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/StyleTTS2/Pipeline/StyleTTS2Sampler.swift) — ADPM2 sampler.
- [`Sources/FluidAudio/TTS/StyleTTS2/Pipeline/StyleTTS2Phonemizer.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/StyleTTS2/Pipeline/StyleTTS2Phonemizer.swift) — espeak-ng IPA + vocab lookup.
- [`Sources/FluidAudio/TTS/StyleTTS2/Assets/StyleTTS2BundleConfig.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/StyleTTS2/Assets/StyleTTS2BundleConfig.swift) — `config.json` schema.

## Edge Cases & Failure Modes
- The fp32 HiFi-GAN decoder is required because SineGen phase saturation requires fp32 — using fp16 produces audible artifacts.
- Bundle config validation throws early — partial downloads or mismatched versions don't reach the synthesizer.
- [REVIEW: WER quality — README/manager doc cites ~44% WER on long English vs Kokoro's 1.3%; not recommended as a default backend.]

## Test Coverage
- `Tests/FluidAudioTests/TTS/StyleTTS2/StyleTTS2BundleConfigTests.swift`, `StyleTTS2SamplerTests.swift`, `StyleTTS2VocabTests.swift`, `StyleTTS2VoiceStyleTests.swift`.

## Changelogs
