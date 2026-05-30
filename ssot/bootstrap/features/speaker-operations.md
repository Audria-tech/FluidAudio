---
id: speaker-operations
name: Speaker Operations
repo: FluidAudio
status: active
linked_features: [speaker-manager, speaker-types]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Speaker Operations

## TL;DR
`SpeakerUtilities` is a stateless enum namespace of speaker-domain helpers (cosine distance, embedding validation, EMA updates, raw-embedding FIFO, merge math, platform-specific assignment thresholds). Two `SpeakerManager` extensions in the same file add segment reassignment and global-stats helpers.

## Motivation — why it exists

## Context

## What It Does
`SpeakerUtilities.AssignmentConfig` carries `.macOS` (0.65/0.45/4.0/1.0) and `.iOS` (0.55/0.45/4.0/1.0) preset thresholds; `.current` picks via `#if os(macOS)`. `cosineDistance` uses `vDSP_dotpr` + `vDSP_svesq`, takes a fast-path when both vectors are unit-norm (tolerance `1e-3`), and clamps similarity to `[-1, 1]` before returning `1 - similarity`. `validateEmbedding` requires non-empty, all-finite, magnitude > 0.1. `shouldAssignSpeaker` is a pure decision function producing `AssignmentDecision { shouldAssign, shouldUpdate, confidence, reason }`; it has a "very high confidence" fast path at `distance < 0.2`. `updateEmbedding` performs EMA with `alpha=0.9` after L2-normalizing both sides via `VDSPOperations.l2Normalize`. `addRawEmbedding` validates and maintains a 50-entry FIFO; `removeRawEmbedding` removes by `segmentId`. `updateSpeakerWithSegment` combines duration gating + raw-embedding addition + main-embedding EMA update. `mergeSpeakers` concatenates raw embeddings (keeping the 50 most-recent by timestamp) and computes a fresh average via `averageEmbeddings`. The `SpeakerManager` extension adds `reassignSegment(segmentId:from:to:)` and read-only `getCurrentSpeakerNames()` / `getGlobalSpeakerStats()`.

## Key Code
- [`SpeakerOperations.swift:62`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerOperations.swift#L62) — `cosineDistance` with unit-norm fast path; the `normalizationTolerance` constant is `1e-3`.
- [`SpeakerOperations.swift:106`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerOperations.swift#L106) — `validateEmbedding(_:minMagnitude:)` — minimum magnitude defaults to 0.1.
- [`SpeakerOperations.swift:139`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerOperations.swift#L139) — `shouldAssignSpeaker` decision tree; pure function, no state.
- [`SpeakerOperations.swift:299`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerOperations.swift#L299) — `updateEmbedding` EMA with default `alpha=0.9`.
- [`SpeakerOperations.swift:516`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerOperations.swift#L516) — `SpeakerManager.reassignSegment` extension method — moves a raw embedding between speakers but does NOT recalculate the main embeddings (logs "successful reassignment" without doing the math).

## Edge Cases & Failure Modes
- `cosineDistance` returns `Float.infinity` for length mismatches, empty inputs, or zero-magnitude vectors.
- `validateEmbedding` is strict — any NaN/Inf rejects the entire vector.
- `reassignSegment` is documented as TODO: it moves the embedding between speakers' `rawEmbeddings` arrays but does not call `recalculateMainEmbedding` on either side. [REVIEW: docstring promises "updating both speakers' main embeddings" but the code only logs.]
- `mergeSpeakers` averages embeddings without L2-renormalizing the per-vector inputs (only the final average is L2-normalized by `averageEmbeddings`).
- `getGlobalSpeakerStats` returns `(0, 0, 0, 0)` for an empty database.

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/SpeakerOperationsTests.swift` — exhaustive coverage of the static utility functions.

## Changelogs
