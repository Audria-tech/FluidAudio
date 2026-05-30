---
id: mlmodel-configuration-utils
name: MLModelConfiguration Utils
repo: FluidAudio
status: active
linked_features: [model-names]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# MLModelConfiguration Utils

## TL;DR
Two static helpers used across pipelines: `defaultConfiguration(computeUnits:)` returns a stock `MLModelConfiguration` with low-precision GPU accumulation enabled, and `defaultModelsDirectory(for:)` produces the cache directory under `~/Library/Application Support/FluidAudio/Models/` (optionally appended with a `Repo.folderName`).

## Motivation — why it exists

## Context

## What It Does
`defaultConfiguration` sets `allowLowPrecisionAccumulationOnGPU = true` and `computeUnits = .cpuAndNeuralEngine` by default — every FluidAudio model loader uses this shape so ANE-eligible ops run on the Neural Engine while CPU handles the rest. `defaultModelsDirectory` resolves `.applicationSupportDirectory` from `FileManager` and appends `FluidAudio/Models[/<Repo.folderName>]`.

## Key Code
- [`Sources/FluidAudio/Shared/MLModelConfigurationUtils.swift:11`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/MLModelConfigurationUtils.swift#L11) — `defaultConfiguration(computeUnits:)` — `allowLowPrecisionAccumulationOnGPU = true`
- [`Sources/FluidAudio/Shared/MLModelConfigurationUtils.swift:25`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/MLModelConfigurationUtils.swift#L25) — `defaultModelsDirectory(for:)` — Application Support path

## Edge Cases & Failure Modes
- Force-unwraps `.applicationSupportDirectory` lookup (`.first!`) — would trap on a sandboxed environment that returns an empty array (extremely rare on Apple platforms but possible in restricted contexts). [REVIEW: `defaultModelsDirectory` force-unwraps `FileManager.urls(...)` — fragile under sandbox restrictions]
- Does not create the directory; callers must `createDirectory(at:withIntermediateDirectories:)` themselves (`DownloadUtils.loadModelsOnce` does this).
- The default Application Support path is the macOS/iOS standard — TTS uses a different `~/.cache/fluidaudio/Models` root (see `DownloadUtils.clearAllModelCaches`). This helper does not cover the TTS path.

## Test Coverage
No dedicated tests; covered indirectly by every load path.

## Changelogs
