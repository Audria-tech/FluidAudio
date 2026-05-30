---
id: custom-vocabulary-bk-tree
name: Custom Vocabulary BK-Tree
repo: FluidAudio
status: active
linked_features:
  - custom-vocabulary-rescorer
  - custom-vocabulary-word-spotting
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Custom Vocabulary BK-Tree

## TL;DR
`BKTree` is the Burkhard-Keller tree used by `VocabularyRescorer` for fast approximate-string lookups of vocabulary terms. Immutable nodes make it cheap-to-share-across-actors (`Sendable` without `@unchecked`) and the search prunes to roughly `O(log V)` for small edit-distance thresholds — important when vocabularies hit the thousands.

## Context

## What It Does
`BKTree.swift` builds the tree: each node stores a `CustomVocabularyTerm`, its normalized text, and a `[Int: Node]` map from edit distance to child. Children are assembled bottom-up at init so the struct is immutable. `search(query:maxDistance:)` returns `[SearchResult]` with terms within the threshold.

`VocabularyRescorer+CandidateMatching.swift` is where the BK-tree is queried inside the rescoring pipeline. The extension's `findCandidateTermsForWord(normalizedWord:adjacentNormalized:minSimilarity:)` runs three queries — single-word, two-word compound, three-word compound — to detect both isolated terms and multi-word phrases. Each query's `maxDistance` is derived from `minSimilarity` and the candidate length, then capped by `bkTreeMaxDistance`. When BK-tree mode is disabled, the same routine falls back to linear scanning across all terms.

## Key Code
- [`BKTree.swift:19`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/BKTree/BKTree.swift#L19) — `BKTree` struct + immutable `Node`.
- [`BKTree.swift:23`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/BKTree/BKTree.swift#L23) — node layout with distance-keyed children.
- [`VocabularyRescorer+CandidateMatching.swift:26`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/CustomVocabulary/BKTree/VocabularyRescorer%2BCandidateMatching.swift#L26) — `findCandidateTermsForWord(...)` triple compound search.

## Edge Cases & Failure Modes
- Empty input word returns empty candidates fast.
- When `useBKTree` is false (config knob), candidate matching falls back to a linear scan — correct but `O(V)`.
- BK-tree max-distance is bounded by `bkTreeMaxDistance` (configured at rescorer build) — very lenient thresholds can degenerate toward full traversal.
- Compound matching only considers adjacent words; a 4+ word phrase will not be detected as a single candidate.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Parakeet/SlidingWindow/CustomVocabulary/BKTreeTests.swift`.

## Changelogs
