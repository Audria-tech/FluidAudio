---
id: qwen3-asr-models
name: Qwen3-ASR Models Container
repo: FluidAudio
status: active
linked_features:
  - qwen3-asr-manager
  - qwen3-asr-config
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Qwen3-ASR Models Container

## TL;DR
`Qwen3AsrModels` is the Sendable container that bundles the audio encoder, the stateful decoder (with fused LM head), the Swift-side `EmbeddingWeights`, and the vocabulary for the 2-model Qwen3-ASR pipeline. Defines the `Qwen3AsrVariant` enum (`.f32` / `.int8`) for precision selection, the download + load + cache helpers, and the FP16 embedding-matrix loader that the manager uses to skip a CoreML call per token.

## Context

## What It Does
`Qwen3AsrVariant` maps precisions to HuggingFace repos. The `load(from:computeUnits:)` factory loads the two CoreML models with on-the-fly `mlpackage → mlmodelc` compilation (cached after first compile), loads the binary embeddings file via `EmbeddingWeights.init(contentsOf:)`, and parses `vocab.json` (string → int) into the reversed `[Int: String]` mapping the manager uses for detokenization. `downloadAndLoad(...)` chains `download(...)` + `load(...)`; `download(...)` calls `DownloadUtils.downloadRepo(...)` and is force-deletable.

`EmbeddingWeights` is a `Sendable` final class that mmaps (`Data(contentsOf:)`) the binary embeddings file with a `uint32 vocab, uint32 hidden, float16[vocab*hidden]` layout. Validates that the on-disk vocab/hidden sizes match `Qwen3AsrConfig` and that file size matches `8 + vocabSize*hiddenSize*2`. `embedding(for:)` reads one row as Float16 → Float32 on Apple Silicon (`#if arch(arm64)`); other archs `fatalError` because Swift `Float16` is arm64-only. `embeddings(for:)` is the per-token-list convenience.

## Key Code
- [`Qwen3AsrModels.swift:8`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrModels.swift#L8) — `Qwen3AsrVariant` precision enum.
- [`Qwen3AsrModels.swift:35`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrModels.swift#L35) — `Qwen3AsrModels` struct.
- [`Qwen3AsrModels.swift:50`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrModels.swift#L50) — `load(from:computeUnits:)` factory.
- [`Qwen3AsrModels.swift:179`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrModels.swift#L179) — `loadModel(named:...)` mlpackage → mlmodelc compile-and-cache.
- [`Qwen3AsrModels.swift:258`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrModels.swift#L258) — `EmbeddingWeights` class.
- [`Qwen3AsrModels.swift:265`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrModels.swift#L265) — header parse + size validation.
- [`Qwen3AsrModels.swift:302`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrModels.swift#L302) — `embedding(for:)` arm64-only Float16 read.
- [`Qwen3AsrModels.swift:336`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3AsrModels.swift#L336) — `Qwen3AsrError` enum.

## Edge Cases & Failure Modes
- Non-arm64 builds will `fatalError` at first embedding lookup — the Qwen3 path is effectively Apple Silicon only.
- Embedding file size mismatch is caught with a precise expected vs actual byte count.
- Out-of-range token IDs in `embedding(for:)` return a zero vector instead of crashing — silent failure mode if upstream IDs are wrong.
- `vocab.json` must be `[String: Int]`; the manager inverts to `[Int: String]`.
- `defaultCacheDirectory(variant:)` falls back to `temporaryDirectory` if Application Support is unavailable — cache persistence breaks but the pipeline still functions.

## Test Coverage
- No dedicated test file for `Qwen3AsrModels`; covered indirectly by manager-level tests.

## Changelogs
