---
id: model-warmup
name: Model Warmup
trigger_type: startup
trigger: A manager finishes loading CoreML models and runs a zero-valued prediction so the ANE/GPU allocates compute buffers before the first real call.
end_state: Each warmed model has executed at least one prediction; ANE buffer pools are seeded; user-visible latency on the first real inference excludes first-dispatch cost.
involves_features: [model-warmup, ane-memory-optimizer, mlmodel-prediction]
linked_flows: [model-download-and-load]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Model Warmup

## TL;DR
After `loadModels`, each pipeline runs one or more zero-input predictions through the freshly compiled `MLModel`s so the Neural Engine allocates its internal buffers before the user-visible first call. Generic primer `warmup(...)` covers the standard case; offline diarizer uses the specialized `warmupEmbeddingModel(...)` with three-tier shape fallback.

## Trigger & Preconditions
Triggered after `DownloadUtils.loadModels` returns — `OfflineDiarizerManager.prewarmSegmentationModel` and `prewarmEmbeddingStack` run during `prepareModels`, ASR/TTS variants run a per-variant warmup after compile, Magpie/Kokoro/PocketTTS managers also run their own throwaway synthesis. Models must already be compiled and held by the caller.

## Stages
1. **Allocate zero input** — caller (or `warmup`) builds an `MLMultiArray` of the requested shape, dtype `.float32`, and zero-fills it via the vDSP-accelerated `resetToZeros()` extension: [`Sources/FluidAudio/Shared/ModelWarmup.swift:147`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ModelWarmup.swift#L147).
2. **Wrap in feature provider** — generic path uses `MLDictionaryFeatureProvider` keyed by the model's first input name; ANE-aligned allocations route through `ANEMemoryUtils.createAlignedArray(...)` so the buffer matches the model's stride layout: [`Sources/FluidAudio/Shared/ModelWarmup.swift:17`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ModelWarmup.swift#L17), [`Sources/FluidAudio/Shared/ANEMemoryUtils.swift:23`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ANEMemoryUtils.swift#L23).
3. **Run prediction(s)** — `model.prediction(from:)` is invoked `iterations` times; the result is discarded. Offline pipelines prefer `compatPrediction(from:options:)` (the async wrapper) to avoid blocking on a contended ANE: [`Sources/FluidAudio/Shared/MLModel+Prediction.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/MLModel%2BPrediction.swift#L5).
4. **Embedding-model fallback chain** — `warmupEmbeddingModel(_:weightFrames:audioSamples:)` probes the model description for `fbank_features` + `weights` with positive constraint shapes; on failure it tries the combined `audio_and_weights` input, then the legacy dual `audio` + `weights` pair: [`Sources/FluidAudio/Shared/ModelWarmup.swift:47`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ModelWarmup.swift#L47).
5. **Return timing** — both helpers return elapsed seconds for diagnostic logging. Callers (`OfflineDiarizerManager`, `KokoroTtsManager`, `MagpieTtsManager`) log this in their initialize path.

## Outputs
- Elapsed warmup duration in seconds.
- Side effect: ANE/GPU buffer pools and CoreML internal caches seeded for subsequent real calls.

## Error Modes
- `precondition` trap on `iterations <= 0` or empty `inputShape`.
- `warmupEmbeddingModel` silently falls through tiers 1 → 2 → 3 without logging which shape was accepted [REVIEW noted in feature.md].
- Flexible-shape models (`RangeDim`) report shape `0` for ranges → falls through to hard-coded default `[1,1,80,998]` and `[1, weightFrames]`.
- Any CoreML prediction error in the final tier propagates upward; managers typically log + treat as non-fatal.

## Prompts and Models Used

## Usage Metrics

## Changelogs
