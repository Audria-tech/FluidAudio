---
id: pockettts-stream-synthesize
name: PocketTTS Streaming Synthesize
trigger_type: user-action
trigger: Caller invokes `PocketTtsManager.synthesizeStreaming(text:voice:temperature:deEss:)` (or batches via `synthesize`).
end_state: 24 kHz Float32 audio frames (1920 samples ≈ 80 ms each) are yielded on the async stream as flow-matching language-model AR steps produce them; on stream completion the manager is ready for the next call.
involves_features: [pocket-tts-manager, pocket-tts-pipeline, pocket-tts-tokenizer]
linked_flows: [model-download-and-load]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# PocketTTS Streaming Synthesize

## TL;DR
PocketTTS flow-matching language model. Per chunk: tokenize → embed → prefill KV cache → AR loop (one 80 ms / 1920-sample frame per step at 24 kHz) → flow decode → Mimi decode. The streaming API yields each `AudioFrame` as soon as its Mimi decode finishes; long text is auto-chunked to fit the 512-position KV cache.

## Trigger & Preconditions
`PocketTtsManager(...).initialize()` (lazy CoreML download + load via `PocketTtsModelStore.loadIfNeeded()`). Language is immutable after construction. Default precision `.fp16` (matches upstream); `.int8` swaps `flowlm_step` for `flowlm_stepv2`.

## Stages
1. **Resolve voice** — explicit `voice:` argument or `defaultVoice` from init. `PocketTtsVoiceData` is loaded from disk-cached `.bin` (or fetched lazily).
2. **Enter task-local context** — `PocketTtsSynthesizer.withModelStore(...)` makes the model store available to the struct's static methods.
3. **Chunk text** — sentence-based splitter (≤ 50 tokens each) to stay within the 512-position KV cache: [`Sources/FluidAudio/TTS/PocketTTS/Pipeline/PocketTtsSynthesizer.swift:13`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/Pipeline/PocketTtsSynthesizer.swift#L13).
4. **Per chunk: tokenize** — SentencePiece tokenizer (`SentencePieceProto` + `SentencePieceTokenizer`) produces token IDs.
5. **Embed** — token IDs → embedding rows from the FlowLM model.
6. **Prefill KV** — voice prefill (~125 tokens) primes the FlowLM KV cache; `PocketTtsSynthesizer+KVCache` holds decoder state across AR steps.
7. **AR generate** — per step the FlowLM emits flow features; `PocketTtsSynthesizer+Flow` runs the flow-matching transformer step.
8. **Mimi decode** — `PocketTtsSynthesizer+Mimi` invokes the Mimi neural codec decoder, producing one 1920-sample (80 ms) Float32 frame at 24 kHz.
9. **Yield frame** — `synthesizeStreaming` yields `AudioFrame(samples: [Float])` immediately so playback can begin before the utterance completes: [`Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift:178`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift#L178).
10. **Optional de-ess** — when `deEss: true` (default), `AudioPostProcessor.deEss(...)` is applied to each frame (biquad high-shelf): [`Sources/FluidAudio/TTS/Shared/AudioPostProcessor.swift:15`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Shared/AudioPostProcessor.swift#L15).
11. **Continue or stop** — loop until EOS or chunk exhausted; next chunk reuses the warm KV cache when using a `PocketTtsSession`.

## Outputs
- `AsyncThrowingStream<PocketTtsSynthesizer.AudioFrame, Error>` — each frame carries 1920 Float32 samples at 24 kHz.
- Batch `synthesize(...)` returns a 24 kHz WAV `Data` (RIFF/fmt/data) instead.

## Error Modes
- `PocketTTSError.modelNotFound("PocketTTS model not initialized")` before `initialize`.
- `PocketTTSError.modelNotFound` if `cloneVoice(...)` is called without the optional `mimi_encoder` model.
- `synthesizeToFile` deletes any existing file at `outputURL` first.
- Language is immutable — switching languages requires a new manager.
- Sessions: concurrent `enqueue` on the same manager share Mimi decoder state (interference possible).
- KV cache hard-capped at 512 positions; the chunker splits to fit.
- Statics throw `PocketTTSError.processingFailed` if invoked outside the `withModelStore(...)` scope.
- [REVIEW] Stream cancellation mid-utterance: Mimi state behavior not validated in scope.

## Prompts and Models Used

## Usage Metrics

## Changelogs
