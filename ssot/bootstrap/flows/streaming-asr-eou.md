---
id: streaming-asr-eou
name: Streaming ASR — Parakeet EOU
trigger_type: event
trigger: Application uses a `StreamingEouAsrManager` (factory: `StreamingModelVariant.createManager()` with an EOU variant) and pushes audio via `appendAudio(_:)`.
end_state: Tokens stream via the partial callback; when the model emits token ID 1024 (EOU), the accumulated transcript is delivered to the `EouCallback` and the decoder LSTM is reset for the next utterance.
involves_features: [streaming-asr-manager, eou-detector, rnnt-decoder]
linked_flows: [streaming-asr-parakeet, model-download-and-load]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Streaming ASR — Parakeet EOU

## TL;DR
The EOU 120M variant of the streaming Parakeet pipeline. Same cache-aware encoder + RNN-T decoder loop as the generic streaming flow, but the model is trained to emit a special end-of-utterance token (`1024`) which the decoder loop surfaces as `eouDetected: true` so the upper-level manager can flush text and reset decoder state without losing acoustic context.

## Trigger & Preconditions
`StreamingEouAsrManager` constructed with a `StreamingChunkSize` (160 ms / 320 ms / 1280 ms — each maps to a separately-exported CoreML model with bespoke mel-frame and pre-cache configuration). `loadModels()` must have completed. Audio fed via `appendAudio(_:)` at any sample rate (resampled to 16 kHz internally).

## Stages
1. **Buffer + chunk gate** — `appendAudio(_:)` appends to internal `[Float]`; `processBufferedAudio()` peels off `chunkSamples` at the variant's pace: [`Sources/FluidAudio/ASR/Parakeet/Streaming/EOU/StreamingEouAsrManager.swift:104`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/EOU/StreamingEouAsrManager.swift#L104).
2. **Native mel** — `AudioMelSpectrogram.computeFlatTransposed(...)` runs in pure Swift for "exact NeMo parity"; even minor float drift here corrupts the encoder cache: [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:299`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L299).
3. **Streaming encoder + cache** — single loopback encoder receives mel + the three encoder caches; outputs include the new cache state via `_out` suffixed names — `EncoderCacheManager.extractCachesFromOutput(...)` rotates them for the next chunk. The 160 ms variant uses 50% audio overlap (`shiftSamples = 1280`) so a chunk advance only consumes 1280 fresh samples: [`Sources/FluidAudio/ASR/Parakeet/Streaming/EOU/StreamingEouAsrManager.swift:138`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/EOU/StreamingEouAsrManager.swift#L138).
4. **RNN-T loop** — `RnntDecoder.decodeWithEOU` walks encoder time-steps, slices `[1, 640, 1]`, runs decoder + joint; blank (`1026`) breaks the inner symbol loop; EOU (`1024`) sets `eouDetected = true` and breaks the outer loop: [`Sources/FluidAudio/ASR/Parakeet/Streaming/RnntDecoder.swift:63`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/RnntDecoder.swift#L63).
5. **Token append + partial emit** — non-blank tokens are appended to `accumulatedTokenIds`; `Tokenizer.decode(ids:)` produces an incremental string; `PartialCallback` fires.
6. **EOU branch** — when `eouDetected == true`, the manager invokes `EouCallback(transcript)` with the accumulated text, calls `RnntDecoder.resetState()` to zero the LSTM hidden/cell and reset `lastToken = blankId`, but **leaves the encoder cache untouched** (next utterance still benefits from acoustic context): [`Sources/FluidAudio/ASR/Parakeet/Streaming/RnntDecoder.swift:14`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/RnntDecoder.swift#L14).
7. **Continue or finish** — manager continues processing further audio; `finish()` returns the cumulative transcript.

## Outputs
- Partial transcript `String` via `PartialCallback`.
- Per-utterance final transcript `String` via `EouCallback` when token 1024 fires.
- Cumulative `String` from `finish()` at end-of-stream.

## Error Modes
- Variant mismatch: changing `StreamingChunkSize` requires re-downloading the matching HuggingFace subrepo and re-loading the model.
- 320 ms variant uses NeMo's `setup_streaming_params(8, 4)` shape that diverges from the 160/1280 ms paths; bespoke handling in `chunk_size = [57, 64]` lookup.
- Mel-cache + encoder-cache must round-trip in float — drift across chunks degrades EOU detection.
- Inherits all `streaming-asr-parakeet.md` error modes.

## Prompts and Models Used

## Usage Metrics

## Changelogs
