---
id: string-utils
name: String Utils
repo: FluidAudio
status: active
linked_features: []
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# String Utils

## TL;DR
Two overloads of `levenshteinDistance` — one generic over `[T: Equatable]` (used for word-level WER and token-list diffing), one specialised for `String` (character-level). Standard `O(m × n)` dynamic-programming table.

## Motivation — why it exists

## Context

## What It Does
Classic Wagner-Fischer DP: builds an `(m+1) × (n+1)` Int matrix, seeds row 0 / column 0 with index values (insertion / deletion costs), then fills cell `(i, j)` as the min of deletion + 1, insertion + 1, and substitution + (mismatch ? 1 : 0). Returns `dp[m][n]`. The `String` overload converts to `[Character]` and delegates.

## Key Code
- [`Sources/FluidAudio/Shared/StringUtils.swift:12`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/StringUtils.swift#L12) — generic `levenshteinDistance<T: Equatable>([T], [T]) -> Int`
- [`Sources/FluidAudio/Shared/StringUtils.swift:39`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/StringUtils.swift#L39) — `String` convenience overload (character-level)

## Edge Cases & Failure Modes
- Empty inputs short-circuit to `n` / `m` — no DP allocation.
- Allocates a full `(m+1) × (n+1)` matrix — `O(m × n)` memory; large inputs may be expensive. No row-pair-only optimisation.
- The DP increments by `1` per edit — unweighted cost only; no support for substitution-cost ≠ insertion/deletion-cost (which would matter for some WER variants).

## Test Coverage
`Tests/FluidAudioTests/Shared/StringUtilsTests.swift` covers the standard distance behaviours and empty / equal / disjoint edge cases.

## Changelogs
