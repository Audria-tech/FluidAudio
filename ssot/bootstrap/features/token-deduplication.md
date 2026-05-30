---
id: token-deduplication
name: Token Deduplication (Sequence Matcher)
repo: FluidAudio
status: active
linked_features:
  - tdt-decoder
  - sliding-window-asr-manager
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Token Deduplication (Sequence Matcher)

## TL;DR
`SequenceMatcher<Element>` is the generic suffix-prefix / bounded-substring matcher used to deduplicate tokens at chunk boundaries during streaming TDT and during long-form `ChunkProcessor` merges. Returns `SequenceMatch` (left start, right start, length) so callers can splice without copying.

## Context

## What It Does
`SequenceMatch.swift` defines the lightweight value type with `leftStartIndex`, `rightStartIndex`, `length`, and computed `leftRange` / `rightRange`. `SequenceMatcher.swift` is the generic algorithm host: `findSuffixPrefixMatch(previous:current:maxOverlap:matcher:)` finds the longest sequence where the *end* of `previous` matches the *beginning* of `current` (the streaming case), and the bounded-substring helpers look for overlapping subsequences within configurable windows (used by `ChunkProcessor` when consecutive long-form windows overlap by 2 s).

The matcher is generic over `Element` with a caller-supplied comparison closure so it can be applied to raw token IDs, `(token, timestamp)` pairs, or any other element type — `AsrManager+TokenProcessing.swift` uses it for token-based streaming dedup and `ChunkProcessor` uses it for time-based windowed merges.

## Key Code
- [`SequenceMatch.swift:4`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/TokenDeduplication/SequenceMatch.swift#L4) — value type.
- [`SequenceMatcher.swift:9`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/TokenDeduplication/SequenceMatcher.swift#L9) — generic matcher.
- [`SequenceMatcher.swift:24`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/TokenDeduplication/SequenceMatcher.swift#L24) — `findSuffixPrefixMatch(previous:current:maxOverlap:matcher:)`.

## Edge Cases & Failure Modes
- Returns `nil` when either side has length < 2 — single-token overlaps are not considered (avoids spurious matches).
- Searches longest → shortest (greedy); the first match wins.
- Generic over `Element` so caller controls match semantics — wrong closure produces silently incorrect dedup.

## Test Coverage
- `Tests/FluidAudioTests/ASR/TokenDeduplicationRegressionTests.swift`.

## Changelogs
