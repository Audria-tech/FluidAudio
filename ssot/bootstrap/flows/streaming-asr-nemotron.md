---
id: streaming-asr-nemotron
name: Streaming ASR — Nemotron
trigger_type: event
trigger: Application uses a `StreamingNemotronAsrManager` (factory: `StreamingModelVariant.createManager()` with a Nemotron variant) and pushes audio via `appendAudio(_:)`.
end_state: Tokens stream via the partial callback as cached chunks are processed; `finish()` returns the cumulative transcript and clears mid-stream state.
involves_features: [streaming-asr-manager, nemotron-streaming, rnnt-decoder]
linked_flows: [streaming-asr-parakeet, model-download-and-load]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Streaming ASR — Nemotron

## TL;DR
The Nemotron Speech 0.6B variant of the streaming Parakeet pipeline. Four CoreML models (`preprocessor`, `encoder_int8.mlmodelc`, `decoder`, `joint`) plus per-variant `metadata.json` driving mel/cache shapes. Four latency tiers (80 / 160 / 560 / 1120 ms). No EOU token — utterances are bounded by the caller via `finish()`.

## Trigger & Preconditions
`StreamingNemotronAsrManager(configuration:requestedChunkSize:)` constructed via `StreamingModelVariant.createManager()`. `requestedChunkSize` is set externally and consumed in `loadModels()` to pick the HF subrepo. CoreML int8 encoder must compile on the target device — no FP32 fallback.

## Stages
1. **Load metadata** — `loadModels()` fetches the variant's HF subrepo and reads `metadata.json` into `NemotronStreamingConfig` (sample rate, mel features, chunk mel frames, pre-encode cache, encoder cache shapes, decoder dims, LSTM layers): [`Sources/FluidAudio/ASR/Parakeet/Streaming/Nemotron/NemotronStreamingConfig.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/Nemotron/NemotronStreamingConfig.swift#L5).
2. **Buffer + chunk gate** — `appendAudio(_:)` resamples and appends; `processBufferedAudio()` peels `chunkSamples = chunkMelFrames * 160` per call: [`Sources/FluidAudio/ASR/Parakeet/Streaming/Nemotron/StreamingNemotronAsrManager.swift:10`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/Nemotron/StreamingNemotronAsrManager.swift#L10).
3. **Preprocessor** — runs mel extraction with the previous chunk's mel cache (last 9 frames) prepended → encoder input. Mel cache is rotated each chunk to provide encoder left-context.
4. **Encoder + cache rotation** — int8 encoder model receives mel + three cache arrays (`cacheChannel`/`cacheTime`/`cacheLen`); `EncoderCacheManager.extractCachesFromOutput(...)` rotates them for the next chunk: [`Sources/FluidAudio/ASR/Parakeet/Streaming/EncoderCacheManager.swift:58`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/EncoderCacheManager.swift#L58).
5. **RNN-T decode** — `+Pipeline.swift` runs the manual decoder + joint AR loop (mirroring the EOU pattern but without an EOU token). Decoder LSTM state carried across steps: [`Sources/FluidAudio/ASR/Parakeet/Streaming/Nemotron/StreamingNemotronAsrManager+Pipeline.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/Nemotron/StreamingNemotronAsrManager%2BPipeline.swift#L1).
6. **Token append + partial emit** — decoded tokens appended to `accumulatedTokenIds`; `Tokenizer.decode(ids:)` produces incremental text; `NemotronPartialCallback` fires.
7. **Finish** — `finish()` drains the buffer (no EOU bound), returns final `String`. `reset()` clears mel cache, encoder caches, decoder LSTM state.

## Outputs
- Partial transcript `String` via `NemotronPartialCallback`.
- Final transcript `String` from `finish()`.

## Error Modes
- Int8 encoder compile failure → no fallback.
- `metadata.json` shape mismatch with the CoreML model produces shape errors at first prediction.
- `NemotronChunkSize.ms80` / `.ms160` enum cases exist but `StreamingModelVariant` only surfaces 560 / 1120 [REVIEW noted in feature.md].
- Inherits `streaming-asr-parakeet.md` error modes.

## Prompts and Models Used

## Usage Metrics

## Changelogs
