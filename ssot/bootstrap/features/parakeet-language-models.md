---
id: parakeet-language-models
name: Parakeet Language Models (Generic Container)
repo: FluidAudio
status: active
linked_features:
  - ctc-decoder
  - sliding-window-asr-manager
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Parakeet Language Models (Generic Container)

## TL;DR
`ParakeetLanguageModels<Config: ParakeetLanguageModelConfig>` is the generic container that holds preprocessor / encoder / decoder / optional joint CoreML models plus a vocabulary for language-specific Parakeet builds. The `ParakeetLanguageModelConfig` protocol is the static knob-board (blank ID, repo, file names, int8-encoder support, joint-file presence) so a new language (e.g. `CtcZhCnConfig`, `TdtJaConfig`) is added by writing a new conforming type.

## Context

## What It Does
The struct stores the four model objects, the `MLModelConfiguration`, the `[Int: String]` vocabulary, and the blank token ID. The `load(from:useInt8Encoder:configuration:progressHandler:)` static factory resolves the encoder file name (int8 vs fp32 if supported), calls `DownloadUtils.loadModels(...)` for the bundle of `[preprocessorFile, encoderFile, decoderFile, jointFile?]`, validates each result, loads the vocabulary, and returns the populated container. `download(...)` performs the cache-aware download (optionally fetching both encoder precisions). `downloadAndLoad(...)` is the convenience that chains both. `modelsExist(at:)` checks that all required files (and at least one encoder variant for int8-capable configs) plus the vocab are present. `loadVocabulary(from:)` accepts both array format (`["<unk>", "▁t", ...]`) and dictionary format (`{"0": "<unk>", ...}`).

## Key Code
- [`ParakeetLanguageModels.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/ParakeetLanguageModels.swift#L5) — `ParakeetLanguageModelConfig` protocol surface.
- [`ParakeetLanguageModels.swift:35`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/ParakeetLanguageModels.swift#L35) — generic struct definition.
- [`ParakeetLanguageModels.swift:78`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/ParakeetLanguageModels.swift#L78) — `load(from:...)` factory.
- [`ParakeetLanguageModels.swift:172`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/ParakeetLanguageModels.swift#L172) — `download(...)` cache-aware downloader.
- [`ParakeetLanguageModels.swift:235`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/ParakeetLanguageModels.swift#L235) — `downloadAndLoad(...)` convenience.
- [`ParakeetLanguageModels.swift:303`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/ParakeetLanguageModels.swift#L303) — `loadVocabulary(from:)` supports array + dictionary formats.

## Edge Cases & Failure Modes
- If int8 is enabled but neither int8 nor fp32 encoder exists locally, `modelsExist(at:)` returns false and triggers re-download.
- Missing joint file when `Config.jointFile` is non-nil throws `AsrModelsError.loadingFailed`.
- `loadVocabulary` falls back from array → dictionary parsing; if both fail it throws `AsrModelsError.loadingFailed`.
- `force: true` removes the entire target directory before re-download — destroys any sibling files.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Parakeet/SlidingWindow/CTC/CtcZhCnTests.swift` and TDT-Ja tests exercise both int8-capable and joint-equipped configs.

## Changelogs
