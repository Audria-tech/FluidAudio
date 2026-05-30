---
id: fluidaudio-umbrella
name: FluidAudio Umbrella
repo: FluidAudio
status: active
linked_features: []
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# FluidAudio Umbrella

## TL;DR
Top-level module file. Defines a no-op `public struct FluidAudio` (namespace placeholder) and two backward-compat typealiases (`SpeakerDiarizationConfig = DiarizerConfig`, `SpeakerDiarizationError = DiarizerError`). All actual functionality is exposed by sibling files via `public` declarations.

## Motivation — why it exists

## Context

## What It Does
Single-file shim. The struct exists to give consumers a discoverable `FluidAudio` identifier; the typealiases keep old call sites compiling after the rename from "SpeakerDiarization" to "Diarizer".

## Key Code
- [`Sources/FluidAudio/FluidAudioSwift.swift:12`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/FluidAudioSwift.swift#L12) — `SpeakerDiarizationConfig` / `SpeakerDiarizationError` typealiases + empty `FluidAudio` namespace struct

## Edge Cases & Failure Modes
- Empty struct creates "namespace collision" concerns that the file's own comments call out — types like `RawEmbedding` etc. are not nested under `FluidAudio`. [REVIEW: `FluidAudio` empty struct exists as a namespace placeholder but no types nest under it — comment in source acknowledges the collision risk]

## Test Coverage
No dedicated tests; existence is validated by `Tests/FluidAudioTests/CI/BasicInitializationTests.swift`.

## Changelogs
