---
id: custom-vocabulary-word-spotting
name: Custom Vocabulary Word Spotting (CTC Keyword Spotter)
repo: FluidAudio
status: active
linked_features:
  - custom-vocabulary-rescorer
  - custom-vocabulary-bk-tree
  - ctc-decoder
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Custom Vocabulary Word Spotting (CTC Keyword Spotter)

## TL;DR
The WordSpotting subpackage implements the CTC-based keyword spotter used by the rescorer to verify that vocabulary terms have acoustic support before they replace any TDT output. `CtcKeywordSpotter` drives the CoreML mel + encoder + CTC models, `CtcDPAlgorithm` runs the NeMo `ctc_word_spotter` dynamic programming match per keyword, `CtcModels` is the model container with `CtcModelVariant` (110m / 0.6b), and `BpeTokenizer` + `CtcTokenizer` translate keywords into CTC token IDs.

## Context

## What It Does
`CtcKeywordSpotter.swift` defines the public struct (`Sendable`) and constants — sample rate, max model samples, 32 000 sample overlap (2 s @ 16 kHz), debug flag, CTC `temperature`, and a blank-bias knob. `CtcKeywordSpotter+Inference.swift` provides `spotKeywordsWithLogProbs(audioSamples:customVocabulary:minScore:)` — runs the CoreML mel+encoder over chunks of `maxModelSamples` with 32 000-sample overlap, applies temperature + blank-bias to logits, returns `(logProbs: [[Float]], frameDuration: Double)`.

`CtcDPAlgorithm.swift` ports NeMo's keyword DP: `fillDPTable(logProbs:keywordTokens:)` builds `dp[t][n]` (best score to match first n keyword tokens by time t), `backtrack[t][n]` (start frame for best alignment), and `lastMatch[t][n]`. Wildcards (`wildcardTokenId = -1`) match any token at zero cost. The actual `spot...` calls live in `CtcKeywordSpotter+Inference.swift`.

`CtcModels.swift` holds the model container (`melSpectrogram`, `encoder`, `MLModelConfiguration`, `vocabulary`, `blankId`) and the `CtcModelVariant` enum (`.ctc110m`, `.ctc06b`). Both variants are explicitly broken for greedy decode (per the file-level comment) and exist only to feed the rescorer.

`BpeTokenizer.swift` + `CtcTokenizer.swift` load HuggingFace `tokenizer.json` for Parakeet, applying NFKC + lowercase normalization before BPE encoding so the rescorer can convert vocabulary terms into the CTC token IDs the DP algorithm expects.

## Key Code
- [`CtcKeywordSpotter.swift:12`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/WordSpotting/CtcKeywordSpotter.swift#L12) — struct + constants.
- [`CtcKeywordSpotter+Inference.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/WordSpotting/CtcKeywordSpotter%2BInference.swift#L1) — CoreML inference + log-prob assembly.
- [`CtcDPAlgorithm.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/WordSpotting/CtcDPAlgorithm.swift#L1) — pure DP algorithms.
- [`CtcModels.swift:14`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/WordSpotting/CtcModels.swift#L14) — `CtcModelVariant` (`ctc110m` / `ctc06b`).
- [`BpeTokenizer.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/WordSpotting/BpeTokenizer.swift#L1) — BPE encoder with NFKC normalization.
- [`CtcTokenizer.swift:7`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/WordSpotting/CtcTokenizer.swift#L7) — async loader around `BpeTokenizer`.

## Edge Cases & Failure Modes
- Greedy CTC decode against these models produces poor WER per the file-header comments — the spotter is *only* used in the rescoring path.
- Long audio is chunked at `maxModelSamples` with 32 000-sample overlap; boundary keywords can be detected at most twice (once per overlapping chunk).
- Wildcards (`-1`) are only respected by the DP, not by the spotter's inference path.
- Temperature applied to logits before softmax can flatten or sharpen the distribution — tuned via `ContextBiasingConstants.ctcTemperature`.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Parakeet/SlidingWindow/CustomVocabulary/CtcDPAlgorithmTests.swift` (DP), `CustomVocabularyTests.swift` (end-to-end).

## Changelogs
