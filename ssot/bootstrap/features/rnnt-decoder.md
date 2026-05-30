---
id: rnnt-decoder
name: RNN-T Greedy Decoder (Parakeet EOU)
repo: FluidAudio
status: active
linked_features:
  - eou-detector
  - parakeet-tokenizer
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# RNN-T Greedy Decoder (Parakeet EOU)

## TL;DR
`RnntDecoder` is the greedy RNN-Transducer decoding loop used by the Parakeet EOU streaming pipeline. It owns the decoder LSTM hidden/cell state, walks an encoder output of shape `[1, 512, T]` frame-by-frame, drives the decoder + joint CoreML models, and emits token IDs while also signalling End-of-Utterance (token ID 1024) so the upper-level manager can flush text.

## Context

## What It Does
`decodeWithEOU(encoderOutput:timeOffset:skipFrames:validOutLen:)` returns a `DecodeResult` carrying `tokenIds` and an `eouDetected` flag. For each encoder time-step (clamped by `validOutLen` to match NeMo's streaming truncation), it slices `encoderOutput[:, :, t]` into a `[1, 640, 1]` step, prepares the decoder input with `lastToken / h_in / c_in`, runs the decoder model, slices the last decoder frame (NeMo convention `decoder_step[:, :, -1:]`), and runs the joint model. The joint outputs `token_id` (already argmaxed). Blank (`1026`) breaks the inner symbol loop; EOU (`1024`) sets `eouDetected = true` and breaks outerLoop; otherwise the token is appended, `h_out`/`c_out` are reassigned, and the inner loop tries up to `maxSymbolsPerStep=2` extra emissions. `resetState()` zeros both LSTM tensors and resets `lastToken` to the blank ID — used by the upper-level manager on EOU and `reset()`.

## Key Code
- [`RnntDecoder.swift:14`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/RnntDecoder.swift#L14) — class with blank ID `1026`, EOU ID `1024`, `maxSymbolsPerStep=2`, hidden size 640, 1 LSTM layer.
- [`RnntDecoder.swift:63`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/RnntDecoder.swift#L63) — `decodeWithEOU` main loop including `validOutLen` truncation and `skipFrames` for overlap handling.
- [`RnntDecoder.swift:146`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/RnntDecoder.swift#L146) — `extractEncoderStep` stride-aware copy.
- [`RnntDecoder.swift:190`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/RnntDecoder.swift#L190) — `sliceDecoderStep` keeps only the last decoder time frame.

## Edge Cases & Failure Modes
- Missing CoreML outputs (`decoder`, `token_id`, `h_out`, `c_out`) throw `RnntDecoderError.missingOutput(name)`.
- Stride math reads `encoderOutput.strides[0/1/2]` so reshapes that don't follow `[1, D, T]` C-contiguous layout would alias incorrectly — there is no shape assertion beyond the inferred dims.
- `maxSymbolsPerStep=2` caps emissions per encoder frame to prevent runaway loops on degenerate joint outputs.
- `lastToken` is initialised to blank — first-call decoder gets a "no previous token" prior, matching NeMo's SOS handling.

## Test Coverage
- Exercised indirectly through `Tests/FluidAudioTests/ASR/Parakeet/Streaming/StreamingAsrManagerTests.swift` (EOU path).

## Changelogs
