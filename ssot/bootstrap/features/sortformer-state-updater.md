---
id: sortformer-state-updater
name: Sortformer State Updater
repo: FluidAudio
status: active
linked_features: [sortformer-diarizer, sortformer-types, sortformer-model-inference]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Sortformer State Updater

## TL;DR
`SortformerStateUpdater` ports NeMo's `StateUpdater` algorithm: it consumes chunk embeddings + predictions from the main model, slices out the core-frame predictions, appends to FIFO, compresses the speaker cache when it overflows via log-pred-score top-k selection, and returns confirmed + tentative predictions.

## Motivation — why it exists

## Context

## What It Does
`streamingUpdate(state:&:chunk:preds:leftContext:rightContext:)` is the main entry point. It extracts FIFO predictions (slice `[spkcache..spkcache+fifo]` from preds), extracts core-frame chunk embeddings (skipping `leftContext` frames), and extracts confirmed chunk predictions for `coreFrames` frames followed by tentative predictions for `rightContext` frames. The core embeddings are appended to FIFO; once FIFO exceeds `fifoCapacity`, `popOutLength = clamp(spkcacheUpdatePeriod, contextLength - fifoCapacity, contextLength)` frames are popped and (a) absorbed into the silence-profile via `updateSilenceProfile` (frames with `probSum < silenceThreshold` update the running mean silence embedding), (b) appended to `spkcache`. If `spkcacheLength` then exceeds `spkcacheCapacity`, `compressSpkcache` is invoked.

`compressSpkcache` computes per-frame log-based prediction scores `log(p) - log(1-p) + sum_others(log(1-p)) - log(0.5)` via vDSP/vForce, disables non-speech and overlapped-speech scores (`disableLowScores`), boosts the most-recent frames by `scoresBoostLatest`, applies a strong boost to the top-k=`strongBoostRate * (spkcacheLenPerSpk - silFramesPerSpk)` per speaker, then a weak boost, appends `silFramesPerSpk` infinity-scored placeholders per speaker, and uses `getTopKIndices` to select `spkcacheCapacity` frames (with permute-then-flatten-then-modulo arithmetic mirroring the NeMo reference). Disabled top-k slots are filled with the mean silence embedding; real top-k slots copy embeddings + preds from the old spkcache.

## Key Code
- [`SortformerStateUpdater.swift:8`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerStateUpdater.swift#L8) — `public struct SortformerStateUpdater`; this is a value type, not actor-isolated.
- [`SortformerStateUpdater.swift:31`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerStateUpdater.swift#L31) — `streamingUpdate(state:inout:chunk:preds:leftContext:rightContext:)` is the only public method.
- [`SortformerStateUpdater.swift:175`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerStateUpdater.swift#L175) — `updateSilenceProfile` running mean update.
- [`SortformerStateUpdater.swift:220`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerStateUpdater.swift#L220) — `compressSpkcache` orchestration.
- [`SortformerStateUpdater.swift:311`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerStateUpdater.swift#L311) — `getLogPredScores` uses `vDSP.clip`, `vForce.log`, `vForce.log1p`, `vDSP.subtract`, `vDSP.add` for the score formula.
- [`SortformerStateUpdater.swift:465`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerStateUpdater.swift#L465) — `getTopKIndices` mirrors the NeMo permute-then-flatten-then-modulo construction, then disables frames `>= nFramesNoSil`.

## Edge Cases & Failure Modes
- Throws `SortformerError.insufficientPredsLength` and `.insufficientChunkLength` when `preds`/`chunk` are shorter than the slice math requires.
- The "FIFO predictions nil after update" path logs `"THIS SHOULD NEVER HAPPEN!"` and returns the current chunk's preds without compressing — a defensive fallback that masks an internal invariant violation. [REVIEW]
- The score-boost top-k tie-break is "smaller index wins" (older frames preferred); this matches the NeMo Python tie-breaking but is implementation-defined.
- **State isolation**: `SortformerStateUpdater` is a `struct` (value type) and its methods take `state: inout SortformerStreamingState`. The caller (`SortformerDiarizer`) holds an NSLock; the updater itself has no isolation. Concurrent calls to `streamingUpdate` from different threads on the *same* `state` value would race. The struct contains no mutable internal state beyond `config: let` and a `let logger`, so the struct itself is concurrency-safe; only the `state` parameter is the shared resource. [REVIEW: confirmed — the updater is a pure struct; thread safety is the caller's responsibility via NSLock. This matches the user's flagged concern.]
- The silence-profile running mean uses a per-element division by `newN`; for very long-running streams this could lose precision but the buffer-overflow compression keeps spkcacheLength bounded.

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/Sortformer/SortformerStateUpdaterTests.swift` — direct tests of `streamingUpdate`, score computation, top-k selection.
- `Tests/FluidAudioTests/Diarizer/Sortformer/SortformerStreamingIntegrationTests.swift` — round-trip behavior via `SortformerDiarizer`.

## Changelogs
