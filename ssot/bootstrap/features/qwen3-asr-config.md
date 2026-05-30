---
id: qwen3-asr-config
name: Qwen3-ASR Config
repo: FluidAudio
status: active
linked_features:
  - qwen3-asr-manager
  - qwen3-asr-models
  - qwen3-rope
  - whisper-mel-spectrogram
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Qwen3-ASR Config

## TL;DR
`Qwen3AsrConfig` is the namespace enum that pins every Qwen3-ASR-0.6B architectural constant: audio (16 kHz, 128 mels, 100-frame encoder window with 8× downsample), encoder (18 layers, 1024-dim output), decoder (28 LLM layers, 1024 hidden, 151 936 vocab, 28 × 16 attention heads, RoPE θ=1 000 000), special tokens (audio start/end, audio placeholder, asr-text marker, EOS set), chat-template tokens (im_start, im_end, system, user, assistant), and the 30-language enum used for task-prompt routing.

## Context

## What It Does
Top-level constants are split into clearly-labelled blocks: audio params (sample rate, mel bins, `melWindowSize: 100`, `convDownsampleFactor: 8`, derived `outputFramesPerWindow: 13`), encoder dims, decoder dims (`hiddenSize: 1024`, `numDecoderLayers: 28`, `vocabSize: 151_936`, `ropeTheta: 1_000_000`, `mropeSection: [24, 20, 20]`), special-token IDs (`audioTokenId: 151_676`, `asrTextTokenId: 151_704`, `eosTokenIds: {151_645, 151_643}`), chat-template tokens (`imStartTokenId: 151_644`, `userTokenId: 872`, `assistantTokenId: 77_091`, `newlineTokenId: 198`), `maxCacheSeqLen: 512` (the stateful KV cache cap).

The nested `Language` enum has 30 cases with `englishName` and `init?(from:)` that accepts either ISO codes (`"en"`, `"zh"`) or English names (`"English"`, `"Chinese"`). The manager uses the resolved `Language` to look up pre-tokenized task tokens; `nil` triggers automatic language detection.

## Key Code
- [`Qwen3AsrConfig.swift:10`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrConfig.swift#L10) — enum start + audio params.
- [`Qwen3AsrConfig.swift:23`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrConfig.swift#L23) — `outputFramesPerWindow = (100 + 7) / 8`.
- [`Qwen3AsrConfig.swift:34`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrConfig.swift#L34) — decoder dims (`mropeSection`, 28 layers, 16 / 8 attention heads).
- [`Qwen3AsrConfig.swift:46`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrConfig.swift#L46) — special token IDs.
- [`Qwen3AsrConfig.swift:69`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrConfig.swift#L69) — `maxCacheSeqLen: 512`.
- [`Qwen3AsrConfig.swift:75`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrConfig.swift#L75) — `Language` enum with 30 cases.
- [`Qwen3AsrConfig.swift:144`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrConfig.swift#L144) — `Language.init?(from:)` accepts ISO + English name.

## Edge Cases & Failure Modes
- `maxCacheSeqLen: 512` is the hard upper bound on prompt + new tokens — long audio + long prompt rejects with `Qwen3AsrError.generationFailed`.
- `mropeSection [24, 20, 20]` is documented but unused for ASR (no spatial dims) — `Qwen3RoPE` collapses to standard RoPE on `headDim = 128`.
- Special-token IDs are baked from the trained checkpoint; a re-export would invalidate them.
- The docstring mentions "30 languages + 22 Chinese dialects" but the enum only enumerates 30 — dialects are presumably handled at the language-token granularity.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Qwen3/Qwen3AsrConfigTests.swift`.

## Changelogs
