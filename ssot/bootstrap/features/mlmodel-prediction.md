---
id: mlmodel-prediction
name: MLModel Prediction Compat
repo: FluidAudio
status: active
linked_features: []
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# MLModel Prediction Compat

## TL;DR
Tiny `MLModel` extension that exposes a `compatPrediction(from:options:)` wrapper over Core ML's async `prediction` API. The import is `@preconcurrency import CoreML` so callers using strict-concurrency builds don't need to mark `MLModel` themselves.

## Motivation — why it exists

## Context

## What It Does
Calls `prediction(from: input, options: options)` directly. The wrapper exists so a single name (`compatPrediction`) can be swapped out if Core ML's prediction signature changes again, and so the `@preconcurrency` annotation is centralised.

## Key Code
- [`Sources/FluidAudio/Shared/MLModel+Prediction.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/MLModel%2BPrediction.swift#L5) — `compatPrediction(from:options:)` async wrapper

## Edge Cases & Failure Modes
- No fallback to the synchronous `prediction(from:)` form — purely a name passthrough at this SHA. [REVIEW: `compatPrediction` is currently a 1-line passthrough — purpose only justifies itself if the compat layer ever forks the implementation]
- Rethrows any Core ML prediction error unchanged.

## Test Coverage
Exercised indirectly through every model invocation in ASR / Diarizer / TTS pipelines.

## Changelogs
