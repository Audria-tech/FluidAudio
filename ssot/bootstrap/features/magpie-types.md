---
id: magpie-types
name: Magpie Types (Language, Speaker, Options)
repo: FluidAudio
status: active
linked_features: [magpie-tts-manager, magpie-pipeline, magpie-constants]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Magpie Types (Language, Speaker, Options)

## TL;DR
Public `Sendable` value types backing the Magpie API: `MagpieLanguage` (8 cases, Japanese explicitly omitted), `MagpieSpeaker` (5 built-ins indexed 0-4), `MagpieSynthesisOptions` (temperature, topK, maxSteps, minFrames, cfgScale, seed, peakNormalize, allowIpaOverride), and the result/chunk types.

## Motivation — why it exists

## Context

## What It Does
`MagpieLanguage` is a `String, CaseIterable, Sendable, Hashable` enum: english, spanish, german, french, italian, vietnamese, mandarin, hindi. Japanese (`ja`) is intentionally excluded — the docstring notes OpenJTalk (static C++ lib) + MeCab dictionary (~102 MB) are deferred to a follow-up PR.

`MagpieSpeaker` is an `Int, Sendable, CaseIterable` enum with raw values 0-4 mapping to `john`, `sofia`, `aria`, `jason`, `leo`; carries a `displayName` getter. Speaker 0 has a known trailing-word fp16 sampler artifact (called out in `MagpieTtsManager`).

`MagpieSynthesisOptions` is a `Sendable` struct with knob defaults pulled from `MagpieConstants` (defaultTemperature, defaultTopK, maxSteps, minFrames, defaultCfgScale). `seed: UInt64?` selects reproducibility, `peakNormalize: Bool = true` controls post-process normalization to 0.9, `allowIpaOverride: Bool = true` enables `|...|` IPA pass-through tokenization. There's a `.default` static instance.

Result + chunk types (`MagpieSynthesisResult`, `MagpieAudioChunk`, `MagpiePhonemeTokens`) carry the synthesized output.

## Key Code
- [`Sources/FluidAudio/TTS/Magpie/MagpieTypes.swift:8`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieTypes.swift#L8) — `MagpieLanguage` (Japanese omitted).
- [`Sources/FluidAudio/TTS/Magpie/MagpieTypes.swift:20`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieTypes.swift#L20) — `MagpieSpeaker` 5 built-ins.
- [`Sources/FluidAudio/TTS/Magpie/MagpieTypes.swift:39`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieTypes.swift#L39) — `MagpieSynthesisOptions` knobs + `.default`.

## Edge Cases & Failure Modes
- Adding Japanese requires OpenJTalk + MeCab — explicitly out of scope.
- `MagpieSpeaker.john` (index 0) has a known fp16 trailing-word artifact.
- `peakNormalize` is force-disabled by `synthesizeStream` regardless of options input.
- [REVIEW: speakers limited to 5 — model card may support more; confirm whether additional embeddings can be wired in.]

## Test Coverage
- `Tests/FluidAudioTests/TTS/Magpie/MagpieConstantsTests.swift` (covers constants used by defaults).

## Changelogs
