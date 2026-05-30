---
id: cosyvoice3-pipeline-synthesize
name: CosyVoice3 Pipeline (Synthesize)
repo: FluidAudio
status: active
linked_features: [cosyvoice3-tts-manager, cosyvoice3-pipeline-preprocess, cosyvoice3-models]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# CosyVoice3 Pipeline (Synthesize)

## TL;DR
CosyVoice3 inference: prefill+decode (Qwen2 LM with external KV cache), Flow CFM (fp32 CPU/GPU only — fp16+ANE NaNs through fused `layer_norm`), HiFT vocoder (~12 sinegen ops fall back to CPU). `CosyVoice3RasSampler` is the speech-token sampler; `CosyVoice3SpeechEmbeddings` mmaps the speech embedding table.

## Motivation — why it exists

## Context

## What It Does
**`CosyVoice3Synthesizer`** — the actor orchestrating one synthesis. Takes a `CosyVoice3FrontendFixture` (whether from a real fixture file or assembled in-memory by the manager's chunked text path). Runs LM prefill (consumes `lmInputEmbeds`, returns initial KV), then a stateless decode loop (external KV cache rather than `MLState`) until EOS or `maxNewTokens`, calling the RAS sampler each step. Speech tokens then go into the Flow CFM to produce a mel spectrogram, which the HiFT vocoder renders to fp32 PCM at the target sample rate.

**`CosyVoice3RasSampler`** — Speech-token sampling at each LM decode step. Honors `seed` and `maxNewTokens` from `CosyVoice3SynthesisOptions`. Emits both decoded tokens and an `finishedOnEos` flag.

**`CosyVoice3SpeechEmbeddings`** — fp32 safetensors mmap of the speech embedding rows (`init(url:)`), looked up per decoded token.

**`CosyVoice3Types`** — result + options types: `CosyVoice3SynthesisResult` (samples, sampleRate, generatedTokenCount, decodedTokens, finishedOnEos), `CosyVoice3SynthesisOptions` (maxNewTokens, seed, disableAutoChunking), `CosyVoice3ParityOptions` (parity replay knobs).

The manager's text-mode path packages chunked output into a fixture adapter then calls `synthesize(fixture:options:)` — so Phase 1 and Phase 2 share a single synthesizer entry point.

## Key Code
- [`Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Synthesize/CosyVoice3Synthesizer.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Synthesize/CosyVoice3Synthesizer.swift) — `actor CosyVoice3Synthesizer`.
- [`Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Synthesize/CosyVoice3RasSampler.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Synthesize/CosyVoice3RasSampler.swift) — RAS sampler.
- [`Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Synthesize/CosyVoice3SpeechEmbeddings.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Synthesize/CosyVoice3SpeechEmbeddings.swift) — speech embedding mmap.
- [`Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Synthesize/CosyVoice3Types.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Synthesize/CosyVoice3Types.swift) — result/options types.

## Edge Cases & Failure Modes
- Flow CFM forced fp32 CPU/GPU — ANE+fp16 NaNs through fused `layer_norm` (CoreMLTools limitation per docstring).
- HiFT has ~12 sinegen/windowing ops that fall back to CPU.
- Stateless decode with external KV cache avoids `MLState` so the iOS 17 / macOS 14 floor still applies.
- `replayDecodedTokens` mode (`CosyVoice3ParityOptions`) is only valid in fixture mode; text-mode synthesis must pass `replayDecodedTokens: false`.
- [REVIEW: performance — RTFx < 1.0 typical. Investigation ongoing whether the residual cost is fundamental or recoverable via better conversion.]

## Test Coverage
- No dedicated synthesizer test in scope; covered by `CosyVoice3PromptMelTests`, `CosyVoice3TextChunkerTests`, fixture parity harnesses.

## Changelogs
