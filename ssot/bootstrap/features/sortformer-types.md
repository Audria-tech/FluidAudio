---
id: sortformer-types
name: Sortformer Types
repo: FluidAudio
status: active
linked_features: [sortformer-diarizer, sortformer-state-updater, sortformer-model-inference]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Sortformer Types

## TL;DR
Value types for the Streaming Sortformer pipeline: `SortformerConfig` (architecture + streaming + thresholds, with preset configs for fast/balanced/highContext × v2/v2.1), `SortformerStreamingState` (spkcache + FIFO + silence profile), `SortformerFeatureLoader` (offline mel-feature chunker), `StreamingUpdateResult` (confirmed/tentative split), and `SortformerError`.

## Motivation — why it exists

## Context

## What It Does
`SortformerConfig` is `Sendable` and pins `numSpeakers=4`, `preEncoderDims=512`, `subsamplingFactor=8`, plus configurable `chunkLen`, `chunkLeftContext`, `chunkRightContext`, `fifoLen`, `spkcacheLen`, `spkcacheUpdatePeriod`. Audio params (`sampleRate=16000`, `melWindow=400`, `melStride=160`, `melFeatures=128`) and thresholds (`silenceThreshold=0.2`, `predScoreThreshold=0.25`, `scoresBoostLatest=0.05`, `strongBoostRate=0.75`, `weakBoostRate=1.5`, `minPosScoresRate=0.5`, `spkcacheSilFramesPerSpk=3`, `maxIndex=99999`) round out the configuration. Computed properties: `chunkMelFrames = (chunkLen+lc+rc)*subsampling`, `coreFrames = chunkLen*subsampling`, `frameDurationSeconds = subsampling*melStride/sampleRate`. The `init` clamps `spkcacheLen >= (1+silFramesPerSpk)*numSpeakers` and `spkcacheUpdatePeriod` into `[chunkLen, fifoLen+chunkLen]`. Eight preset configs (`.default`, `.fastV2`, `.fastV2_1`, `.balancedV2`, `.balancedV2_1`, `.highContextV2`, `.highContextV2_1`) plus an `isCompatible(with:)` check that compares `chunkMelFrames`, `fifoLen`, `spkcacheLen`.

`SortformerStreamingState` holds `spkcache`/`fifo` as flat `[Float]` arrays (logical shape `[len, fcDModel]`), corresponding `*Length: Int`, optional `*Preds` arrays (logical `[len, numSpeakers]`), `meanSilenceEmbedding: [Float]` (fcDModel-sized), and `silenceFrameCount: Int`. Init reserves capacity for both buffers. `cleanup()` deallocates everything.

`SortformerFeatureLoader` is the offline batch-mode mel chunker. It computes mel features for the entire audio up front via `AudioMelSpectrogram().computeFlatTransposed(audio:)`, then `next()` returns `(chunkFeatures, chunkLength, leftOffset, rightOffset)` for each chunk respecting the same full-right-context gate as the streaming path. `numChunks` is `(featLength - rc) / chunkLen`.

`StreamingUpdateResult` is the explicit confirmed-vs-tentative carrier; provides `confirmedFrameCount` / `tentativeFrameCount` divisors. `SortformerError` covers `notInitialized`, model/preproc/inference failures, `invalidAudioData`/state/configuration, and `insufficient{Chunk,Preds}Length`.

## Key Code
- [`SortformerTypes.swift:9`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerTypes.swift#L9) — `SortformerConfig` declaration.
- [`SortformerTypes.swift:198`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerTypes.swift#L198) — `init` constraint-enforcement on `spkcacheLen` and `spkcacheUpdatePeriod`.
- [`SortformerTypes.swift:247`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerTypes.swift#L247) — `SortformerStreamingState` flat-array layout.
- [`SortformerTypes.swift:339`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerTypes.swift#L339) — `SortformerFeatureLoader.next()` chunk emission with full-right-context gate.

## Edge Cases & Failure Modes
- The `numSpeakers=4` and `preEncoderDims=512` constants are immutable `let`s — pipelines for different model dimensions require source changes.
- `SortformerStreamingState.spkcachePreds` is initially `nil`; it only becomes non-nil once the spkcache overflows for the first time (a soft state flag).
- `SortformerFeatureLoader.numChunks` can be 0 for audio shorter than `coreFrames + rc`; `next()` correctly returns nil immediately.
- `isCompatible(with:)` does not check `spkcacheUpdatePeriod` or any threshold — only the three structural sizes that affect MLMultiArray shape. [REVIEW: switching `chunkLeftContext` while keeping `chunkMelFrames` constant would still pass this check.]
- `StreamingUpdateResult.confirmedFrameCount` assumes `numSpeakers=4` (the inline comment notes this); the divisor is `numSpeakers` not a hard 4, so it's actually correct for any value, but the comment is misleading.

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/Sortformer/SortformerTypesTests.swift` — config presets, constraint clamping, state init.

## Changelogs
