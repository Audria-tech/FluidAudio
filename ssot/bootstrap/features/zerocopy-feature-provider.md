---
id: zerocopy-feature-provider
name: Zero-Copy Feature Provider
repo: FluidAudio
status: active
linked_features: [ane-memory-optimizer]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Zero-Copy Feature Provider

## TL;DR
Minimal `MLFeatureProvider` whose `featureValue(for:)` reads from a fixed `[String: MLFeatureValue]` dictionary. Used to wrap model outputs (`MLFeatureValue` holds an `MLMultiArray` by reference) and pass them as inputs to the next model in a chain without copying the underlying buffer.

## Motivation — why it exists

## Context

## What It Does
Constructed with a `[String: MLFeatureValue]` and exposes Core ML's `MLFeatureProvider` protocol over it. The `chain(from:outputName:to:)` static factory pulls one named feature from an upstream `MLFeatureProvider` (typically a model's prediction result) and rewraps it under a new input name for the downstream model — there is no `MLMultiArray` copy, just a feature-value re-key. A second feature-provider class (`ZeroCopyDiarizerFeatureProvider`) lives in `ANEMemoryOptimizer.swift` and adds diarizer-specific batch helpers (waveform + mask pairs).

## Key Code
- [`Sources/FluidAudio/Shared/ZeroCopyFeatureProvider.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ZeroCopyFeatureProvider.swift#L5) — class definition; stored dict + protocol conformance
- [`Sources/FluidAudio/Shared/ZeroCopyFeatureProvider.swift:22`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ZeroCopyFeatureProvider.swift#L22) — `chain(from:outputName:to:)` — rename a feature without copying

## Edge Cases & Failure Modes
- `chain` returns `nil` if the upstream output is missing — silent failure that callers must check.
- Dictionary is captured by value at init; subsequent mutations to the source dict don't affect the provider.
- Class is not marked `Sendable` even though it's a `let`-stored immutable dict — sharing across actors requires careful Core ML invocation. [REVIEW: `ZeroCopyFeatureProvider` is not `Sendable` despite logically being immutable]

## Test Coverage
No dedicated unit tests; exercised through every multi-stage ASR / Diarizer pipeline that chains models.

## Changelogs
