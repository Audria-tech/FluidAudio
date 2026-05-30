---
id: lseend-inference
name: LS-EEND Inference
repo: FluidAudio
status: active
linked_features: [lseend-diarizer, lseend-types, lseend-preprocessor]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# LS-EEND Inference

## TL;DR
`LSEENDModel` wraps a single CoreML T-block model exported with per-instance `LSEENDMetadata` JSON. `LSEENDInput` carries mel features + decoder mask + the seven recurrent-state `MLMultiArray`s required by the model. Inference returns a flat `[Float]` of speaker probabilities with warmup frames stripped via stride-aware `vDSP_mmov`.

## Motivation — why it exists

## Context

## What It Does
`LSEENDModel.init(modelURL:computeUnits:)` loads the model and decodes the `creatorDefinedKey.config` JSON into an `LSEENDMetadata`. `loadFromHuggingFace(variant:stepSize:cacheDirectory:computeUnits:progressHandler:)` resolves the variant-specific HF repo path via `variant.fileName(forStep:)`, downloads via `DownloadUtils.downloadSubdirectory`, and verifies the model is present after download. Default compute units are `.cpuOnly` (model docstring notes CPU is fastest for this topology). `predict(from:)` is wrapped in `autoreleasepool` and a per-instance `NSLock`; it runs the model, extracts seven output `MLMultiArray`s (probs + enc_kv + enc_scale + enc_conv_cache + cnn_window + dec_kv + dec_scale), updates the input state in place with the new arrays, then copies the probs tensor (skipping `warmupFrames` rows) into a flat `[outputFrames * outputSpeakers]` array using stride-aware `vDSP_mmov`.

`LSEENDInput: MLFeatureProvider` carries `state: LSEENDState`, pre-allocated `melFeatures` `[1, M, N]` and `decoderMask` `[T]` MLMultiArrays, and `warmupFrames: Int`. `featureNames` exposes the eight required inputs. `loadInputs(melFeatures:decoderMask:warmupFrames:)` is the per-chunk update: it memcpys both buffers and computes `warmupFrames` from the count of zero entries in `decoderMask` if not provided.

## Key Code
- [`LSEENDInference.swift:16`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDInference.swift#L16) — `LSEENDModel.init` config-from-metadata JSON decoding.
- [`LSEENDInference.swift:42`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDInference.swift#L42) — `loadFromHuggingFace` partial-download (just the one mlmodelc).
- [`LSEENDInference.swift:85`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDInference.swift#L85) — `predict(from:)` lock + autoreleasepool + state in-place update + warmup-strip via `vDSP_mmov`.
- [`LSEENDInference.swift:117`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDInference.swift#L117) — Probs innermost-stride assertion: throws if `strides.last != 1`.
- [`LSEENDInference.swift:174`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDInference.swift#L174) — `loadInputs` `warmupFrames` derivation from decoder mask zero-count.

## Edge Cases & Failure Modes
- Throws `LSEENDError.initializationFailed("No `config` found in model metadata")` if the model's `creatorDefinedKey.config` is missing or malformed JSON.
- Throws `LSEENDError.initializationFailed("HF download completed but mlmodelc missing...")` if the download succeeds but the expected file isn't present.
- Throws `LSEENDError.inferenceFailed("Failed to extract predictions...")` if any of the seven outputs is missing from the prediction result.
- Throws `LSEENDError.inferenceFailed("Probs innermost stride must be 1...")` if the CoreML model produces a non-contiguous probs tensor — guards against silently mis-copying via `vDSP_mmov`.
- `outputFrames * outputSpeakers == 0` (degenerate warmup-only chunk) returns `[]` without a vDSP call.
- `loadInputs` throws `LSEENDError.invalidInputSize` if the new buffer's count != the destination's count — strict shape check.

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/LS-EEND/LSEENDFeatureProvider.swift` — test helper for `LSEENDInput`.
- `Tests/FluidAudioTests/Diarizer/LS-EEND/LSEENDQueueTests.swift` — exercises the streaming chunk path that feeds into `predict`.

## Changelogs
