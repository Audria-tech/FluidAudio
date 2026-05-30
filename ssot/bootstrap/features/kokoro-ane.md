---
id: kokoro-ane
name: Kokoro ANE 7-Stage Pipeline
repo: FluidAudio
status: active
linked_features: [kokoro-tts-manager, tts-backend-protocol, tts-compute-unit-preset]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Kokoro ANE 7-Stage Pipeline

## TL;DR
ANE-resident Kokoro 82M variant derived from laishere/kokoro-coreml. Splits the model into 7 stages (ALBERT → PostAlbert → Alignment → Prosody → Noise → Vocoder → Tail) with per-stage compute-unit assignment so ANE-friendly layers stay on the Neural Engine. Yields ~3-11× RTFx vs. the single-graph `KokoroTtsManager` at the cost of single-voice (`af_heart`), ≤ 512 IPA tokens, no custom lexicon.

## Motivation — why it exists

## Context

## What It Does
`KokoroAneManager` is a `public actor` facade that mirrors the surface of `KokoroTtsManager` so callers can swap backends with minimal churn. Internally it owns a `KokoroAneModelStore` (downloads 7 `.mlmodelc` files from `FluidInference/kokoro-82m-coreml/ANE/`), a `KokoroAneVocab` (IPA → input ids), a `KokoroAneVoicePack` (loads `af_heart.bin` as a flat `[510, 256]` fp32 row table — row chosen by utterance length, cols 0..128 are `style_timbre`, cols 128..256 are `style_s`), and a `KokoroAneSynthesizer` that wires the 7 stages with explicit ANE/GPU placement. G2P comes from the same `G2PModel` used by the single-graph path; the assets are read from the kokoro repo cache so first-time KokoroAne users who never ran the regular backend still pick them up. Constants in `KokoroAneConstants` define BOS/EOS = 0, max 512 input tokens / 510 phonemes (excluding BOS/EOS), 24 kHz output, and the voice-pack row schema.

## Key Code
- [`Sources/FluidAudio/TTS/KokoroAne/KokoroAneManager.swift:32`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/KokoroAne/KokoroAneManager.swift#L32) — `public actor KokoroAneManager` declaration + per-stage compute-units init.
- [`Sources/FluidAudio/TTS/KokoroAne/KokoroAneConstants.swift:14`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/KokoroAne/KokoroAneConstants.swift#L14) — sample rate, BOS/EOS, max tokens, voice-pack schema.
- [`Sources/FluidAudio/TTS/KokoroAne/Pipeline/KokoroAneSynthesizer.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/KokoroAne/Pipeline/KokoroAneSynthesizer.swift) — 7-stage orchestration.
- [`Sources/FluidAudio/TTS/KokoroAne/Pipeline/KokoroAneVoicePack.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/KokoroAne/Pipeline/KokoroAneVoicePack.swift) — flat `[510, 256]` fp32 row table.
- [`Sources/FluidAudio/TTS/KokoroAne/Assets/KokoroAneVocab.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/KokoroAne/Assets/KokoroAneVocab.swift) — IPA → input id vocabulary.

## Edge Cases & Failure Modes
- IPA sequences > 510 phonemes are rejected — no chunker.
- Only single voice `af_heart` ships in the ANE bundle.
- G2P CoreML assets must be present in the kokoro repo cache; `G2PModel.loadIfNeeded` does not download them.
- `KokoroAneError` enumerates per-stage / model-not-found / unsupported-input failures.
- [REVIEW: per-stage compute-unit assignment is documented as "4 ANE / 3 GPU" in `KokoroTtsManager`'s comparison table but the actual mapping lives in `KokoroAneComputeUnits`; confirm.]

## Test Coverage
- `Tests/FluidAudioTests/TTS/KokoroAne/KokoroAneAsrRoundtripTests.swift`, `KokoroAneSynthesizerTests.swift`, `KokoroAneVocabTests.swift`, `KokoroAneVoicePackTests.swift`.

## Changelogs
