---
id: multilingual-g2p
name: Multilingual G2P (CharsiuG2P ByT5)
repo: FluidAudio
status: active
linked_features: [kokoro-tts-manager, kokoro-pipeline]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Multilingual G2P (CharsiuG2P ByT5)

## TL;DR
Thread-safe `actor` that wraps a CoreML CharsiuG2P ByT5 encoder/decoder model and converts a single word in one of nine supported languages to an array of IPA phoneme strings. Byte-level tokenization (no vocab file required), greedy autoregressive decode up to 128 steps.

## Motivation — why it exists

## Context

## What It Does
`MultilingualG2PModel` is a public actor exposing a shared singleton (`.shared`) and a single hot method: `phonemize(word:language:)`. The model is lazy-loaded — `loadIfNeeded()` resolves the encoder/decoder .mlmodelc files under `~/.cache/fluidaudio/Models/kokoro/` (the same cache as the main Kokoro bundle, sharing the repo). If loading fails and the `CI` environment variable is set, `phonemize` returns `nil` instead of throwing so test runs without model artifacts don't fail. Both models are loaded with `computeUnits = .cpuOnly`.

ByT5 tokenization is byte-level with a 3-token offset: input `"<\(language.charsiuCode)>: \(word)"` is UTF-8-encoded and each byte `b` becomes token `b + 3`. The pad token (0) is also the decoder start token, and EOS is token 1. The encoder runs once per word; the decoder runs in an autoregressive greedy loop, capped at `maxDecodeSteps = 128`. Each step argmax-picks the next token from the logits' last position. Output tokens are inverse-mapped (`tokenId - 3`) back to bytes, joined as UTF-8, then split into individual phoneme characters (Swift `.map { String($0) }`).

`MultilingualG2PLanguage` is a `String, CaseIterable` enum with nine cases: `americanEnglish ("eng-us")`, `britishEnglish ("eng-uk")`, `spanish ("spa")`, `french ("fra")`, `hindi ("hin")`, `italian ("ita")`, `japanese ("jpn")`, `brazilianPortuguese ("por-bz")`, `mandarinChinese ("cmn")`. The `prefix` property returns `"<{code}>: "` (used directly in the model input). `fromKokoroVoice(_:)` maps Kokoro's 2-character voice prefix (`af/am` → English, `bf/bm` → British, `ef/em` → Spanish, `ff/fm` → French, `hf/hm` → Hindi, `if/im` → Italian, `jf/jm` → Japanese, `pf/pm` → Portuguese, `zf/zm` → Mandarin) to a `MultilingualG2PLanguage`, returning `nil` for anything unrecognized.

`MultilingualG2PError` cases: `modelLoadFailed(String)`, `encoderPredictionFailed`, `decoderPredictionFailed` — each with a `LocalizedError.errorDescription` that names the failure.

## Key Code
- [`Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift:9`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift#L9) — `public actor MultilingualG2PModel` with shared singleton.
- [`Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift:37`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift#L37) — `phonemize(word:language:)` end-to-end pipeline.
- [`Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift:51`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift#L51) — input construction: `"<{prefix}>{word}"` → UTF-8 bytes → token IDs.
- [`Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift:83`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift#L83) — greedy autoregressive decode loop (max 128 steps).
- [`Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift:107`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift#L107) — argmax over last-position logits.
- [`Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift:126`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift#L126) — output bytes → UTF-8 string → per-character phoneme array.
- [`Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift:147`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/G2P/MultilingualG2PModel.swift#L147) — `loadIfNeeded()`: CPU-only model load from `~/.cache/fluidaudio/Models/kokoro/`.
- [`Sources/FluidAudio/TTS/G2P/MultilingualG2PLanguage.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/G2P/MultilingualG2PLanguage.swift#L5) — 9-case language enum mapped to CharsiuG2P codes.
- [`Sources/FluidAudio/TTS/G2P/MultilingualG2PLanguage.swift:27`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/G2P/MultilingualG2PLanguage.swift#L27) — `fromKokoroVoice(_:)` 2-char prefix mapping.
- [`Sources/FluidAudio/TTS/G2P/MultilingualG2PError.swift:4`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/G2P/MultilingualG2PError.swift#L4) — 3-case `LocalizedError` enum.

## Edge Cases & Failure Modes
- Returns `nil` if loading fails and `CI` env var is set — preserves test ergonomics in unattended runs.
- Returns `nil` if the decoded byte sequence is empty or fails UTF-8 decode.
- Returns `nil` if `loadIfNeeded` succeeds but encoder/decoder is somehow still `nil` (defensive `guard let`).
- Throws `MultilingualG2PError.modelLoadFailed` if either .mlmodelc is missing on disk.
- Throws `MultilingualG2PError.encoderPredictionFailed` / `.decoderPredictionFailed` when CoreML returns nil from `prediction(from:)`.
- 128-step decode cap silently truncates very long IPA outputs — no error surfaced.
- Output split is per Swift `Character` (grapheme cluster), so multi-byte IPA combining marks may be split incorrectly.
- [REVIEW: per-character split assumes IPA is single-character-per-phoneme — combining diacritics (e.g. tone marks, length) may need grapheme-aware grouping; check whether downstream `PhonemeMapper` handles this.]
- [REVIEW: greedy decode discards probability information — no temperature/topK exposed; not suitable for sampling-based pronunciation diversity.]

## Performance / Concurrency Notes
- Single shared actor — all calls serialize through the singleton. Hot loops doing many word lookups will be bottlenecked.
- `.cpuOnly` compute units chosen for stability; CoreML GPU dispatch on these encoder/decoder graphs is not exercised.
- Per-word decode rebuilds `decoder_input_ids = [padTokenId] + outputTokens` each step (no KV cache) — O(n²) in output length.
- Loaded once per process via `loadIfNeeded()` short-circuit.

## Test Coverage
- `Tests/FluidAudioTests/TTS/MultilingualG2PTests.swift`

## Changelogs
