---
id: cohere-asr-pipeline
name: Cohere Transcribe ASR Pipeline
repo: FluidAudio
status: active
linked_features:
  - cohere-asr-config
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Cohere Transcribe ASR Pipeline

## TL;DR
`CoherePipeline` is the self-contained Swift implementation of the Cohere Transcribe CoreML ASR model: a 128-mel FilterbankFeatures spectrogram → Conformer encoder → cache-external Transformer decoder loop with repetition-penalty and no-repeat-n-gram sampling, and SentencePiece byte-fallback detokenization. It supports loading the encoder and decoder from different directories so callers can mix precisions (e.g. INT8 encoder + FP16 decoder), and ships two decoder variants (`.v2` static-shape ANE-friendly, `.v1` dynamic-shape CPU/GPU).

## Motivation — why it exists

## Context

## What It Does
The file is structured in three layers. The first is `CohereMelSpectrogram` — a from-scratch FilterbankFeatures-compatible mel extractor that matches the model's training pipeline bit-for-bit: symmetric Hann window zero-padded to the next power of two, optional 0.97 preemphasis, vDSP FFT, magnitude-power scaling (default 2.0), Slaney mel filterbank (HTK-style mel scale with the Slaney 1 kHz break and `log(6.4)/27` log-step), natural log with `2^-24` guard, and per-feature CMVN with `ddof=1` over valid frames only. A static `padOrTruncate(mel:validFrames:fixedFrames:)` reshapes the result to the encoder's fixed `[1, 128, 3500]` input.

The second layer is the `CoherePipeline` actor itself. `loadModels(encoderDir:decoderDir:vocabDir:decoderVariant:computeUnits:)` loads the two `mlmodelc` bundles independently (so they can come from different cache dirs) and verifies the decoder is cache-external by checking that `k_cache_0` is an input. The vocabulary is loaded from `vocab.json` as `[Int: String]`. `transcribe(audio:models:language:maxNewTokens:repetitionPenalty:noRepeatNgram:)` runs the mel extractor, calls the encoder once for the full mel, computes `encoderValidFrames = ceil(featureLength * 438 / 3500)`, then enters the decode loop.

The decode loop allocates one `k_cache_<i>` / `v_cache_<i>` MLMultiArray per decoder layer (probing the dtype from the model description so FP16 vs FP32 just works), builds the cross-attention mask (`buildCrossAttentionMask` writes `-1e4` for padded encoder positions), and steps the decoder one token at a time. The self-attention mask is variant-specific: the `.v2` static mask is `[1,1,1,maxSeqLen]` with positions `[0..step]` zero and the rest `-1e4`; the `.v1` dynamic mask is `[1,1,1,step+1]` all zeros. At each step the loop applies repetition penalty (divide-or-multiply by `penalty` depending on sign — the standard HF heuristic) and no-repeat-n-gram filtering against the history, then takes argmax via `vDSP_maxvi`. Prompt tokens (10 special tokens from `CohereAsrConfig.Language.promptSequence`) are force-fed for the first `prompt.count` steps; output collection begins after that and stops on `eosToken`. Cache updates are pulled from `k_cache_<i>_out` / `v_cache_<i>_out` each step.

The third layer is `convertTokensToText`, a SentencePiece-aware detokenizer that drops special tokens (≤4, EOS, and any `<|...|>`), reassembles `<0xXX>` byte-fallback pieces into UTF-8 bytes, replaces the `\u{2581}` word boundary marker with a space, and trims edges.

## Key Code
- [`CoherePipeline.swift:20`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L20) — `CohereAsrError` enum.
- [`CoherePipeline.swift:41`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L41) — `CohereMelSpectrogram` config + Slaney mel filter setup.
- [`CoherePipeline.swift:127`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L127) — `compute(audio:)` runs the FilterbankFeatures pipeline end-to-end.
- [`CoherePipeline.swift:250`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L250) — `padOrTruncate` reshapes mel to the encoder's fixed 3500-frame input.
- [`CoherePipeline.swift:329`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L329) — `CoherePipeline` actor + `DecoderVariant` enum (`.v2` static, `.v1` dynamic).
- [`CoherePipeline.swift:387`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L387) — `loadModels` mixed-precision loader with `k_cache_0` sanity check.
- [`CoherePipeline.swift:452`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L452) — `transcribe(...)` orchestrates mel → encoder → decode → detokenize.
- [`CoherePipeline.swift:534`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L534) — `encoderValidFrames` = `ceil(featureLength * 438 / 3500)`.
- [`CoherePipeline.swift:544`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L544) — `decodeCacheExternal` step loop with cache rotation.
- [`CoherePipeline.swift:681`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L681) — `buildSelfAttentionMask` static vs dynamic variants.
- [`CoherePipeline.swift:715`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L715) — `buildCrossAttentionMask` writes `-1e4` for padded encoder positions.
- [`CoherePipeline.swift:743`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L743) — `float16Bits` vImage-based FP32→FP16 conversion for mask writes.
- [`CoherePipeline.swift:760`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L760) — `copyLogitsFloat32` (vImage Planar16F → Planar Float path).
- [`CoherePipeline.swift:788`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L788) — `applyRepetitionPenalty` HF-style sign-aware scaling.
- [`CoherePipeline.swift:801`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L801) — `applyNoRepeatNgram` blocks tokens that would complete a repeated n-gram.
- [`CoherePipeline.swift:856`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Cohere/CoherePipeline.swift#L856) — `convertTokensToText` SentencePiece byte-fallback detokenizer.

## Edge Cases & Failure Modes
- Audio shorter than the mel hop produces `featureLength == 0` → throws `CohereAsrError.invalidInput("Audio too short...")`.
- The decoder requires `k_cache_0` as an input; non-cache-external mlmodelc bundles fail fast with `CohereAsrError.decodingFailed`.
- `vocab.json` must be `[String: String]` keyed by stringified int; arrays or numeric keys fail with `modelNotFound("Invalid vocab.json format")`.
- Repetition-penalty divide-by-zero is guarded by the `penalty != 1.0` early exit; sign-aware scaling means very negative logits get *more* negative (correct).
- No-repeat-n-gram with `n == 1` blocks every previously-seen token; the code handles this branch separately.
- The prompt-feeding history records the consumed prompt token rather than the discarded model prediction — this comment in the code is critical: otherwise `applyNoRepeatNgram` would suppress legitimate first output tokens.
- FP16 mask writes go via vImage conversion of `-1e4` — extremely negative values in FP16 still round to `-1e4` (within FP16's representable range; safe).
- Argmax via `vDSP_maxvi` returns the first max in case of ties (vDSP semantic) — deterministic.

## Performance / Concurrency Notes
- `CoherePipeline` is an actor; concurrent calls serialize on the actor's executor.
- Each decode step is a single CoreML prediction; cache arrays are reassigned (not memcpy'd) by reading the model's `k/v_cache_*_out` outputs and re-binding them next step — zero-copy when CoreML returns the underlying buffer.
- Mel computation is single-threaded but uses vDSP/Accelerate inner loops.
- For long audio (35 s max input — controlled by `CohereAsrConfig.maxAudioSeconds`), the encoder runs once and the decoder dominates wall time (~108 steps max by default).
- Mixed-precision loading allows callers to trade quality vs speed without touching the pipeline code.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Cohere/CohereAsrConfigTests.swift` — config invariants.
- `Tests/FluidAudioTests/ASR/Cohere/CoherePipelineMaskTests.swift` — self-attention and cross-attention mask construction across both decoder variants.

## Changelogs
