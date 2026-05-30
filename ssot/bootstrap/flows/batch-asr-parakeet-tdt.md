---
id: batch-asr-parakeet-tdt
name: Batch ASR — Parakeet TDT
trigger_type: user-action
trigger: Caller invokes `SlidingWindowAsrManager.transcribe(...)` (or the `AsrManager.transcribe(_:)` short-path) with a complete `[Float]` 16 kHz buffer or an `AVAudioPCMBuffer`.
end_state: A finalized `ASRResult` is returned with the transcript text, token IDs, per-token timings, confidence scores; the decoder LSTM state is fully drained, the manager remains usable for the next call.
involves_features: [sliding-window-asr-manager, sliding-window-asr-session, tdt-decoder, parakeet-tokenizer, audio-converter, audio-mel-spectrogram, encoder-cache-manager]
linked_flows: [model-download-and-load, model-warmup, custom-vocab-rescoring, token-dedup-cross-chunk]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Batch ASR — Parakeet TDT

## TL;DR
Offline-style transcription via the Parakeet TDT 0.6B family. The sliding-window manager slices the input audio into `[leftContext, chunk, rightContext]` windows, runs each through the preprocessor → encoder → TDT joint pipeline with a per-call `TdtDecoderState`, accumulates tokens with cross-chunk deduplication, and returns the final transcript.

## Trigger & Preconditions
`SlidingWindowAsrManager.transcribe(_:)` or its streaming-but-bounded variant. Models must already be loaded via `SlidingWindowAsrSession.createStream(...)` or `AsrManager.loadModels(...)` — first call paths through `model-download-and-load.md` then `model-warmup.md`. Audio is any format `AudioConverter` can consume (resampled to mono 16 kHz Float32 on entry).

## Stages
1. **Audio normalize** — `AudioConverter.resampleBuffer(...)` / `resample(_:from:)` produces a `[Float]` at 16 kHz mono Float32 (fast path skips conversion when input already matches): [`Sources/FluidAudio/Shared/AudioConverter.swift:77`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L77).
2. **Window slicing** — `SlidingWindowAsrManager.appendSamplesAndProcess` advances by `chunk` samples while preserving `leftContext` for context. Buffer trimming uses absolute indexing (`bufferStartIndex`) so partial advances stay aligned: [`Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift:319`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift#L319).
3. **Per-window transcribe** — `processWindow` calls `AsrManager.transcribeChunk(...)` with the current `TdtDecoderState`, `previousTokens`, and an `isLastChunk` flag: [`Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift:399`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift#L399).
4. **Encoder pipeline** — `AsrManager+Pipeline.executeMLInferenceWithTimings(...)` runs `Preprocessor` → `Encoder` CoreML models. Encoder cache state is created via `EncoderCacheManager.createInitialCaches(...)` on first call and rotated on subsequent calls: [`Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/AsrManager+Pipeline.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/AsrManager%2BPipeline.swift#L5).
5. **TDT decode loop** — `TdtDecoderV3` walks the encoder frames; per step it builds the joint input, picks `(token, durationIndex)` via `TdtJointDecision`, advances by `durationBins[durationIndex]`, and only updates the LSTM `decoderModel` on non-blank tokens (`blankId = 8192`). Inner symbol loop capped at `maxSymbolsPerStep = 10`; per-chunk emissions capped at `maxTokensPerChunk = 150`; `consecutiveBlankLimit = 5` triggers early stop on tail-of-silence: [`Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/Decoder/TdtDecoderV3.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/Decoder/TdtDecoderV3.swift#L1), [`Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/Decoder/TdtConfig.swift:1`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/Decoder/TdtConfig.swift#L1).
6. **Token alignment to global timeline** — `applyGlobalFrameOffset` adds `windowStartSample / samplesPerEncoderFrame` to every chunk-local timestamp so the global track is correct: [`Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift:623`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift#L623).
7. **Cross-chunk dedup** — `SequenceMatcher.findSuffixPrefixMatch(...)` (from `token-deduplication`) removes overlapped tokens between window N and N+1: [`Sources/FluidAudio/ASR/Parakeet/TokenDeduplication/SequenceMatcher.swift:24`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/TokenDeduplication/SequenceMatcher.swift#L24).
8. **Flush remaining** — `flushRemaining` drains the trailing partial chunk at end-of-stream without right context, with `isLastChunk: true` so TDT decoder produces final blanks: [`Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift:358`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift#L358).
9. **Detokenize** — `AsrManager.convertTokensToText(...)` looks up each ID in the Parakeet SentencePiece vocabulary; word boundaries (`\u{2581}`) become spaces; trailing whitespace is trimmed: [`Sources/FluidAudio/ASR/Parakeet/Streaming/Tokenizer.swift:19`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/Tokenizer.swift#L19).
10. **Assemble result** — return `ASRResult` with `text`, `tokens`, `tokenTimings`, `confidence`, `language`.

## Outputs
- `ASRResult` carrying transcript `String`, token IDs `[Int]`, per-token timings `[TokenTiming]`, overall confidence, and language code.
- For long-form `ChunkProcessor` mode: ~14.96 s windows, 2 s overlap, 80 ms left mel-context. Concurrency governed by `ASRConfig.parallelChunkConcurrency`.

## Error Modes
- `ASRError.notInitialized` — `loadModels()` never called.
- `ASRError.modelProcessingFailed` — CoreML prediction error; `attemptErrorRecovery` rebuilds `TdtDecoderState` with the correct layer count from `AsrManager.decoderLayerCount`.
- TDT shape errors when the 1-layer (tdtCtc110m) vs 2-layer LSTM split is misconfigured.
- Audio shorter than `winLength` produces a single zero-frame mel → typically empty transcript.

## Prompts and Models Used

## Usage Metrics

## Changelogs
