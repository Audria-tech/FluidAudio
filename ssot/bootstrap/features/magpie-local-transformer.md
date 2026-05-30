---
id: magpie-local-transformer
name: Magpie Local Transformer (8-Codebook Sampler)
repo: FluidAudio
status: active
linked_features: [magpie-tts-manager, magpie-pipeline]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Magpie Local Transformer (8-Codebook Sampler)

## TL;DR
Pure-Swift, BLAS-backed 1-layer "local transformer" that samples the 8 NanoCodec codebook tokens per AR step. Mirrors `local_transformer_forward` from the upstream `generate_coreml.py`. Pre-norm causal self-attention + pre-norm FFN with tanh-GELU; single attention head, `localDim = 256`. Stateless across frames — caller rebuilds the input sequence each step.

## Motivation — why it exists

## Context

## What It Does
`MagpieLocalTransformer` is a `Sendable` struct that holds a `MagpieLocalTransformerWeights` payload and exposes `forward(sequence:length:)`. The forward pass adds positional embeddings, runs a single attention head over `T ≤ numCodebooks + 2` positions, then a tanh-GELU FFN — every matmul is `cblas_sgemm` so the AR loop stays cache-resident. `MagpieSampler` lives next door and wraps the forward pass with the actual token-sampling loop (per-codebook logit projection + topK + temperature). Weights load from a `.npy` blob via `NpyReader` (see `MagpieLocalTransformerWeights`); they're CPU-resident fp32 and small enough to stay hot in L2.

## Key Code
- [`Sources/FluidAudio/TTS/Magpie/LocalTransformer/MagpieLocalTransformer.swift:15`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/LocalTransformer/MagpieLocalTransformer.swift#L15) — `public struct MagpieLocalTransformer: Sendable`.
- [`Sources/FluidAudio/TTS/Magpie/LocalTransformer/MagpieLocalTransformer.swift:29`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/LocalTransformer/MagpieLocalTransformer.swift#L29) — `forward(sequence:length:)` with positional add, pre-norm attention, pre-norm FFN.
- [`Sources/FluidAudio/TTS/Magpie/LocalTransformer/MagpieSampler.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/LocalTransformer/MagpieSampler.swift) — codebook-loop sampler.
- [`Sources/FluidAudio/TTS/Magpie/Assets/MagpieLocalTransformerWeights.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/Assets/MagpieLocalTransformerWeights.swift) — weights payload.

## Edge Cases & Failure Modes
- Caller must pass `T` explicitly to disambiguate partial buffers; preconditions enforce `sequence.count >= T * localDim` and `T <= maxPositions`.
- Stateless design means the sampler reallocates input each step — relies on the small `T` (≤ 10) to stay cheap.
- [REVIEW: performance — chosen as Swift+BLAS rather than a 9th CoreML graph; the doc cites cache-residency. This is a contributor to the overall slow Magpie RTFx; confirm whether ANE-fusing the local transformer is feasible.]

## Test Coverage
- No dedicated `MagpieLocalTransformerTests` in scope; covered indirectly by `MagpieKvCacheTests`, `MagpieNpyReaderTests`, `MagpieMT19937Tests`.

## Changelogs
