---
id: ctc-decoder
name: CTC Decoder (Greedy + zh-CN Pipeline)
repo: FluidAudio
status: active
linked_features:
  - parakeet-language-models
  - custom-vocabulary-word-spotting
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# CTC Decoder (Greedy + zh-CN Pipeline)

## TL;DR
The CTC package implements greedy CTC decoding for Parakeet CTC models. `CtcDecoder.swift` provides the standard collapse-and-strip-blanks decoder over both `[[Float]]` log-prob matrices and `MLMultiArray` outputs. `CtcZhCnManager` is the actor-level pipeline for the 600 M Mandarin Chinese CTC model (preprocessor → encoder → CTC decoder → greedy decode). `ARPALanguageModel` loads ARPA-format unigram/bigram models for beam-search rescoring (natural-log scale).

## Context

## What It Does
`ctcGreedyDecode(logProbs:vocabulary:blankId:)` performs the canonical CTC collapse: argmax per frame, drop blanks and consecutive duplicates, and run `decodeCtcTokenIds(...)` to detokenize. Works on both nested `[[Float]]` log-prob matrices and `MLMultiArray` outputs shape `[1, T, V]`. The default `blankId: 1024` matches Parakeet CTC-110m vocabulary size.

`CtcZhCnManager` (actor) wraps a `CtcZhCnModels` instance (a typealias of `ParakeetLanguageModels<CtcZhCnConfig>`, blank ID `7000`, int8-encoder capable). `transcribe(audio:audioLength:)` runs the preprocessor, encoder, and CTC decoder MLModel calls in sequence, then greedy-decodes against the vocabulary. `CtcZhCnModels.swift` is the config + typealias.

`ARPALanguageModel` parses plain-text ARPA files into `[String: Entry]` unigrams and `[String: [String: Entry]]` bigrams, converting log10 probabilities to natural log (`log10ToNat = ln(10)`). OOV words fall back to `unkLogProb ≈ -23.026`. Trigrams and higher orders are skipped.

## Key Code
- [`CtcDecoder.swift:17`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CTC/CtcDecoder.swift#L17) — `ctcGreedyDecode(logProbs:[[Float]], vocabulary:blankId:)`.
- [`CtcDecoder.swift:46`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CTC/CtcDecoder.swift#L46) — `ctcGreedyDecode(logProbs:MLMultiArray, ...)`.
- [`CtcZhCnManager.swift:11`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CTC/CtcZhCnManager.swift#L11) — actor + `transcribe(audio:audioLength:)`.
- [`CtcZhCnModels.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CTC/CtcZhCnModels.swift#L5) — `CtcZhCnConfig` (blank `7000`, int8 capable) + `CtcZhCnModels` typealias.
- [`ARPALanguageModel.swift:17`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CTC/ARPALanguageModel.swift#L17) — `ARPALanguageModel` struct.
- [`ARPALanguageModel.swift:46`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CTC/ARPALanguageModel.swift#L46) — `load(from:)` plain-text parser.

## Edge Cases & Failure Modes
- Empty frames are silently skipped (no argmax).
- Greedy decode is broken for `ctc06b` per `CtcModels.swift` comment (CoreML conversion artefact) — recommended use is constrained CTC via the rescorer rather than greedy.
- `ARPALanguageModel.load(from:)` only supports plain-text ARPA — gzipped or binary KenLM throws `cannotOpen`.
- `CtcZhCnManager` is synchronous within the actor (`throws` not `async throws` on `transcribe`) — long audio blocks the actor; chunking is the caller's responsibility.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Parakeet/SlidingWindow/CTC/CtcDecoderTests.swift`, `CtcDecoderDemoTests.swift`, `CtcZhCnTests.swift`, `ARPALanguageModelTests.swift`.

## Changelogs
