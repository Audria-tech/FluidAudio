---
id: batch-asr-cohere
name: Batch ASR — Cohere Transcribe
trigger_type: user-action
trigger: Caller invokes `CoherePipeline.transcribe(audio:models:language:maxNewTokens:repetitionPenalty:noRepeatNgram:)` with a complete audio buffer (≤ 35 s).
end_state: A transcript `String` is returned from the Conformer encoder + cache-external Transformer decoder with repetition-penalty / no-repeat-n-gram sampling; pipeline state is empty for the next call.
involves_features: [cohere-asr-pipeline, cohere-asr-config, audio-mel-spectrogram]
linked_flows: [model-download-and-load]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Batch ASR — Cohere Transcribe

## TL;DR
Cohere Transcribe CoreML pipeline: 128-mel FilterbankFeatures spectrogram → Conformer encoder (single forward pass over a fixed `[1, 128, 3500]` mel) → cache-external Transformer decoder loop with prompt-forced first tokens, repetition penalty, and no-repeat-n-gram filtering → SentencePiece byte-fallback detokenization.

## Trigger & Preconditions
`CoherePipeline.transcribe(...)`. Models loaded via `CoherePipeline.loadModels(encoderDir:decoderDir:vocabDir:decoderVariant:computeUnits:)` — encoder and decoder may come from different cache directories (e.g. INT8 encoder + FP16 decoder). Vocabulary loaded from `vocab.json` as `[Int: String]`. Audio ≤ 35 s (`CohereAsrConfig.maxAudioSeconds`).

## Stages
1. **Mel extraction** — `CohereMelSpectrogram.compute(audio:)` runs the bit-for-bit FilterbankFeatures pipeline: zero-pad-to-nextpow2 Hann, optional 0.97 preemphasis, vDSP FFT, magnitude-power scaling, Slaney mel (HTK scale with Slaney 1 kHz break), natural log with `2^-24` guard, per-feature CMVN with `ddof=1` over valid frames only: [`Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift:127`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L127).
2. **Shape to encoder input** — `padOrTruncate(mel:validFrames:fixedFrames:)` produces `[1, 128, 3500]` (the encoder's fixed input): [`Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift:250`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L250).
3. **Encoder forward** — single CoreML call over the full mel; `encoderValidFrames = ceil(featureLength * 438 / 3500)` (encoder downsamples 8×): [`Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift:534`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L534).
4. **Allocate decoder caches** — one `k_cache_<i>` / `v_cache_<i>` MLMultiArray per decoder layer; dtype probed from model description (so FP16 vs FP32 transparently works).
5. **Build cross-attention mask** — `buildCrossAttentionMask` writes `-1e4` for padded encoder positions (FP16 conversion via vImage when needed): [`Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift:715`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L715).
6. **Force-feed prompt** — first 10 steps consume `CohereAsrConfig.Language.promptSequence` tokens; the history records the consumed prompt token (not the discarded model prediction) so `applyNoRepeatNgram` doesn't suppress legitimate first outputs.
7. **Decode loop step** — `decodeCacheExternal` for each step builds the variant-specific self-attention mask (`.v2` static `[1,1,1,maxSeqLen]`, `.v1` dynamic `[1,1,1,step+1]`), runs the decoder, applies `applyRepetitionPenalty` (HF sign-aware scaling, skipped when `penalty == 1.0`) and `applyNoRepeatNgram` (special-cased for `n == 1`), takes argmax via `vDSP_maxvi`, reads `k_cache_<i>_out`/`v_cache_<i>_out` and re-binds for the next step: [`Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift:544`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L544).
8. **Stop condition** — loop terminates on `eosToken` or after `maxNewTokens` (≈ 108 default).
9. **Detokenize** — `convertTokensToText` drops special tokens (≤4, EOS, any `<|...|>`), reassembles `<0xXX>` byte-fallback pieces into UTF-8 bytes, replaces `\u{2581}` with space, trims: [`Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift:856`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L856).

## Outputs
- Transcribed `String`.

## Error Modes
- `CohereAsrError.invalidInput("Audio too short...")` for sub-hop audio (`featureLength == 0`).
- `CohereAsrError.decodingFailed` if the decoder is missing the `k_cache_0` input (non-cache-external bundle).
- `CohereAsrError.modelNotFound("Invalid vocab.json format")` if `vocab.json` isn't `[String: String]` keyed by stringified ints.
- Repetition-penalty divide-by-zero guarded by `penalty != 1.0` early exit.
- FP16 mask values rounded to `-1e4` (within FP16 range — safe).

## Prompts and Models Used

## Usage Metrics

## Changelogs
