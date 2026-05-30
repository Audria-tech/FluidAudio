---
id: streaming-asr-parakeet
name: Streaming ASR — Parakeet (Generic)
trigger_type: event
trigger: Application pushes audio buffers into a `StreamingAsrManager` via `appendAudio(_:)` and periodically calls `processBufferedAudio()`.
end_state: Tokens are emitted progressively through the partial-transcript callback as audio arrives; on `finish()` a final transcript is returned and decoder/encoder state is cleared.
involves_features: [streaming-asr-manager, rnnt-decoder, encoder-cache-manager, parakeet-tokenizer, token-deduplication, punctuation-commit-layer]
linked_flows: [model-download-and-load, model-warmup, streaming-asr-eou, streaming-asr-nemotron]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Streaming ASR — Parakeet (Generic)

## TL;DR
Cache-aware streaming engines (EOU, Nemotron) share a single `StreamingAsrManager` protocol — feed audio via `appendAudio(_:)`, advance inference via `processBufferedAudio()`, get tokens via the `setPartialTranscriptCallback(_:)` callback. Behind the scenes a chunked encoder retains state via the `EncoderCacheManager` cache trio, an RNN-T decoder loops with its own LSTM cache, and the `Tokenizer` detokenizes incrementally.

## Trigger & Preconditions
Created via `SlidingWindowAsrSession.createEngine(variant:source:)` (which dispatches through `StreamingModelVariant.createManager()`) or constructed directly. `loadModels()` must succeed first. Audio resampled to 16 kHz mono Float32 inside `appendAudio`.

## Stages
1. **Append audio** — `appendAudio(_:)` synchronously resamples (if needed) and appends to the engine's internal `[Float]` buffer. No model call here: [`Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrManager.swift:34`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrManager.swift#L34).
2. **Chunk gate** — `processBufferedAudio()` slices off complete `chunkSamples` (per-variant: 160/320/560/1120/1280 ms) and only proceeds when one full chunk is available: [`Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrManager.swift:40`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrManager.swift#L40).
3. **Mel features** — native Swift `AudioMelSpectrogram.computeFlatTransposed(...)` with `.prePadded` mode and `lastAudioSample` carry-over for preemphasis continuity (exact NeMo parity required): [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:299`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L299).
4. **Encoder with cache** — encoder CoreML model receives mel features plus the three encoder caches (`cacheChannel`, `cacheTime`, `cacheLen`) created on first call via `EncoderCacheManager.createInitialCaches(...)`; outputs include `_out`-suffixed cache arrays for the next call: [`Sources/FluidAudio/ASR/Parakeet/Streaming/EncoderCacheManager.swift:23`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/EncoderCacheManager.swift#L23).
5. **Cache rotation** — `EncoderCacheManager.extractCachesFromOutput(...)` pulls the updated caches by `_out` suffix (or plain name) and binds them as the input caches for the next chunk: [`Sources/FluidAudio/ASR/Parakeet/Streaming/EncoderCacheManager.swift:58`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/EncoderCacheManager.swift#L58).
6. **RNN-T decode loop** — `RnntDecoder.decodeWithEOU(encoderOutput:timeOffset:skipFrames:validOutLen:)` walks each encoder time-step, slices `[1, 640, 1]`, runs decoder + joint, picks `token_id`, breaks on blank (`1026`), sets `eouDetected = true` on `1024`. Decoder LSTM state `h_in`/`c_in` carried across steps; `maxSymbolsPerStep = 2` caps emissions per encoder frame: [`Sources/FluidAudio/ASR/Parakeet/Streaming/RnntDecoder.swift:63`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/RnntDecoder.swift#L63).
7. **Cross-chunk dedup** — `SequenceMatcher.findSuffixPrefixMatch(...)` merges boundary tokens when the model emits overlapping prefixes across chunks: [`Sources/FluidAudio/ASR/Parakeet/TokenDeduplication/SequenceMatcher.swift:24`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/TokenDeduplication/SequenceMatcher.swift#L24).
8. **Detokenize + emit** — `Tokenizer.decode(ids:)` produces incremental text; the partial callback is invoked with the cumulative transcript so far: [`Sources/FluidAudio/ASR/Parakeet/Streaming/Tokenizer.swift:19`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/Tokenizer.swift#L19).
9. **Optional commit layer** — caller wires `PunctuationCommitLayer.processPartialText(_:)` to split into stable `committedText` and volatile `ghostText`, debouncing or committing on `.`/`!`/`?` punctuation: [`Sources/FluidAudio/ASR/Shared/PunctuationCommitLayer.swift:150`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Shared/PunctuationCommitLayer.swift#L150).
10. **Finish** — `finish()` flushes remaining buffer, returns final `String`; `reset()` keeps weights but clears caches/state: [`Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrManager.swift:43`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/StreamingAsrManager.swift#L43).

## Outputs
- Streaming: partial transcript `String` via `@Sendable (String) -> Void` callback.
- Final `String` from `finish()`.
- Optional `CommitLayerUpdate { committed, ghost, total, reason, timestamp }` if commit layer wired.

## Error Modes
- Protocol does not surface backpressure: unconsumed audio grows the internal buffer if `processBufferedAudio()` isn't called.
- `cleanup()` makes the engine unusable until `loadModels()` is called again.
- Concurrent calls into the actor are serialized but may queue if callbacks are slow.
- `setPartialTranscriptCallback` replaces (not appends to) the callback; callers re-entering into the manager from the callback risk deadlock.

## Prompts and Models Used

## Usage Metrics

## Changelogs
