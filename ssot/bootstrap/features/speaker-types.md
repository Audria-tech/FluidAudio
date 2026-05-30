---
id: speaker-types
name: Speaker Types
repo: FluidAudio
status: active
linked_features: [speaker-manager, speaker-operations]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Speaker Types

## TL;DR
Value types for speaker identity and embedding history: `Speaker` (the main mutable profile, Codable + Hashable + Equatable), `RawEmbedding` (per-segment timestamped embedding), `SendableSpeaker` (cross-boundary projection with Int id), and `SpeakerInitializationMode` (`.reset`/`.merge`/`.overwrite`/`.skip`).

## Motivation — why it exists

## Context

## What It Does
`Speaker` carries `id: String`, `name`, `currentEmbedding: [Float]` (L2-normalized on init), `duration`, `createdAt`, `updatedAt`, `updateCount`, `rawEmbeddings: [RawEmbedding]` (capped at 50, FIFO), and `isPermanent`. `Equatable`/`Hashable` are defined purely on `id` (two speakers with the same id are equal regardless of state). The init normalizes `currentEmbedding` via `VDSPOperations.l2Normalize`. Mutating methods: `updateMainEmbedding(duration:embedding:segmentId:alpha:)` validates with `vDSP_svesq` > 0.01, appends a raw embedding, and applies an EMA blend (default `alpha=0.9`). `addRawEmbedding(_:)` runs the same magnitude check, maintains the 50-cap FIFO, and calls `recalculateMainEmbedding` (which averages all raw embeddings and re-normalizes). `removeRawEmbedding(segmentId:)` removes by id and recalculates. `mergeWith(_:keepName:)` concatenates raw embeddings (keeping the 50 most-recent by `timestamp`), sums durations, applies optional rename, and recalculates the main embedding.

`RawEmbedding` is `Codable & Sendable` with `segmentId: UUID`, `embedding: [Float]` (also L2-normalized on init), and `timestamp: Date`.

`SendableSpeaker` is a flat `Sendable & Identifiable & Hashable` projection of `Speaker` for use across async boundaries. It uses an `Int` id: if `speaker.id` parses as Int it's used directly; otherwise `abs(speaker.id.hashValue)`. The `label` computed property returns `"Speaker #<id>"` for empty names.

`SpeakerInitializationMode` is a four-case `Sendable` enum used by `SpeakerManager.initializeKnownSpeakers`.

## Key Code
- [`SpeakerTypes.swift:6`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerTypes.swift#L6) — `Speaker` declaration; note `Equatable` based purely on `id`.
- [`SpeakerTypes.swift:68`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerTypes.swift#L68) — `updateMainEmbedding` magnitude gate (`sumSquares > 0.01`) and EMA loop.
- [`SpeakerTypes.swift:132`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerTypes.swift#L132) — `recalculateMainEmbedding` averages all raw embeddings and L2-normalizes.
- [`SpeakerTypes.swift:248`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerTypes.swift#L248) — `SendableSpeaker.init(from:)` Int-id fallback via `abs(hashValue)`.

## Edge Cases & Failure Modes
- `addRawEmbedding` silently drops low-magnitude embeddings (sum of squares ≤ 0.01).
- `recalculateMainEmbedding` is a no-op when `rawEmbeddings` is empty or the first embedding is empty — `currentEmbedding` stays unchanged.
- The 50-cap FIFO discards the oldest by *insertion order*, not timestamp; only `mergeWith` sorts by timestamp before pruning.
- `SendableSpeaker.hashValue`-fallback for non-numeric ids can collide between speakers with different IDs but the same Swift hash → `==` still works (compares id+name) but `Hashable` collisions reduce dictionary performance.
- `Speaker.id` is a `let`, so once set it cannot change; `SpeakerManager.upsertSpeaker` updates *other* fields but never rewrites the id.

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/SpeakerTests.swift` — `Speaker` mutation, equality, raw-embedding management.

## Changelogs
