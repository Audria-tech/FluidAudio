---
id: eou-detector
name: EOU Streaming Detector
repo: FluidAudio
status: active
linked_features:
  - streaming-asr-manager
  - rnnt-decoder
  - encoder-cache-manager
  - streaming-asr-utils
  - parakeet-tokenizer
  - parakeet-model-variant
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# EOU Streaming Detector

## TL;DR
`StreamingEouAsrManager` is the Parakeet EOU 120M streaming engine — a cache-aware loopback encoder + RNN-T decoder + joint pipeline that emits tokens as audio arrives and signals End-of-Utterance via token ID 1024. It conforms to `StreamingAsrManager` and supports three latency tiers (160 ms / 320 ms / 1280 ms) selected at construction via `StreamingChunkSize`.

## Context

## What It Does
The file defines both `StreamingChunkSize` (with chunk sample counts, mel frame counts, `validOutLen`, pre-cache size, and audio shift derived from NeMo's `setup_streaming_params`) and the manager actor. `appendAudio(_:)` resamples and appends to an internal audio buffer; `processBufferedAudio()` slices off complete chunks of `chunkSamples`, runs the native Swift `AudioMelSpectrogram` (for exact NeMo parity), feeds the streaming encoder with the encoder cache + mel-cache state, then drives `RnntDecoder.decodeWithEOU` against the encoder output. New tokens are appended to `accumulatedTokenIds`, decoded by the `Tokenizer`, and surfaced through `PartialCallback`. When the decoder returns `eouDetected: true`, the accumulated transcript is delivered to `EouCallback` and the decoder state is reset (the audio buffer/cache continue).

The chunk-size knobs are critical: `melFrames` and `validOutLen` are baked into the NeMo CoreML conversion (separate exported models per chunk size), and `shiftSamples` controls the audio overlap. For 160 ms the model is trained with 50 % overlap (1280 samples shift); 320 ms uses 5120 samples (32 mel frames × 160); 1280 ms uses zero audio overlap and relies on the 16-mel-frame pre-cache instead.

## Key Code
- [`StreamingEouAsrManager.swift:17`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/EOU/StreamingEouAsrManager.swift#L17) — `StreamingChunkSize` enum + per-variant config doc.
- [`StreamingEouAsrManager.swift:104`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/EOU/StreamingEouAsrManager.swift#L104) — `chunkSamples` derived from mel-frame math.
- [`StreamingEouAsrManager.swift:138`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/EOU/StreamingEouAsrManager.swift#L138) — `shiftSamples` (per-variant audio shift logic).
- [`StreamingEouAsrManager.swift:165`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/EOU/StreamingEouAsrManager.swift#L165) — `StreamingEouAsrManager` actor declaration with `streamingEncoder` (single loopback model), `RnntDecoder`, native `AudioMelSpectrogram`.

## Edge Cases & Failure Modes
- Mel computation is native Swift for "exact NeMo parity"; even small float drift here corrupts the encoder cache.
- EOU detection resets the RNN-T decoder but not the encoder cache — assumption is that the next utterance still benefits from acoustic context.
- For 160 ms with 50 % audio overlap, the same audio is run through the encoder twice — `chunkSamples` (2560) but `shiftSamples` (1280) means each new processBufferedAudio call only consumes 1280 fresh samples. This is the trained behavior.
- The 320 ms variant has the most internal divergence from the others — `chunk_size = [57, 64]` from NeMo's `setup_streaming_params(8, 4)`, code uses the index `[1]` = 64 mel frames.
- Switching variants requires loading a different CoreML model and (re-)downloading from the matching HuggingFace subrepo.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Parakeet/Streaming/StreamingAsrManagerTests.swift` (covers the EOU path via `StreamingAsrManager`).
- `Tests/FluidAudioTests/ASR/Parakeet/Streaming/EouChunkSizeFrameCountTests.swift` (chunk-size math).
- `Tests/FluidAudioTests/ASR/Parakeet/Streaming/AudioMelSpectrogramTests.swift`.

## Changelogs
