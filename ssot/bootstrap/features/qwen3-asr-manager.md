---
id: qwen3-asr-manager
name: Qwen3-ASR Manager
repo: FluidAudio
status: active
linked_features:
  - qwen3-asr-config
  - qwen3-asr-models
  - qwen3-asr-streaming-manager
  - qwen3-rope
  - whisper-mel-spectrogram
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Qwen3-ASR Manager

## TL;DR
`Qwen3AsrManager` is the actor that drives the Qwen3-ASR-0.6B 2-model pipeline: a Whisper-style mel encoder feeds into a 28-layer LLM decoder with a stateful KV cache, with Swift-side embedding lookup eliminating one CoreML call per token. Audio is processed in 100-frame mel windows through `audio_encoder`, prompt tokens are built via a chat-template, and the decoder is prefilled then auto-regressively sampled until an EOS token (`151645` or `151643`) is emitted.

## Motivation — why it exists

## Context

## What It Does
The pipeline has five stages, all driven from the public `transcribe(audioSamples:language:maxNewTokens:)` entry point. Stage 1 (`encodeAudio`) runs the `WhisperMelSpectrogram` over the input audio, then slides a 100-frame window through the audio encoder (`qwen3_asr_audio_encoder_v2`). Each window produces 13 output frames (`(100+7)/8`); the last partial window produces `(remaining+7)/8` frames. Each frame is unpacked into a `[1024]` Float vector and appended to a flat `[[Float]]` audio-feature list.

Stage 2 (`buildPromptTokens`) constructs the Qwen3 chat-template token sequence: `im_start system \n <task tokens> im_end \n im_start user \n audio_start <audio_token>×N audio_end im_end \n im_start assistant \n`. The optional language hint is realized by pre-tokenized "Transcribe the audio to {Language} text." strings stored in a static `taskTokens` dictionary keyed by `Qwen3AsrConfig.Language`. The `audio_token` (151676) placeholders are exactly `numAudioFrames` long so they line up 1:1 with audio features.

Stage 3 (`embedAndMerge`) does the Swift-side embedding lookup. The full prompt-token sequence is embedded by `models.embeddingWeights.embeddings(for:)` (which reads the preloaded `[151936, 1024]` weight matrix in FP16) and then every position whose token ID equals `audio_token` is *overwritten* with the corresponding audio feature vector. This is the trick that avoids the embedding CoreML model entirely.

Stage 4 (`generate`) runs the stateful decoder. It first does a single "prefill" call over the entire prompt (with a `[L, L]` causal mask, RoPE cos/sin computed by `Qwen3RoPE.computeRange`, and an `MLState` from `decoderStateful.makeState()`). The first generated token comes from the prefill logits. Then it enters the decode loop: per step it reads one embedding from `embeddingWeights.embedding(for:)`, copies it into the preallocated `decHiddenArray`, fills `decodeCosArray`/`decodeSinArray` via `Qwen3RoPE.fill(position:cosPtr:sinPtr:)`, builds the `[1, 1, 1, endStep]` zero-mask, and calls `decoderStateful.prediction(from:using:state)`. `argmaxFromLogits` uses `vDSP_maxvi` over the 151,936-entry logits row. Generation stops on any EOS token or when `effectiveMaxNew` (capped to `maxCacheSeqLen - promptLength`) is reached.

Stage 5 (`decodeTokens`) slices the generated tokens after `asrTextTokenId` (151704, the marker the model emits before transcription content), joins the vocabulary pieces, and runs the standard HuggingFace GPT-2-style BPE byte-decode: every Unicode scalar is mapped back through the `bpeUnicodeToByte` table, the bytes are interpreted as UTF-8, and the result is trimmed.

## Key Code
- [`Qwen3AsrManager.swift:21`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L21) — `@available(macOS 15, iOS 18, *)` actor declaration; stores `Qwen3AsrModels?`, `Qwen3RoPE`, and `WhisperMelSpectrogram`.
- [`Qwen3AsrManager.swift:71`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L71) — `transcribe(melSpectrogram:language:maxNewTokens:)` orchestrates encode → prompt → embed → generate → decode and logs per-stage timings.
- [`Qwen3AsrManager.swift:134`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L134) — `encodeAudio` slides 100-frame mel windows through the audio encoder.
- [`Qwen3AsrManager.swift:215`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L215) — `taskTokens` static map of pre-tokenized language hints (30 languages).
- [`Qwen3AsrManager.swift:248`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L248) — `buildPromptTokens` assembles the chat-template token stream with N `audioTokenId` placeholders.
- [`Qwen3AsrManager.swift:283`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L283) — `embedAndMerge` overwrites audio-token positions with encoder features.
- [`Qwen3AsrManager.swift:305`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L305) — `generate` prefill + decode loop with preallocated decode buffers and the EOS check on the first token.
- [`Qwen3AsrManager.swift:427`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L427) — `runStatefulDecoder` packages inputs into `MLDictionaryFeatureProvider` and pulls `logits` from output.
- [`Qwen3AsrManager.swift:453`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L453) — `argmaxFromLogits` via `vDSP_maxvi` over the full vocab.
- [`Qwen3AsrManager.swift:463`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L463) — `bpeUnicodeToByte` GPT-2 reversible byte ↔ Unicode-printable mapping.
- [`Qwen3AsrManager.swift:484`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L484) — `decodeTokens` slices after `asrTextTokenId` and applies BPE byte-decode → UTF-8.
- [`Qwen3AsrManager.swift:512`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrManager.swift#L512) — `createPrefillMask` causal `[1,1,L,L]` with `-1e9` upper triangle.

## Edge Cases & Failure Modes
- Empty mel spectrogram (audio shorter than one hop) throws `Qwen3AsrError.generationFailed("Audio too short to extract mel spectrogram")`.
- Models not loaded → `Qwen3AsrError.generationFailed("Models not loaded")` from `transcribe(melSpectrogram:...)`.
- Prompt length exceeding `Qwen3AsrConfig.maxCacheSeqLen` (512) is caught: `effectiveMaxNew` is computed as `min(maxNewTokens, maxCacheSeqLen - promptLength)` and throws if `<= 0`.
- Language strings that don't match an ISO code or English name log a warning and fall back to automatic detection (no task tokens are appended).
- The first generated token is computed from the prefill logits — if it is already EOS, the function returns an empty token list (and `decodeTokens` returns an empty string).
- Unknown vocab IDs silently produce missing pieces — the BPE byte map drops any scalar not in the table, so corrupted output yields shorter strings rather than crashes.
- Decode-loop break condition uses `generatedTokens.last` — `break` exits early on any unexpected nil.
- `argmaxFromLogits` binds the data pointer to `Float`; if a future model exports FP16 logits this will need updating. [REVIEW: `Qwen3AsrError` is referenced but the enum definition is not shown in this file — confirmed defined elsewhere in the module (likely `Qwen3AsrModels.swift`).]

## Performance / Concurrency Notes
- Per-token decode is dominated by the stateful decoder CoreML call. Reusing preallocated `decHiddenArray`, `decodeCosArray`, and `decodeSinArray` and writing via raw pointers eliminates per-step allocation.
- Swift-side embedding lookup avoids one MLModel prediction per token — significant for the 28-layer 1024-dim decoder.
- RoPE cos/sin are precomputed for positions `[0, maxCacheSeqLen)` in `Qwen3RoPE.init()`; the hot loop only memcpys into the input arrays.
- Prefill is a single batched call so the entire prompt latency is one model invocation — typically the largest chunk of wall time for short clips.
- `actor` isolation serializes concurrent `transcribe(...)` calls; for true streaming use `Qwen3StreamingManager` which wraps this manager.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Qwen3/Qwen3AsrConfigTests.swift` — config invariants.
- `Tests/FluidAudioTests/ASR/Qwen3/Qwen3RoPETests.swift` — RoPE cos/sin parity (covers the position embeddings used by `generate`).

## Changelogs
