---
id: magpie-pipeline
name: Magpie Pipeline (Preprocess + Synthesize)
repo: FluidAudio
status: active
linked_features: [magpie-tts-manager, magpie-local-transformer, magpie-types, magpie-constants]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Magpie Pipeline (Preprocess + Synthesize)

## TL;DR
Internal Magpie synthesis pipeline. `Preprocess/` handles per-language tokenization (char / Mandarin / IPA tokenizers), chunking, and the `|...|` IPA-override regex; `Synthesize/` runs the prefill → AR loop → NanoCodec decode, with `MagpieKvCache` holding decoder state across steps.

## Motivation — why it exists

## Context

## What It Does
**Preprocess** — `MagpieTokenizer` is the per-language dispatcher that lazy-loads `MagpieCharTokenizer`, `MagpieMandarinTokenizer`, or `MagpiePhonemeTokenizer` from `tokenizer_data/<lang>/`. Pads/truncates to `MagpieConstants.maxTextLength = 256` and produces a `MagpieTokenizedText` (paddedIds + mask + realLength). `MagpieIpaOverride` parses `|...|` regions for direct IPA token lookup, used when `options.allowIpaOverride` is true. `MagpieChunker` splits long input into chunker chunks; the streaming path uses a short first chunk (~50 codec frames) for low time-to-first-audio.

**Synthesize** — `MagpieSynthesizer` is the actor orchestrating one synthesis: tokenize → text_encoder → optional CFG (zero-context encoder pair) → 110-step prefill → AR loop up to 500 steps (embed current 8 codes → `decoder_step` → local-transformer sample → new codes) → NanoCodec decode → optional peak normalize to 0.9. `MagpiePrefill` holds the prefill helper logic; `MagpieKvCache` is the decoder-step KV cache (size capped at `maxCacheLength = 512`). `MagpieNanocodec` runs the final fp32 PCM 22 kHz decoder. `warmup()` runs a 16-step throwaway "." synthesis with EOS forbidden so every CoreML graph specializes before the user-visible call.

## Key Code
- [`Sources/FluidAudio/TTS/Magpie/Pipeline/Synthesize/MagpieSynthesizer.swift:16`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/Pipeline/Synthesize/MagpieSynthesizer.swift#L16) — `public actor MagpieSynthesizer`.
- [`Sources/FluidAudio/TTS/Magpie/Pipeline/Synthesize/MagpieSynthesizer.swift:36`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/Pipeline/Synthesize/MagpieSynthesizer.swift#L36) — `warmup()` with EOS forbidden.
- [`Sources/FluidAudio/TTS/Magpie/Pipeline/Preprocess/MagpieTokenizer.swift:30`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/Pipeline/Preprocess/MagpieTokenizer.swift#L30) — top-level dispatcher.
- [`Sources/FluidAudio/TTS/Magpie/Pipeline/Synthesize/MagpieKvCache.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/Pipeline/Synthesize/MagpieKvCache.swift) — decoder KV cache.
- [`Sources/FluidAudio/TTS/Magpie/Pipeline/Synthesize/MagpieNanocodec.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/Pipeline/Synthesize/MagpieNanocodec.swift) — fp32 PCM 22 kHz decoder.

## Edge Cases & Failure Modes
- Pad / truncate to 256 tokens silently discards overflow.
- `|...|` regions outside `allowIpaOverride` mode treat `|` as a literal character.
- AR loop minFrames > maxSteps disables EOS — used only by warmup.
- KV cache hard-capped at 512 positions — long outputs are truncated.
- [REVIEW: streaming chunker reserves first chunk ~50 codec frames; check interaction with very short inputs (< 50 frames total) — does the chunker still emit a single chunk?]

## Test Coverage
- `Tests/FluidAudioTests/TTS/Magpie/MagpieKvCacheTests.swift`, `MagpieIpaOverrideTests.swift`.

## Changelogs
