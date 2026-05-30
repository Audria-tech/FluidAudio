---
id: qwen3-rope
name: Qwen3 RoPE (Rotary Position Embeddings)
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

# Qwen3 RoPE (Rotary Position Embeddings)

## TL;DR
`Qwen3RoPE` precomputes the cos/sin tables for every position `[0, maxCacheSeqLen)` (512) at init so the hot decode loop only memcpys into the input arrays. Qwen3 uses M-RoPE with sections `[24, 20, 20]` and `theta = 1_000_000` — for ASR (no spatial dims) this collapses to standard RoPE over `headDim = 128` with the concatenated-halves layout the CoreML model's `rotate_half` op expects.

## Context

## What It Does
At init, `invFreq[i] = 1 / theta^(2i/dim)` is computed for `i in [0, headDim/2)`, then for every position `p in [0, 512)` cos and sin are filled into flat tables laid out as `[pos0(128), pos1(128), ...]` row-major. The crucial detail is the per-position layout: cos values at `[i]` and `[i + halfDim]` are *the same* — concatenated-halves so the CoreML rotate-half splits at `headDim / 2`.

`fill(position:cosPtr:sinPtr:)` memcpys 128 floats directly into caller-supplied raw pointers — used by the decode loop's preallocated `decodeCosArray` / `decodeSinArray`. `compute(position:)` returns standalone `[Float]` arrays for callers building `MLMultiArray` independently. `computeRange(startPosition:count:)` returns `count * 128` floats for prefill (batched position embeddings for the entire prompt). Positions beyond `maxPosition` fall back to `computeDynamic(...)` / `computeRangeDynamic(...)` which run the trig live.

## Key Code
- [`Qwen3RoPE.swift:13`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3RoPE.swift#L13) — `Qwen3RoPE` struct.
- [`Qwen3RoPE.swift:27`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3RoPE.swift#L27) — `init()` precomputes all positions with concatenated-halves layout.
- [`Qwen3RoPE.swift:68`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3RoPE.swift#L68) — `fill(position:cosPtr:sinPtr:)` memcpy into raw pointers (hot path).
- [`Qwen3RoPE.swift:113`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3RoPE.swift#L113) — `computeRange(startPosition:count:)` for batched prefill.
- [`Qwen3RoPE.swift:128`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/Qwen3RoPE.swift#L128) — `computeDynamic(...)` fallback for positions beyond the precomputed table.

## Edge Cases & Failure Modes
- `maxPosition = maxCacheSeqLen = 512` — any position ≥ 512 hits the dynamic path; this is slower per-position but functionally identical.
- The concatenated-halves layout (cos at `[i]` and `[i+halfDim]` the same) is mandatory — the CoreML model assumes it. Splitting into `[cos, sin]` interleaved or other variants will silently produce garbage.
- `Sendable` conformance is safe because all stored state is immutable after init.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Qwen3/Qwen3RoPETests.swift`.

## Changelogs
