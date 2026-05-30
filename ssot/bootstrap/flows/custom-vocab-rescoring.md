---
id: custom-vocab-rescoring
name: Custom Vocabulary Rescoring
trigger_type: event
trigger: A sliding-window TDT/CTC chunk is being promoted from volatile to confirmed and `vocabBoostingEnabled == true`.
end_state: The chunk's `ASRResult` may have one or more words replaced by vocabulary terms when CTC acoustic evidence supports the swap; `detected` and `applied` term lists are surfaced in the `SlidingWindowTranscriptionUpdate`.
involves_features: [custom-vocabulary-bk-tree, custom-vocabulary-word-spotting, custom-vocabulary-rescorer, tdt-decoder, ctc-decoder]
linked_flows: [batch-asr-parakeet-tdt, batch-asr-parakeet-ctc]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Custom Vocabulary Rescoring

## TL;DR
CTC-based shallow-fusion rescorer that integrates a `CustomVocabularyContext` into Parakeet TDT output. Per confirmed chunk the manager re-runs CTC inference on the same audio, queries a BK-tree for candidate vocabulary terms near each transcript word, scores each candidate against the CTC log-probs around that word's timing, and replaces only when the vocabulary term has significantly higher acoustic evidence than the TDT word.

## Trigger & Preconditions
`SlidingWindowAsrManager.configureVocabularyBoosting(vocabulary:ctcModels:config:)` must have been called to attach a `CtcKeywordSpotter` and `VocabularyRescorer`. Then a window's promotion from volatile to confirmed (high confidence + sufficient context) gates the rescoring call.

## Stages
1. **TDT output ready** — chunk's `ASRResult` carries text, token IDs, and chunk-local `tokenTimings`; promotion to confirmed gates this flow.
2. **Run CTC over the window** — `CtcKeywordSpotter.spotKeywordsWithLogProbs(audioSamples:customVocabulary:minScore:)` runs the CTC mel + encoder CoreML models with `temperature` + blank-bias adjustments on logits, returning `(logProbs: [[Float]], frameDuration: Double)`: [`Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/WordSpotting/CtcKeywordSpotter+Inference.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/WordSpotting/CtcKeywordSpotter%2BInference.swift#L1).
3. **Enter rescorer** — `VocabularyRescorer.ctcTokenRescore(transcript:tokenTimings:logProbs:frameDuration:cbw:marginSeconds:minSimilarity:)` walks transcript words, applies a hard-coded English `stopwords` filter: [`Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/Rescorer/VocabularyRescorer+TokenRescoring.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/Rescorer/VocabularyRescorer%2BTokenRescoring.swift#L1).
4. **BK-tree candidate lookup** — `findCandidateTermsForWord(normalizedWord:adjacentNormalized:minSimilarity:)` runs three queries (single-word, two-word compound, three-word compound) against the BK-tree, with per-query `maxDistance` derived from `minSimilarity` and capped by `bkTreeMaxDistance`; falls back to linear scan when `useBKTree == false`: [`Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/BKTree/VocabularyRescorer+CandidateMatching.swift:26`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/BKTree/VocabularyRescorer%2BCandidateMatching.swift#L26).
5. **Tokenize candidates to CTC IDs** — `ctcTokenizer` (BPE with NFKC + lowercase normalization) converts each vocabulary candidate into CTC token IDs expected by the DP scorer.
6. **Score against CTC** — `evaluateCTCMatch(candidate:logProbs:frameDuration:cbw:marginSeconds:)` runs the NeMo `ctc_word_spotter` DP (`fillDPTable` in `CtcDPAlgorithm`) within a temporal margin around the word's chunk-local timing: [`Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/Rescorer/VocabularyRescorer+TokenEvaluation.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/Rescorer/VocabularyRescorer%2BTokenEvaluation.swift#L1).
7. **Apply threshold hierarchy** — adaptive thresholds derived from vocab size and candidate token count decide `shouldReplace`. Short words → stricter; longer phrases → more lenient.
8. **Apply replacements** — `RescoreOutput.replacements` (per-word `(original, candidate, similarity, shouldReplace)`) feeds `withRescoring(...)` to mutate the displayed `ASRResult` in place when `wasModified` is true: [`Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift:86`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift#L86).
9. **Surface in update** — `SlidingWindowTranscriptionUpdate` carries `detected` (BK-tree matches) and `applied` (passed acoustic gate) term lists.

## Outputs
- Possibly-mutated `ASRResult.text`; the same tokens but with replaced word strings.
- `detected: [String]` and `applied: [String]` term lists surfaced in the update.

## Error Modes
- Rescoring silently skipped when CTC spotter produces empty log-probs, token timings are empty, or rescorer throws (warning logged).
- Greedy CTC against `ctc06b` produces poor WER per file-level comments — model is only correctly used in this rescoring path.
- BK-tree disabled (`useBKTree=false`) degrades to `O(V)` linear scan.
- Hard-coded English stopwords → multilingual vocab won't filter foreign-language stopwords.
- 4+ word phrases not detected as a single candidate (only 1/2/3-word compounds queried).
- Wildcards (-1) honored by DP but not by the spotter's inference path.

## Prompts and Models Used

## Usage Metrics

## Changelogs
