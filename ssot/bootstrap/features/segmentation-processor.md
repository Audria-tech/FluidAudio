---
id: segmentation-processor
name: Segmentation Processor
repo: FluidAudio
status: active
linked_features: [diarizer-manager, segmentation-sliding-window]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Segmentation Processor

## TL;DR
`SegmentationProcessor` runs the pyannote segmentation CoreML model on a 10-second audio chunk (160k samples at 16 kHz), reads the powerset output, and decodes it into a binarized `[batch][frame][speaker]` activity tensor plus a `SlidingWindowFeature` for time-locked downstream use.

## Motivation — why it exists

## Context

## What It Does
`getSegments(audioChunk:segmentationModel:chunkSize:)` allocates an ANE-aligned `MLMultiArray` of shape `[1, 1, chunkSize]`, copies up to `chunkSize` samples via `vDSP_mmov`, zero-fills the remainder with `vDSP_vfill`, calls `prefetchToNeuralEngine`, and runs the model. The `segments` output (`MLMultiArray` of shape `[1, frames, 7]`) is processed in `processSegmentsOptimized`: it copies frame-by-frame into a nested `[[[Float]]]` (using `vDSP_mmov` per row), then calls `powersetConversionOptimized` which finds the argmax class (via `vDSP_maxvi`) per frame and looks up the 3-speaker activity vector from the hard-coded powerset table `[[], [0], [1], [2], [0,1], [0,2], [1,2]]`. The optimized path writes results into another ANE-aligned `MLMultiArray` first; falls back to `powersetConversionFallback` if alignment fails. `createSlidingWindowFeature(binarizedSegments:chunkOffset:)` wraps the binarized tensor with `SlidingWindow(start: chunkOffset, duration: 0.0619375s = 991 samples, step: 0.016875s = 270 samples)` — model output rate ~59.26 fps.

## Key Code
- [`SegmentationProcessor.swift:24`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Segmentation/SegmentationProcessor.swift#L24) — `getSegments(audioChunk:segmentationModel:chunkSize:)` ANE-aligned alloc + run.
- [`SegmentationProcessor.swift:113`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Segmentation/SegmentationProcessor.swift#L113) — `powersetConversionOptimized` argmax-then-lookup against the 7-entry powerset table.
- [`SegmentationProcessor.swift:224`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Segmentation/SegmentationProcessor.swift#L224) — `createSlidingWindowFeature` — note hard-coded `duration: 0.0619375` and `step: 0.016875` for community-1.

## Edge Cases & Failure Modes
- Throws `DiarizerError.processingFailed("Missing segments output from segmentation model")` if the CoreML model doesn't expose `segments`.
- `chunkSize` defaults to 160_000 but is parameterized; callers may pass a smaller value to support alternative weights.
- Audio shorter than `chunkSize` is zero-padded; the model sees a partly-silent window.
- The 7-entry powerset is hard-coded — adapting to different speaker counts requires editing this constant. [REVIEW: any new community-N weights with > 3 speakers would silently mis-decode.]
- The "optimized" path falls back to `powersetConversionFallback` if `createAlignedArray` fails (e.g., memory pressure), but the result tensor is identical.

## Test Coverage
- Exercised indirectly through `Tests/FluidAudioTests/Diarizer/Offline/SegmentationProcessorTests.swift` and `SegmentationProcessorEdgeCaseTests.swift` (which target the offline variant but share semantics for the powerset decode).
- `Tests/FluidAudioTests/Diarizer/SpeakerEnrollmentTests.swift` uses this processor end-to-end via `DiarizerManager`.

## Changelogs
