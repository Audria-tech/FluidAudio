---
id: performance-metrics
name: Performance Metrics
repo: FluidAudio
status: active
linked_features: []
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Performance Metrics

## TL;DR
`ASRPerformanceMetrics` is a `Codable` value type holding stage-by-stage ASR timings (preprocessor / encoder / decoder / total), the real-time factor (`rtfx`), peak resident memory in MB, and optional GPU utilization. Plus a `summary` text formatter for log output.

## Motivation — why it exists

## Context

## What It Does
Pure data holder. Construction is up to call sites (ASR pipelines populate it after measuring elapsed `Date` deltas and reading `SystemInfo.peakResidentMemoryBytes()`). The `summary` computed property formats every field into a multi-line string.

## Key Code
- [`Sources/FluidAudio/Shared/PerformanceMetrics.swift:4`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/PerformanceMetrics.swift#L4) — `ASRPerformanceMetrics` — `Codable, Sendable`, 7 fields + `summary`

## Edge Cases & Failure Modes
- `gpuUtilization` is `Float?` — `summary` renders `"N/A"` when nil.
- `rtfx` is unitless real-time factor (audio seconds / processing seconds); higher is better. No semantic validation.
- Not bound to any global registry; callers must persist instances themselves if they want historical data.

## Test Coverage
None dedicated; the struct is consumed by benchmark CLI tools and ASR demo outputs.

## Changelogs
