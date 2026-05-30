---
id: token-language-filter
name: Token Language Filter
repo: FluidAudio
status: active
linked_features: [model-names]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Token Language Filter

## TL;DR
Script-based filter that restricts ASR decoder candidates to the target language's writing system (Latin vs Cyrillic). Used by the V3 TDT decoder (`JointDecisionv3.mlmodelc`) to stop the joint network from emitting cross-script tokens — e.g. Cyrillic letters while transcribing Polish (issue #512).

## Motivation — why it exists

## Context

## What It Does
`Language` enumerates 18 supported language codes and maps each to a `Script` (`.latin` or `.cyrillic`). `matches(_:script:)` checks whether every Unicode scalar of a token text falls inside the script's allowed code-point ranges. Latin ranges: ASCII, Latin-1, Latin Extended-A/B, Combining Diacritical Marks (for NFD-decomposed accents), Latin Extended Additional. Cyrillic range: U+0400–U+04FF, plus an explicit reject for ASCII A–Z/a–z (script-neutral but covered by Latin). The SentencePiece word-boundary marker (▁, U+2581) is stripped before checking, and pure-boundary tokens return `true` so the caller's argmax decides on logit alone.

`filterTopK(topKIds:topKLogits:vocabulary:preferredScript:)` walks top-K outputs from the V3 joint, takes the argmax over right-script candidates (no assumption that top-K is logit-sorted — CoreML does not guarantee it), and computes a softmax probability **over the top-K set only** with max-logit stabilisation. The returned probability is consistently larger than a full-vocab softmax — the comment makes this explicit and tells callers to use it for ranking only.

## Key Code
- [`Sources/FluidAudio/Shared/TokenLanguageFilter.swift:4`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/TokenLanguageFilter.swift#L4) — `Language` enum (18 codes, including Latin-script Slavic like Polish/Czech vulnerable to Cyrillic confusion)
- [`Sources/FluidAudio/Shared/TokenLanguageFilter.swift:40`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/TokenLanguageFilter.swift#L40) — `Script` enum (`.latin` / `.cyrillic`)
- [`Sources/FluidAudio/Shared/TokenLanguageFilter.swift:56`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/TokenLanguageFilter.swift#L56) — sentencepiece boundary marker constant (U+2581)
- [`Sources/FluidAudio/Shared/TokenLanguageFilter.swift:64`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/TokenLanguageFilter.swift#L64) — `matches(_:script:)` — Unicode range check; doc explains why Cyrillic needs an explicit ASCII-letter reject
- [`Sources/FluidAudio/Shared/TokenLanguageFilter.swift:106`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/TokenLanguageFilter.swift#L106) — `filterTopK` — argmax over right-script + top-K softmax with stability

## Edge Cases & Failure Modes
- Pure word-boundary tokens (▁ only) pass `matches` for any script — the caller is responsible for sensible argmax behaviour on them.
- Tokens whose IDs are missing from the `vocabulary` dict are silently skipped — out-of-vocab top-K entries are treated as wrong-script.
- Returns `nil` when no right-script candidate exists; callers must fall back to a different policy (typically unfiltered argmax).
- The "Polish vs Czech" distinction is **not** enforced — both fall under `.latin` and a Czech `ř` would pass for Polish. The doc comment notes a per-language allowlist could plug in later. [REVIEW: script-level partitioning treats every Latin-script Slavic language identically — known limitation per docstring]
- Logit `-Infinity` is handled via the `bestIdx < 0` clause that forces the first match to win even when its logit is `-inf` — without this the sentinel would never be beaten.
- Top-K softmax probability is over top-K only; the doc warns about misuse but nothing in the API prevents callers from comparing this with a full-vocab softmax.
- `matches` allocates a new String for the cleaned text on every call (`replacingOccurrences`) — non-zero cost on hot decoder loops with many top-K candidates.

## Test Coverage
`Tests/FluidAudioTests/Shared/TokenLanguageFilterTests.swift` covers script classification, boundary handling, and the top-K argmax behaviour.

## Changelogs
