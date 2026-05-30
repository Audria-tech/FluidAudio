---
id: tdt-decoder
name: TDT Decoder (Token-and-Duration Transducer)
repo: FluidAudio
status: active
linked_features:
  - sliding-window-asr-manager
  - encoder-cache-manager
  - asr-types
  - token-deduplication
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# TDT Decoder (Token-and-Duration Transducer)

## TL;DR
The TDT package implements NVIDIA's Token-and-Duration Transducer decoding loop for the Parakeet TDT 0.6B model family (v2 / v3 / 110m / Ja). The decoder predicts both a token and a duration (how many encoder frames to skip), uses an inner blank-token loop to fly through silence without updating the LSTM, and ships ANE-aligned MLMultiArray buffers and Accelerate-based argmax for performance. The package also hosts the `AsrManager` actor (split into `+Pipeline`, `+TokenProcessing`, `+Transcription` extensions) and the `ChunkProcessor` for long-form sliding-window stateless decoding.

## Context

## What It Does
The decoder algorithm is in `TdtDecoderV3.swift`: for each encoder time-step, build the joint input (encoder frame + cached decoder output), pull a token and a duration index, decide whether the token is blank, and advance the time pointer by `durationBins[durationIndex]`. Non-blank tokens advance the decoder LSTM (`decoderModel`) and update `TdtDecoderState`; blank tokens skip without updating state (the optimization that makes TDT fast on silent regions). `TdtDecoderV2` is a thin shim that adapts the config and delegates to V3. Supporting types: `TdtConfig` (`blankId: 8192`, `durationBins: [0,1,2,3,4]`, `maxSymbolsPerStep: 10`, `boundarySearchFrames: 20`, `maxTokensPerChunk: 150`), `TdtDecoderState` (LSTM `hiddenState` + `cellState`, cached `predictorOutput`, `lastToken`, and `timeJump` for streaming continuity), `TdtHypothesis` (the produced `ySequence` / `timestamps` / `tokenConfidences` / `durations`), `TdtJointInputProvider` (`MLDictionaryFeatureProvider` wrapper), `TdtJointDecision` (logits → token + duration argmax), `TdtModelInference` (CoreML invocation boilerplate), `TdtFrameNavigation` (advance-by-duration math), `TdtDurationMapping`, `BlasIndex`, and `EncoderFrameView`.

`AsrManager.swift` is the public actor; the `+Pipeline.swift` extension drives preprocessor → encoder → TDT joint pipeline through `executeMLInferenceWithTimings(...)`, `+TokenProcessing.swift` handles confidence calculation and token-timing creation, and `+Transcription.swift` exposes `transcribeWithState(...)`, `transcribe(_:)`, and the `transcribeChunk(...)` entry used by sliding-window streaming. `AsrModels.swift` is the public model container with the `AsrModelVersion` enum (`.v2`, `.v3`, `.tdtCtc110m`, `.ctcZhCn`, `.tdtJa`) and `decoderLayerCount`. `ChunkProcessor.swift` runs the long-form stateless path: ~14.96 s windows, 2 s overlap, 80 ms left mel-context, with per-worker concurrent CoreML inference governed by `ASRConfig.parallelChunkConcurrency`.

## Key Code
- [`Decoder/TdtDecoderV3.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/Decoder/TdtDecoderV3.swift#L1) — algorithm overview comment + the main decode loop.
- [`Decoder/TdtConfig.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/Decoder/TdtConfig.swift#L1) — `TdtConfig` with `blankId: 8192` and `durationBins`.
- [`Decoder/TdtDecoderState.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/Decoder/TdtDecoderState.swift#L5) — LSTM state struct with `timeJump` for streaming continuity.
- [`Decoder/TdtHypothesis.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/Decoder/TdtHypothesis.swift#L1) — output hypothesis with `ySequence`, timestamps, confidences.
- [`Decoder/TdtJointDecision.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/Decoder/TdtJointDecision.swift#L1) — logits → (token, duration) argmax.
- [`AsrManager.swift:6`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/AsrManager.swift#L6) — `public actor AsrManager` declaration + `decoderLayerCount`.
- [`AsrManager+Pipeline.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/AsrManager%2BPipeline.swift#L5) — `executeMLInferenceWithTimings(...)` orchestrates preprocessor → encoder → TDT.
- [`AsrManager+Transcription.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/AsrManager%2BTranscription.swift#L5) — `transcribeWithState(_:decoderState:language:)` short-path for single-chunk audio.
- [`AsrModels.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/AsrModels.swift#L5) — `AsrModelVersion` enum.
- [`ChunkProcessor.swift:3`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/ChunkProcessor.swift#L3) — stateless chunking with 2 s overlap and `melContextSamples: 1280`.

## Edge Cases & Failure Modes
- `TdtConfig.consecutiveBlankLimit: 5` triggers early termination on the last chunk if the model keeps emitting blanks (prevents infinite tail-of-silence decoding).
- `maxTokensPerChunk: 150` caps decoding per chunk to prevent runaway loops on degenerate joint outputs.
- `TdtDecoderState.timeJump` carries decoder progress *relative to encoder frames* across chunks — wrong values produce silent token drops at chunk boundaries.
- V2 path is a delegation shim — any config drift between V2 and V3 lives in `adaptConfigForV2(...)`.
- The 1-layer vs 2-layer LSTM split (tdtCtc110m vs the rest) is surfaced through `AsrModels.version.decoderLayers` — wrong sizing causes shape errors at first decoder call.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Parakeet/SlidingWindow/TDT/AsrManagerTests.swift`, `AsrTranscriptionTests.swift`, `ChunkProcessorTests.swift`, `ChunkProcessorEdgeCaseTests.swift`, and the `Decoder/*Tests.swift` set (`TdtDecoderChunkTests`, `TdtDecoderStateV3Tests`, `TdtDecoderTests`, `TdtDecoderV2Tests`, `TdtRefactoredComponentsTests`, `DecoderStateTests`, `BlasIndexAndHypothesisTests`).

## Changelogs
