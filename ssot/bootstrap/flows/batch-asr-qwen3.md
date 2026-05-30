---
id: batch-asr-qwen3
name: Batch ASR — Qwen3
trigger_type: user-action
trigger: Caller invokes `Qwen3AsrManager.transcribe(audioSamples:language:maxNewTokens:)` (or the `melSpectrogram:` overload) with a complete audio buffer.
end_state: A transcript `String` is returned from the audio-encoder + 28-layer stateful LLM decoder pipeline; the manager remains usable for the next call.
involves_features: [qwen3-asr-manager, qwen3-asr-models, qwen3-rope, whisper-mel-spectrogram]
linked_flows: [model-download-and-load]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Batch ASR — Qwen3

## TL;DR
Qwen3-ASR-0.6B two-model pipeline: a Whisper-style mel encoder produces audio features that are spliced into a chat-template embedding stream, then a 28-layer stateful LLM decoder is prefilled and auto-regressively sampled until an EOS token. Swift-side embedding lookup eliminates one CoreML call per token. Macros: macOS 15 / iOS 18 only.

## Trigger & Preconditions
`Qwen3AsrManager.transcribe(...)`. Models loaded via `Qwen3AsrModels.downloadAndLoad`. `Qwen3RoPE` cos/sin precomputed for `[0, maxCacheSeqLen)` at construction. Audio is `[Float]` mono 16 kHz; tooling internally runs `WhisperMelSpectrogram` to derive mel.

## Stages
1. **Encode audio (`encodeAudio`)** — runs `WhisperMelSpectrogram` over the input, slides a 100-frame mel window through `qwen3_asr_audio_encoder_v2`. Each window emits 13 output frames (`(100+7)/8`); last partial window emits `(remaining+7)/8`. Each frame becomes a `[1024]` Float vector appended to the flat audio-feature list: [`Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift:134`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L134).
2. **Build prompt (`buildPromptTokens`)** — assembles Qwen3 chat-template: `im_start system \n <task tokens> im_end \n im_start user \n audio_start <audio_token>×N audio_end im_end \n im_start assistant \n`. Optional language hint maps to pre-tokenized `taskTokens` (30 languages); `audio_token` (151676) placeholder count == `numAudioFrames`: [`Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift:248`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L248).
3. **Embed + merge (`embedAndMerge`)** — full prompt-token sequence embedded by reading `embeddingWeights.embeddings(for:)` (the preloaded `[151936, 1024]` FP16 matrix), then every position whose token ID equals `audio_token` is overwritten with the corresponding encoder feature vector: [`Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift:283`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L283).
4. **Prefill** — single batched call over the entire prompt with `[L, L]` causal mask, RoPE cos/sin from `Qwen3RoPE.computeRange`, and a fresh `MLState` from `decoderStateful.makeState()`. First generated token comes from prefill logits: [`Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift:305`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L305).
5. **Decode loop** — per step: read one embedding from `embeddingWeights.embedding(for:)`, copy into preallocated `decHiddenArray`, fill `decodeCosArray` / `decodeSinArray` via `Qwen3RoPE.fill(position:cosPtr:sinPtr:)`, build the `[1,1,1,endStep]` zero-mask, call `decoderStateful.prediction(from:using:state)`. Argmax via `vDSP_maxvi` over the 151,936-entry logits. Stops on EOS (`151645` or `151643`) or `effectiveMaxNew = min(maxNewTokens, maxCacheSeqLen - promptLength)`: [`Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift:427`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L427).
6. **Decode tokens** — `decodeTokens` slices the generated tokens after `asrTextTokenId` (151704, the marker the model emits before transcription content), joins vocabulary pieces, applies the HuggingFace GPT-2-style BPE byte-decode via `bpeUnicodeToByte` → UTF-8: [`Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift:484`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L484).

## Outputs
- Transcribed `String`.

## Error Modes
- `Qwen3AsrError.generationFailed("Audio too short to extract mel spectrogram")` for sub-hop audio.
- `Qwen3AsrError.generationFailed("Models not loaded")` if `transcribe` is called before load.
- Prompt length exceeding `maxCacheSeqLen` (512) throws (effective new tokens ≤ 0).
- Unknown language strings log a warning and fall back to automatic detection (no task tokens appended).
- First generated token already EOS → empty token list → empty string.
- `argmaxFromLogits` binds the data pointer to `Float` — FP16 logits future-compat hazard [REVIEW].

## Prompts and Models Used

## Usage Metrics

## Changelogs
