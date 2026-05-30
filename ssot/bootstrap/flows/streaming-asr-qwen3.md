---
id: streaming-asr-qwen3
name: Streaming ASR — Qwen3 (Re-transcribe on Growing Buffer)
trigger_type: event
trigger: Application pushes audio chunks via `Qwen3StreamingManager.addAudio(_:)` and reads incremental transcripts every `chunkSeconds`.
end_state: Incremental `Qwen3StreamingResult` values are returned; `finish()` returns a final result and resets the buffer.
involves_features: [qwen3-asr-streaming-manager, qwen3-asr-manager]
linked_flows: [batch-asr-qwen3, model-download-and-load]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Streaming ASR — Qwen3 (Re-transcribe on Growing Buffer)

## TL;DR
`Qwen3StreamingManager` is a thin actor over `Qwen3AsrManager`. Despite the "streaming" name, it does **not** maintain a KV-cache continuation between ticks — each emission re-runs the full offline `transcribe(...)` on the entire (trimmed) accumulated buffer. Words at the buffer head may disappear when the buffer is trimmed past `maxAudioSeconds`.

## Trigger & Preconditions
`Qwen3StreamingManager(asrManager:config:)`. Config carries `minAudioSeconds` (don't transcribe before this), `chunkSeconds` (re-transcribe cadence), `maxAudioSeconds` (rolling buffer cap), and an optional language hint. `asrManager.loadModels()` must already have succeeded.

## Stages
1. **Append audio** — `addAudio(_:)` appends samples to the internal `[Float]` buffer, increments `samplesSinceLastTranscribe`: [`Sources/FluidAudio/ASR/Qwen3/Qwen3StreamingManager.swift:105`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3StreamingManager.swift#L105).
2. **Dual gate** — proceeds only when `hasMinAudio && chunkReady` (audio ≥ `minAudioSeconds` AND new audio since last tick ≥ `chunkSeconds`).
3. **Trim to max** — when total buffered audio exceeds `maxAudioSeconds`, the oldest samples are dropped; there is no continuation cache, so dropped words may simply vanish from the next emission.
4. **Re-transcribe entire buffer** — calls `asrManager.transcribe(audioSamples:language:maxNewTokens:512)` on the full retained buffer — this runs the complete `batch-asr-qwen3.md` flow (encode → prompt → embed → prefill → decode → detokenize) end-to-end every tick.
5. **Reset counter + emit** — resets `samplesSinceLastTranscribe`, returns `Qwen3StreamingResult(transcript, audioDuration, isFinal: false)`.
6. **Finish** — `finish()` runs a last transcription if buffered audio is unprocessed since the last tick, returns `isFinal: true`, and resets all state: [`Sources/FluidAudio/ASR/Qwen3/Qwen3StreamingManager.swift:152`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3StreamingManager.swift#L152).

## Outputs
- `Qwen3StreamingResult { transcript: String, audioDuration: TimeInterval, isFinal: Bool }`.

## Error Modes
- Wall-time grows roughly linearly with `audioDuration` until `maxAudioSeconds` clamps it (each tick re-runs prompt + decode on the full buffer).
- Buffer trim discards audio permanently — words near the head can disappear from the next emission.
- `maxNewTokens: 512` hardcoded — long utterances may truncate.
- If the caller never feeds enough new samples to satisfy `chunkReady`, no emission fires until `finish()`.
- [REVIEW noted in feature.md] The "sliding window" terminology is misleading — the implementation re-transcribes a single growing buffer rather than maintaining a continuation window.

## Prompts and Models Used

## Usage Metrics

## Changelogs
