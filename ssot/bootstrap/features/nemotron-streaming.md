---
id: nemotron-streaming
name: Nemotron Streaming ASR
repo: FluidAudio
status: active
linked_features:
  - streaming-asr-manager
  - encoder-cache-manager
  - streaming-asr-utils
  - parakeet-tokenizer
  - parakeet-model-variant
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Nemotron Streaming ASR

## TL;DR
`StreamingNemotronAsrManager` is the cache-aware streaming engine for NVIDIA's Nemotron Speech 0.6B model. It conforms to `StreamingAsrManager` and supports four latency tiers (80 ms / 160 ms / 560 ms / 1120 ms) via `NemotronChunkSize`. Per-chunk configuration (mel frames, pre-encode cache, encoder cache shapes, decoder hidden size) is loaded from a `metadata.json` shipped with each model so the runtime adapts without recompilation.

## Context

## What It Does
The manager owns four CoreML models (preprocessor, encoder, decoder, joint), a tokenizer, the encoder cache trio (`cacheChannel`/`cacheTime`/`cacheLen`), the mel cache (last 9 frames from the previous chunk), and the decoder LSTM state. `appendAudio(_:)` resamples and buffers; `processBufferedAudio()` peels off `chunkSamples` (== `chunkMelFrames * 160`) samples at a time, runs the preprocessor (mel + previous mel cache → encoder input), feeds the encoder with the three cache arrays, rotates `EncoderCacheManager.extractCachesFromOutput(...)`, and drives the RNN-T decoder + joint loop manually (matching the EOU pattern but without an EOU token). Decoded tokens are appended to `accumulatedTokenIds` and emitted via `NemotronPartialCallback`.

`NemotronStreamingConfig.swift` defines the metadata.json schema (sample rate, mel features, chunk mel frames, pre-encode cache, vocab/blank IDs, encoder/decoder dims, LSTM layer count, cache shapes). A default `init()` ships values for the 1120 ms variant for backward compatibility. `NemotronChunkSize.swift` is a small enum mapping each variant to its `Repo` and HuggingFace subdirectory; `NemotronEncoder.fileName = "encoder_int8.mlmodelc"` is hard-coded because Nemotron ships int8-only encoders.

The `+Pipeline.swift` extension contains the per-chunk inference orchestration (preprocessor call, encoder call with cache rotation, decoder + joint loop, callback invocation).

## Key Code
- [`StreamingNemotronAsrManager.swift:10`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/Nemotron/StreamingNemotronAsrManager.swift#L10) — actor declaration with cache state, mel cache, decoder LSTM state.
- [`StreamingNemotronAsrManager+Pipeline.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/Nemotron/StreamingNemotronAsrManager%2BPipeline.swift#L1) — per-chunk inference loop.
- [`NemotronStreamingConfig.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/Nemotron/NemotronStreamingConfig.swift#L5) — config struct + default 1120 ms init.
- [`NemotronChunkSize.swift:4`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/Nemotron/NemotronChunkSize.swift#L4) — `NemotronChunkSize` enum (ms80 / ms160 / ms560 / ms1120) + repo + subdirectory mapping.

## Edge Cases & Failure Modes
- Encoder is int8-only — there is no FP32 fallback path. Devices that fail to compile the int8 model error out.
- `cacheChannelShape` and `cacheTimeShape` are loaded from `metadata.json` per chunk size — mismatches between metadata and CoreML model produce shape errors at first prediction.
- Mel cache holds 9 frames per the default; rotated each chunk to provide encoder left-context.
- `requestedChunkSize` is set externally by `StreamingModelVariant.createManager()` and consumed during `loadModels()` to pick the HuggingFace subrepo.
- `NemotronChunkSize` exposes `ms80` and `ms160` but `StreamingModelVariant` only surfaces 560 / 1120 — [REVIEW: dead enum cases or future expansion?]

## Test Coverage
- `Tests/FluidAudioTests/ASR/Parakeet/Streaming/StreamingNemotronAsrManagerTests.swift`, `NemotronChunkSizeTests.swift`, `NemotronStreamingConfigTests.swift`, `NemotronBenchmarkTests.swift`.

## Changelogs
