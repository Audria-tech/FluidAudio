---
id: model-warmup
name: Model Warmup
repo: FluidAudio
status: active
linked_features: []
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Model Warmup

## TL;DR
Internal helpers that run zero-valued predictions against a freshly-loaded Core ML model so the ANE/GPU has a chance to allocate compute buffers before the offline diarizer's first real inference. Two entry points: generic single-input warmup and a specialised embedding-model warmup that probes the model's input descriptions to pick the right shape.

## Motivation — why it exists

## Context

## What It Does
`warmup(model:inputName:inputShape:iterations:)` allocates an `MLMultiArray` of the requested shape and dtype `.float32`, zero-fills it with `resetToZeros()` (a vDSP-accelerated `vDSP_vfill` in a `fileprivate` extension), wraps it in an `MLDictionaryFeatureProvider`, and calls `model.prediction(from:)` `iterations` times. Returns elapsed seconds.

`warmupEmbeddingModel(_:weightFrames:audioSamples:)` first inspects `model.modelDescription.inputDescriptionsByName` for the modern `fbank_features` + `weights` shape — if both are present and have strictly-positive constraint shapes it uses those; otherwise it falls back to `[1, 1, 80, 998]` and `[1, weightFrames]`. If the modern call throws (older embedding model formats) it falls back twice more: combined `[1, 1, 1, audioSamples + weightFrames]` `audio_and_weights` input, and finally a legacy dual `audio` + `weights` input pair.

## Key Code
- [`Sources/FluidAudio/Shared/ModelWarmup.swift:17`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ModelWarmup.swift#L17) — `warmup` — single-input generic primer
- [`Sources/FluidAudio/Shared/ModelWarmup.swift:47`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ModelWarmup.swift#L47) — `warmupEmbeddingModel` — three-tier fallback (`fbank_features`+`weights` → `audio_and_weights` → `audio`+`weights`)
- [`Sources/FluidAudio/Shared/ModelWarmup.swift:147`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ModelWarmup.swift#L147) — `resetToZeros` — `vDSP_vfill` over the data pointer

## Edge Cases & Failure Modes
- Preconditions on `iterations > 0` and non-empty `inputShape` — trap on misuse rather than throw.
- Fallback in `warmupEmbeddingModel` is silent: a failed prediction in tier 1 falls through to tier 2 with no logging; the same for tier 2 → tier 3. The final tier is a `try` that propagates errors. [REVIEW: `warmupEmbeddingModel` falls through tiers silently — hard to diagnose which input shape the model actually accepted]
- `resetToZeros` assumes the data pointer can be bound to `Float` — works for `.float32` only. Other dtypes would write `Float` bits over differently-sized cells. The method is `fileprivate` so external misuse is prevented.
- Shape constraint validation only checks `> 0` on each dim — flexible-shape models with `RangeDim` constraints return shape `0` for ranges and fall through to the hard-coded default.
- Result of the warmup prediction is discarded — caller cannot inspect anything other than timing.

## Test Coverage
No dedicated tests; warmup is exercised through diarizer initialization paths.

## Changelogs
