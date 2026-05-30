---
id: speaker-manager
name: Speaker Manager
repo: FluidAudio
status: active
linked_features: [diarizer-manager, speaker-operations, speaker-types]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Speaker Manager

## TL;DR
`SpeakerManager` is a Swift `actor` that owns the in-memory speaker database used by both the online diarizer and the streaming Sortformer/LS-EEND timeline plumbing. It performs nearest-speaker assignment under cosine distance, manages exponential-moving-average updates of speaker embeddings, and exposes merge/remove/upsert/reset operations with permanent-speaker protections.

## Motivation — why it exists

## Context

## What It Does
The actor stores `speakerDatabase: [String: Speaker]` plus monotonic ID counters (`nextSpeakerId`, `highestSpeakerId`). Four mutable thresholds drive its decisions: `speakerThreshold` (assignment cutoff; default 0.65 in init but in practice set to `clusteringThreshold * 1.2` by `DiarizerManager`), `embeddingThreshold` (EMA update gate; default 0.45 / `clusteringThreshold * 0.8`), `minSpeechDuration` (gate to create a new speaker), and `minEmbeddingUpdateDuration` (carried but currently only used implicitly by the duration-aware update logic). All four have actor-isolated setters.

`assignSpeaker(_:speechDuration:confidence:speakerThreshold:newName:)` is the workhorse: it validates the 256-dim embedding, L2-normalizes via `VDSPOperations.l2Normalize`, finds the nearest existing speaker by cosine distance, and either (1) updates that speaker if `distance < speakerThreshold` (with a finer EMA update only when `distance < embeddingThreshold` and `vDSP_svesq` magnitude > 0.01), or (2) creates a new speaker when `speechDuration >= minSpeechDuration`, or (3) returns nil. New speakers are minted with monotonically incrementing string IDs (`"1"`, `"2"`, ...) and a single initial `RawEmbedding`. `updateMainEmbedding` is delegated to the `Speaker` value type with `alpha=0.9` (current EMA weight 0.9, new sample weight 0.1).

`initializeKnownSpeakers` accepts arrays of pre-built `Speaker` values plus a `SpeakerInitializationMode` (`.reset`, `.merge`, `.overwrite`, `.skip`) and a `preserveIfPermanent` flag. It validates embedding size, handles ID collisions per mode, and recomputes `nextSpeakerId` from the maximum integer-parsable ID seen.

Beyond assignment, the actor exposes query helpers (`findSpeaker`, `findMatchingSpeakers`, `findSpeakers(where:)`, `hasSpeaker`, `speakerIds`, `permanentSpeakerIds`, `speakerCount`, `getAllSpeakers`, `getSpeakerList`, `getSpeaker(for:)`), mutation operations (`makeSpeakerPermanent`, `revokePermanence`, `mergeSpeaker`, `findMergeablePairs`, `removeSpeaker`, `removeSpeakersInactive(since:)` / `for:`, `removeSpeakers(where:)`, `reset(keepIfPermanent:)`, `resetPermanentFlags`, `upsertSpeaker`), and a `nonisolated` `cosineDistance` shim that delegates to `SpeakerUtilities.cosineDistance` for backward-compatible test access.

## Key Code
- [`SpeakerManager.swift:13`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerManager.swift#L13) — `public actor SpeakerManager` declaration; the docstring documents the rationale for actor isolation (replacing a previous DispatchQueue that triggered `unsafeForcedSync` warnings and rare libmalloc heap corruption from concurrent `[Float]` COW mutations).
- [`SpeakerManager.swift:55`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerManager.swift#L55) — `initializeKnownSpeakers(_:mode:preserveIfPermanent:)` switches on `SpeakerInitializationMode` for collision handling.
- [`SpeakerManager.swift:128`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerManager.swift#L128) — `assignSpeaker(_:speechDuration:confidence:speakerThreshold:newName:)` — the central decision: match-and-update vs. create vs. drop.
- [`SpeakerManager.swift:275`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerManager.swift#L275) — `findMergeablePairs` runs the O(N²) pairwise cosine-distance comparison and prefers placing speaker1 as destination (preserving older ID order), reversing only when speaker1 is permanent.
- [`SpeakerManager.swift:425`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerManager.swift#L425) — `updateExistingSpeaker` applies the EMA update only when distance is below `embeddingThreshold` AND `vDSP_svesq` > 0.01; otherwise it just bumps duration + `updatedAt`.
- [`SpeakerManager.swift:489`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Clustering/SpeakerManager.swift#L489) — `nonisolated func cosineDistance` allows test code and synchronous callers to use the static `SpeakerUtilities.cosineDistance`.

## Edge Cases & Failure Modes
- Invalid embedding size: `assignSpeaker` returns nil and logs `"Invalid embedding size"` when count ≠ 256.
- Speech segment shorter than `minSpeechDuration` and no existing match closer than `speakerThreshold` → returns nil, logged at debug level (segment is dropped entirely).
- `initializeKnownSpeakers` silently skips speakers whose `currentEmbedding.count != 256` (warns but does not throw).
- `mergeSpeaker(_:into:)` no-ops if either ID is missing, if source == destination, or if `stopIfPermanent` and source is permanent.
- `findMergeablePairs` runs O(N²) per call — there is no spatial index for the cosine distance search; cost grows quadratically with speaker count.
- `confidence` parameter to `assignSpeaker` is accepted but **never used inside the method body**. [REVIEW: dead parameter; flagged for cleanup or wiring through to the update step.]
- `mergeSpeaker` removes the source speaker but does not reassign any external segments that may already reference its ID; callers must track this.
- `upsertSpeaker` updates `currentEmbedding` directly without L2-normalizing first — callers passing raw vectors will pollute the database.

## Performance / Concurrency Notes
- All mutation methods are actor-isolated; calls from sync contexts must await. The actor docstring is explicit about replacing the previous DispatchQueue approach to fix Swift 6 strict-concurrency warnings.
- Cosine distance is computed with `vDSP_dotpr` + `vDSP_svesq` via `SpeakerUtilities`. The fast path detects unit-norm vectors (sum of squares within 1e-3 of 1.0) and skips the square-root division.
- `nonisolated` `cosineDistance` enables pure-function reads but **does not** access actor state.
- Speaker storage is a Swift dictionary keyed by string IDs; ID iteration order is non-deterministic except where `speakerIds`/`permanentSpeakerIds` sort.

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/SpeakerManagerTests.swift` covers assignment, distance thresholds, EMA updates, and merge logic.
- `Tests/FluidAudioTests/Diarizer/SpeakerEnrollmentTests.swift` exercises `initializeKnownSpeakers` modes.
- `Tests/FluidAudioTests/Diarizer/SpeakerTests.swift` covers `Speaker`/`RawEmbedding` value-type behavior used by the manager.
- `Tests/FluidAudioTests/Diarizer/SpeakerOperationsTests.swift` validates the `SpeakerUtilities` cosine-distance and EMA helpers.

## Changelogs
