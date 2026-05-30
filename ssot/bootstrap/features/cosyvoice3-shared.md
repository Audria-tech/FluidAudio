---
id: cosyvoice3-shared
name: CosyVoice3 Shared (Safetensors Reader)
repo: FluidAudio
status: active
linked_features: [cosyvoice3-pipeline-preprocess, cosyvoice3-pipeline-synthesize]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# CosyVoice3 Shared (Safetensors Reader)

## TL;DR
Minimal zero-dependency `SafetensorsFile` parser. Reads the standard safetensors file format (u64 header length + UTF-8 JSON manifest + raw tensor payload) and exposes typed tensor views. Used by CosyVoice3 Phase 1 fixture and the speech-embedding/text-embedding mmap paths.

## Motivation — why it exists

## Context

## What It Does
`SafetensorsFile` is a `public final class` that mmaps a safetensors file and exposes typed accessors. The file format is documented in the source header: little-endian `u64` header length N, then N bytes of UTF-8 JSON of shape `{ "<name>": {"dtype": "...", "shape": [...], "data_offsets": [start, end]}, ... }`, then the raw tensor payload referenced by those offsets.

A nested `DType` enum covers `F16`, `BF16`, `F32`, `F64`, `I8`/`I16`/`I32`/`I64`, `U8`/`U16`/`U32`/`U64` — i.e. the standard safetensors dtype set. Consumers use the file to read individual tensor views by name (callers known: `CosyVoice3SpeechEmbeddings`, `CosyVoice3TextEmbeddings`, `CosyVoice3PromptAssets`, `CosyVoice3PromptMel`, the fixture loader).

## Key Code
- [`Sources/FluidAudio/TTS/CosyVoice3/Shared/SafetensorsReader.swift:11`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Shared/SafetensorsReader.swift#L11) — `public final class SafetensorsFile`.
- [`Sources/FluidAudio/TTS/CosyVoice3/Shared/SafetensorsReader.swift:13`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Shared/SafetensorsReader.swift#L13) — `DType` enum.

## Edge Cases & Failure Modes
- Malformed JSON header / out-of-range offsets surface as parse errors.
- Endianness is assumed little-endian — safetensors spec mandates this.
- BF16 / F64 are listed in `DType` but downstream CosyVoice3 callers only consume F16/F32/I32-class views.
- [REVIEW: no integrity check (hash / signature) — relies on HF CDN for delivery guarantees.]

## Test Coverage
- No dedicated `SafetensorsReaderTests` in scope; exercised via the CosyVoice3 fixture / prompt loaders.

## Changelogs
