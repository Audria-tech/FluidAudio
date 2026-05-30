---
id: sortformer-model-inference
name: Sortformer Model Inference
repo: FluidAudio
status: active
linked_features: [sortformer-diarizer, sortformer-types]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Sortformer Model Inference

## TL;DR
`SortformerModels` owns the CoreML "main model" (combined preprocessor + pre-encoder + head) plus six pre-allocated ANE-aligned `MLMultiArray` input buffers, exposes loaders for local-file and HuggingFace paths, and runs `runMainModel(...)` returning predictions + chunk embeddings + chunk length.

## Motivation — why it exists

## Context

## What It Does
The struct stores `mainModel: MLModel`, `compilationDuration: TimeInterval`, an `ANEMemoryOptimizer`, and six pre-allocated arrays sized from `SortformerConfig`: `chunkArray` `[1, chunkMelFrames, melFeatures]`, `fifoArray` `[1, fifoLen, preEncoderDims]`, `spkcacheArray` `[1, spkcacheLen, preEncoderDims]`, plus three Int32 length scalars. `load(config:mainModelPath:configuration:)` compiles the `.mlpackage` via `MLModel.compileModel`, loads with `.all` compute units, and returns the struct. `loadFromHuggingFace(config:cacheDirectory:computeUnits:progressHandler:)` resolves the bundle name from `ModelNames.Sortformer.bundle(for:)`, runs `DownloadUtils.loadModels(.sortformer, ...)`, and instantiates. `runMainModel(chunk:chunkLength:spkcache:spkcacheLength:fifo:fifoLength:config:)` calls `memoryOptimizer.optimizedCopy(..., pad: true)` to load the three feature buffers, writes the length scalars, builds an `MLDictionaryFeatureProvider`, runs `mainModel.prediction`, and extracts three outputs by checking both the `_out`-suffixed and plain names: `speaker_preds` (Float32), `chunk_pre_encoder_lengths` (Int32), `chunk_pre_encoder_embs` (Float32 preferred, Float16 fallback on arm64+macOS15+/iOS18+). The `_out` suffix is required by the macOS 26+ BNNS compiler which disallows input/output name collisions.

## Key Code
- [`SortformerModelInference.swift:13`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerModelInference.swift#L13) — `SortformerModels` struct + six pre-allocated buffers.
- [`SortformerModelInference.swift:63`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerModelInference.swift#L63) — `load(config:mainModelPath:configuration:)` mlpackage→mlmodelc compile step.
- [`SortformerModelInference.swift:186`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerModelInference.swift#L186) — `runMainModel(...)` with the dual-name (`_out` vs plain) output extraction and Float16/Float32 fallback for chunk embeddings.
- [`SortformerModelInference.swift:237`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Sortformer/SortformerModelInference.swift#L237) — Inline comment documenting the macOS 26+ BNNS naming constraint.

## Edge Cases & Failure Modes
- Throws `SortformerError.inferenceFailed("Missing speaker_preds...")` if neither output name produces a Float32 scalar array.
- Throws `SortformerError.inferenceFailed("Missing chunk_pre_encoder_lengths...")` similarly.
- For chunk embeddings, on non-arm64 platforms the Float16 fallback is `#else throw` — meaning Intel macs cannot load a model that produces Float16 chunk embeddings.
- The buffer allocation in `init` can throw if `ANEMemoryOptimizer.createAlignedArray` fails (e.g., under memory pressure), so the `init` is marked `throws`.
- `MLPredictionOptions` is not passed (the `runMainModel` doesn't construct one) — uses CoreML defaults. The `_state` lengths are passed as Int32 NSNumbers; the model contract assumes int32.

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/Sortformer/SortformerTests.swift` exercises this via `SortformerDiarizer`.
- `Tests/FluidAudioTests/Diarizer/Sortformer/SortformerTimelineTests.swift` validates the predictions/embeddings split.

## Changelogs
