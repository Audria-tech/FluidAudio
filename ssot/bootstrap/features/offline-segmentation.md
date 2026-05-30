---
id: offline-segmentation
name: Offline Segmentation
repo: FluidAudio
status: active
linked_features: [offline-diarizer-manager, offline-diarizer-types]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Offline Segmentation

## TL;DR
`OfflineSegmentationProcessor` runs the pyannote segmentation model over a streaming audio source in 10-second batched windows (configurable step ratio), emits incremental `SegmentationChunk`s through an optional handler, and aggregates a `SegmentationOutput` with log-probabilities + per-frame soft speaker weights and histogram diagnostics.

## Motivation — why it exists

## Context

## What It Does
The processor accepts an `audioSource: AudioSampleSource` plus the segmentation model and config, computes `chunkSize = samplesPerWindow` and `stepSize = samplesPerStep`. It uses a sliding-window `[Float]` buffer with overlap reuse (when `stepSize < chunkSize`, the trailing `chunkSize - stepSize` samples are `memmove`d to the front and only `stepSize` new samples are read from the source). Batches up to 32 windows per CoreML call. Each prediction is decoded from a 7-class powerset (8 with empty) into `[chunk][frame][speaker(0..2)]` log-probs and soft weights, and (if a handler is set) yielded as a `SegmentationChunk` via `chunkHandler`. Diagnostic accumulators track speech-frame counts, empty-class probabilities, threshold histograms, and class histograms for debug logging. Includes a one-time `performWarmup` that runs a zero-filled chunk through the model to JIT compute units.

The `powerset` constant `[[], [0], [1], [2], [0,1], [0,2], [1,2], [0,1,2]]` has 8 entries (vs. 7 in the chunked `SegmentationProcessor`) — the all-three case is included for the community-1 weights. `WeightInterpolation` is used to resample soft weights when frame counts differ (matching scipy.ndimage.zoom half-pixel mapping).

## Key Code
- [`OfflineSegmentationProcessor.swift:44`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Segmentation/OfflineSegmentationProcessor.swift#L44) — `process(audioSource:segmentationModel:config:chunkHandler:)` — the streaming entry point.
- [`OfflineSegmentationProcessor.swift:97`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Segmentation/OfflineSegmentationProcessor.swift#L97) — `performWarmup` — pooled-buffer zero-fill + async prediction.
- [`OfflineSegmentationProcessor.swift:118`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Segmentation/OfflineSegmentationProcessor.swift#L118) — `populateWindow` — sliding-window reuse via `memmove` + `vDSP_vclr` for trailing zero-pad.
- [`OfflineSegmentationProcessor.swift:15`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Segmentation/OfflineSegmentationProcessor.swift#L15) — 8-entry powerset constant including `[0, 1, 2]`.

## Edge Cases & Failure Modes
- Throws `OfflineDiarizationError.noSpeechDetected` if `audioSource.sampleCount == 0`.
- Chunk handler return: `.continue` keeps streaming, `.stop` cancels via `chunkContinuation.terminated` — the manager respects this when its embedding-task consumer exits early.
- The streaming chunk emission is gated by `chunkEmissionEnabled` which can be flipped off once a downstream stop signal is received.
- Warmup is best-effort; failures are logged at debug level and segmentation proceeds.
- Diagnostic accumulators (`classHistogram`, `probabilityThresholdCounts`, etc.) are computed unconditionally — there's no `debugMode` gate, so the work is paid on every run. Small constant overhead. [REVIEW]
- The sliding-window reuse only kicks in when offsets are exactly `previousOffset + stepSize`; any out-of-order or skipped offset triggers a full reread.

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/Offline/SegmentationProcessorTests.swift` — basic behaviour.
- `Tests/FluidAudioTests/Diarizer/Offline/SegmentationProcessorEdgeCaseTests.swift` — boundary conditions including short audio and chunk-handler stop signal.

## Changelogs
