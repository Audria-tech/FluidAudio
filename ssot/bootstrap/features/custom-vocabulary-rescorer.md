---
id: custom-vocabulary-rescorer
name: Custom Vocabulary Rescorer
repo: FluidAudio
status: active
linked_features:
  - custom-vocabulary-word-spotting
  - custom-vocabulary-bk-tree
  - sliding-window-asr-manager
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Custom Vocabulary Rescorer

## TL;DR
`VocabularyRescorer` is the CTC-based shallow-fusion rescorer that integrates a `CustomVocabularyContext` into Parakeet TDT output. Instead of blindly swapping phonetically-similar words, it scores both the original TDT word and candidate vocabulary terms against actual CTC log-probabilities for the same audio segment, replacing only when the vocabulary term has significantly higher acoustic evidence. The implementation is split across four files for testability.

## Context

## What It Does
`VocabularyRescorer.swift` defines the public `Sendable` struct — `spotter`, `vocabulary`, `ctcTokenizer`, BK-tree, and `Config` (adaptive thresholds, reference token count). The `create(spotter:vocabulary:config:ctcModelDirectory:)` async factory loads the BK-tree (for large vocabularies) and the CTC tokenizer (used to translate vocabulary terms to CTC token IDs).

`VocabularyRescorer+TokenRescoring.swift` is the largest piece (~710 lines) and houses `ctcTokenRescore(transcript:tokenTimings:logProbs:frameDuration:cbw:marginSeconds:minSimilarity:)` — the public entry called by `SlidingWindowAsrManager`. It walks transcript words, applies a static `stopwords` filter, queries `findCandidateTermsForWord(...)` (BK-tree backed), and emits a `RescoreOutput` with per-word `replacements`. Each replacement records the original word, candidate, similarity, and `shouldReplace` flag.

`VocabularyRescorer+TokenEvaluation.swift` is the per-candidate evaluation core: `evaluateCTCMatch(candidate:logProbs:frameDuration:cbw:marginSeconds:)` scores a `CTCMatchCandidate` against the CTC log-probs by running DP within a temporal margin around the word's timing, then converts that score into a replacement decision using the configured threshold hierarchy from `ContextBiasingConstants`.

`VocabularyRescorer+Utilities.swift` houses string-similarity helpers (`stringSimilarity` = Levenshtein-based, `lengthPenalizedSimilarity` for compound matches) and other utility functions shared across the rescoring stages.

## Key Code
- [`VocabularyRescorer.swift:12`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/Rescorer/VocabularyRescorer.swift#L12) — struct declaration + `Config`.
- [`VocabularyRescorer+TokenRescoring.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/Rescorer/VocabularyRescorer%2BTokenRescoring.swift#L1) — `ctcTokenRescore(...)` main entry + `stopwords`.
- [`VocabularyRescorer+TokenEvaluation.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/Rescorer/VocabularyRescorer%2BTokenEvaluation.swift#L1) — `evaluateCTCMatch(...)`.
- [`VocabularyRescorer+Utilities.swift:8`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/Rescorer/VocabularyRescorer%2BUtilities.swift#L8) — `stringSimilarity` Levenshtein helper.
- [`VocabularyRescorer+Utilities.swift:21`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/Rescorer/VocabularyRescorer%2BUtilities.swift#L21) — `lengthPenalizedSimilarity` for compounds.

## Edge Cases & Failure Modes
- Stop-words filter is hard-coded and English-only; multilingual vocabulary boosting won't filter foreign-language stopwords.
- `minSimilarity` thresholds form a hierarchy documented in `ContextBiasingConstants` — values are derived from vocab size at the call site.
- When CTC tokenizer fails to load, BK-tree is the only fallback — accuracy drops.
- `RescoreOutput.wasModified` is the only reliable "did we change anything?" signal.
- Adaptive thresholds (when enabled) adjust min-similarity by token count of vocab term — short words get strict thresholds, longer phrases more lenient.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Parakeet/SlidingWindow/CustomVocabulary/CustomVocabularyTests.swift`, `VocabularyRescorerUtilsTests.swift`, `ContextBiasingConstantsTests.swift`, `ContextBiasingSendableTests.swift`.

## Changelogs
