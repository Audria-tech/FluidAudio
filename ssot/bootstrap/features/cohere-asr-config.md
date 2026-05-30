---
id: cohere-asr-config
name: Cohere ASR Config
repo: FluidAudio
status: active
linked_features:
  - cohere-asr-pipeline
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Cohere ASR Config

## TL;DR
`CohereAsrConfig` is the namespace-style enum that holds every architectural constant for the Cohere Transcribe model: sample rate, max audio length, vocabulary size, encoder/decoder dimensions, special-token IDs, language enum, and the per-language `promptSequence` that primes the decoder. Pure data — no logic — so the pipeline can be unit-tested without loading models.

## Context

## What It Does
The top-level enum exposes 35 s max audio (560 000 samples @ 16 kHz, matching the encoder mel input `[1, 128, 3500]`), 16 384-token vocab, 48-layer encoder, 8-layer decoder with 8 heads and 128-dim head, `maxSeqLen: 108` for the decoder KV cache. Nested `MelSpec` carries the FFT / hop / mel-bin parameters. `SpecialTokens` holds the IDs for unk, no-speech, pad, EOS, start-of-transcript, start-of-context, emotion-undefined, PnC, NoITN, no-timestamp, no-diarize, and the word boundary marker (13764). The `Language` enum has 14 cases and per-case `tokenId` + `englishName` properties. The killer feature is `promptSequence`: builds the 10-token prefix the decoder needs `(wordBoundary, startOfContext, startToken, emoUndefined, lang, lang, pnc, noitn, notimestamp, nodiarize)` — the doubled language token is part of the Cohere-trained prompting convention.

## Key Code
- [`CohereAsrConfig.swift:4`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CohereAsrConfig.swift#L4) — top-level enum + scalar constants.
- [`CohereAsrConfig.swift:45`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CohereAsrConfig.swift#L45) — `MelSpec` parameters.
- [`CohereAsrConfig.swift:56`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CohereAsrConfig.swift#L56) — `SpecialTokens` (special-token IDs).
- [`CohereAsrConfig.swift:84`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CohereAsrConfig.swift#L84) — `Language` enum (14 cases).
- [`CohereAsrConfig.swift:151`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CohereAsrConfig.swift#L151) — `promptSequence` (10-token decoder prompt with doubled language token).

## Edge Cases & Failure Modes
- All values are `public static` constants — changing them requires recompiling consumers.
- Special-token IDs are hard-coded; a re-trained model with different token IDs would silently produce garbage output.
- The doubled language token in `promptSequence` is non-obvious and must not be deduplicated by callers.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Cohere/CohereAsrConfigTests.swift`.

## Changelogs
