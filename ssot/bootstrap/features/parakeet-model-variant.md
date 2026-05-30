---
id: parakeet-model-variant
name: Parakeet Streaming Model Variant
repo: FluidAudio
status: active
linked_features:
  - streaming-asr-manager
  - eou-detector
  - nemotron-streaming
  - sliding-window-asr-session
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Parakeet Streaming Model Variant

## TL;DR
`StreamingModelVariant` is the public enum that catalogues all true streaming Parakeet engines (cache-aware encoders) and serves as their factory. It maps each variant to a `Repo`, an `engineFamily` (`.parakeetEou` / `.nemotron`), a chunk size, a display name, and — via `createManager(configuration:)` — to a concrete `any StreamingAsrManager` instance. Explicitly excludes Parakeet TDT, which is offline-encoder + sliding-window.

## Context

## What It Does
Five cases are defined: `.parakeetEou160ms`, `.parakeetEou320ms`, `.parakeetEou1280ms`, `.nemotron560ms`, `.nemotron1120ms`. Each case projects through `displayName`, `repo`, `engineFamily`, and either `eouChunkSize` (returns `StreamingChunkSize?`) or `nemotronChunkSize` (returns `NemotronChunkSize?`). `createManager(configuration:)` switches on `engineFamily` and constructs the appropriate manager — `StreamingEouAsrManager(configuration:chunkSize:)` for EOU, `StreamingNemotronAsrManager(configuration:requestedChunkSize:)` for Nemotron — falling back to defaults if the chunk-size lookup returns nil. The returned engine is `any StreamingAsrManager` (not loaded yet — caller invokes `loadModels()`).

## Key Code
- [`ParakeetModelVariant.swift:14`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/ParakeetModelVariant.swift#L14) — enum + 5 cases.
- [`ParakeetModelVariant.swift:43`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/ParakeetModelVariant.swift#L43) — `repo` mapping into `Repo` enum.
- [`ParakeetModelVariant.swift:54`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/ParakeetModelVariant.swift#L54) — `engineFamily` factory dispatch key.
- [`ParakeetModelVariant.swift:88`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/ParakeetModelVariant.swift#L88) — `createManager(configuration:)` polymorphic constructor.
- [`ParakeetModelVariant.swift:103`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/ParakeetModelVariant.swift#L103) — `EngineFamily` inner enum.

## Edge Cases & Failure Modes
- `eouChunkSize` returns nil for Nemotron variants (and vice versa); the factory uses sensible defaults (`.ms160` / `.ms1120`) when nil, which can mask programmer error if a new case is added without updating both mappings.
- `Sendable` conformance is `String`-backed so the enum can cross actor boundaries cheaply.
- The factory does not check that the configuration's compute units are sensible for the variant — Nemotron 1120ms benefits from ANE but the call site is responsible.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Parakeet/Streaming/NemotronChunkSizeTests.swift`, `Tests/FluidAudioTests/ASR/Parakeet/Streaming/EouChunkSizeFrameCountTests.swift` (chunk-size math).

## Changelogs
