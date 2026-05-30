---
id: magpie-constants
name: Magpie Constants
repo: FluidAudio
status: active
linked_features: [magpie-tts-manager, magpie-pipeline, magpie-types]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Magpie Constants

## TL;DR
Compile-time constants for the NVIDIA Magpie TTS Multilingual 357M backend: 22.05 kHz output, 768-d transformer hidden, 12 decoder layers / heads (`headDim = 64`), 512 KV-cache positions, 256 text tokens, 8 codebooks × 2024 FSQ codes, and the special audio token ids (`audioBosId = 2016`, `audioEosId = 2017`, forbidden ids 2016/2018-2023).

## Motivation — why it exists

## Context

## What It Does
`MagpieConstants` is a pure `enum` namespace with no runtime state. Sections:
- **Audio**: `audioSampleRate = 22_050`, `codecSamplesPerFrame = 1_024` (NanoCodec is 21.5 fps at 22050 Hz ⇒ ~1024 samples/frame), `peakTarget: Float = 0.9`.
- **Model dimensions**: `dModel = 768`, `numDecoderLayers = 12`, `numHeads = 12`, `headDim = 64`, `maxCacheLength = 512`, `maxTextLength = 256`.
- **NanoCodec**: `numCodebooks = 8`, `numCodesPerCodebook = 2_024`, `maxNanocodecFrames = 256`.
- **Special audio token ids**: `audioBosId = 2_016`, `audioEosId = 2_017`, `forbiddenAudioIds = [2016, 2018, 2019, 2020, 2021, 2022, 2023]` (CTX_BOS, CTX_EOS, MASK, reserved).
- Sampling defaults (referenced by `MagpieSynthesisOptions`): default temperature, topK, max steps, min frames, default cfgScale (defined further down the file).

## Key Code
- [`Sources/FluidAudio/TTS/Magpie/MagpieConstants.swift:7`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieConstants.swift#L7) — top of `public enum MagpieConstants`.
- [`Sources/FluidAudio/TTS/Magpie/MagpieConstants.swift:12`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieConstants.swift#L12) — `audioSampleRate = 22_050`.
- [`Sources/FluidAudio/TTS/Magpie/MagpieConstants.swift:21`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieConstants.swift#L21) — model dims (`dModel`, `numHeads`, `headDim`, `maxCacheLength`).
- [`Sources/FluidAudio/TTS/Magpie/MagpieConstants.swift:36`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieConstants.swift#L36) — NanoCodec parameters.
- [`Sources/FluidAudio/TTS/Magpie/MagpieConstants.swift:45`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/MagpieConstants.swift#L45) — special audio token ids.

## Edge Cases & Failure Modes
- All values match the upstream model card; changing requires a re-conversion of the CoreML graphs.
- `maxNanocodecFrames = 256` caps a single NanoCodec forward pass — long synthesis is split across multiple chunks.

## Test Coverage
- `Tests/FluidAudioTests/TTS/Magpie/MagpieConstantsTests.swift`

## Changelogs
