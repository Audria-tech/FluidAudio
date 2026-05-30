---
id: token-dedup-cross-chunk
name: Token Deduplication (Cross-Chunk Boundary Merging)
trigger_type: event
trigger: A new transcription chunk is appended to accumulated tokens during streaming or sliding-window long-form decoding.
end_state: The accumulated token list contains the new chunk with any boundary-overlap suffix/prefix removed; no duplicate spans across chunk seams.
involves_features: [token-deduplication, sliding-window-asr-session]
linked_flows: [batch-asr-parakeet-tdt, streaming-asr-parakeet]
sync_async: sync
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Token Deduplication (Cross-Chunk Boundary Merging)

## TL;DR
`SequenceMatcher<Element>` finds the longest suffix-of-previous == prefix-of-current span and returns a `SequenceMatch(leftStartIndex, rightStartIndex, length)`. Used to splice chunk outputs together without re-emitting tokens that were already in the previous chunk's tail.

## Trigger & Preconditions
Invoked by `AsrManager+TokenProcessing` after every per-chunk TDT decode, and by `ChunkProcessor` when consecutive long-form windows overlap by 2 s. Generic over `Element` so the same algorithm matches raw token IDs or `(token, timestamp)` tuples — the caller supplies a comparison closure.

## Stages
1. **Receive chunk tokens** — caller has `previous: [Element]` (already-accumulated) and `current: [Element]` (just-decoded) plus a `maxOverlap` ceiling.
2. **Find longest suffix-prefix match** — `SequenceMatcher.findSuffixPrefixMatch(previous:current:maxOverlap:matcher:)` walks candidate overlap lengths greedy longest → shortest; first hit wins: [`Sources/FluidAudio/ASR/Parakeet/TokenDeduplication/SequenceMatcher.swift:24`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/TokenDeduplication/SequenceMatcher.swift#L24).
3. **Reject too-short** — returns `nil` when either side has length < 2 (avoids spurious single-token matches): [`Sources/FluidAudio/ASR/Parakeet/TokenDeduplication/SequenceMatcher.swift:9`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/TokenDeduplication/SequenceMatcher.swift#L9).
4. **Splice** — caller uses the returned `SequenceMatch` to drop the overlapping prefix of `current` (via `rightRange`) and append the remainder to the accumulator.
5. **Bounded-substring path** — `ChunkProcessor` uses a second helper for windowed substring search where overlap can be anywhere inside a configurable window, not just at the boundary.

## Outputs
- Optional `SequenceMatch { leftStartIndex, rightStartIndex, length, leftRange, rightRange }`.
- `nil` when no qualifying overlap (single-token or shorter) is found — caller appends `current` whole.

## Error Modes
- Wrong comparison closure produces silently-incorrect dedup (the algorithm is generic; the caller owns semantics).
- Longest-first greedy bias: shorter overlaps that would actually be more correct are skipped if any longer pseudo-match succeeds first.
- Single-token overlaps are intentionally not detected — improves precision but can leave one-word duplicates at chunk seams.

## Prompts and Models Used

## Usage Metrics

## Changelogs
