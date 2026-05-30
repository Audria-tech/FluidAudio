---
id: offline-extraction
name: Offline Extraction
repo: FluidAudio
status: active
linked_features: [offline-diarizer-manager, offline-diarizer-types, offline-utils]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Offline Extraction

## TL;DR
Three files: `OfflineEmbeddingExtractor` runs the FBANK + 256-dim ECAPA-style embedding model with overlap-masking and skip-strategy support; `PLDATransform` projects 256-dim embeddings to 128-dim PLDA "rho" space via a separate CoreML model; `WeightInterpolation` resamples soft VAD weights with scipy.ndimage.zoom half-pixel semantics.

## Motivation — why it exists

## Context

## What It Does
**`OfflineEmbeddingExtractor`** (`@available(macOS 14.0, iOS 17.0, *)`) consumes a stream of `SegmentationChunk`s and a backing `audioSource`, batches up to `config.embedding.batchSize` (≤32) chunks per FBANK + embedding model call, and produces `[TimedEmbedding]`. For each chunk it derives per-speaker masks (excluding overlapped frames when `excludeOverlap=true`), runs FBANK on the audio + weight, runs the embedding model on the FBANK features + weights, and (after the loop) batches the resulting embeddings through `PLDATransform` to get the 128-dim `rho`. The skip strategy (`.none` or `.maskSimilarity(threshold:)`) compares the current speaker mask cosine-similarity against the mask that generated the cached embedding for that speaker; if ≥ threshold, the cached `embedding256` + a freshly transformed `rho128` are reused. Resolves model input/output names dynamically from the model description (`audio`/`fbank_features`/`weight`/`embedding`) so it can adapt to renamed CoreML exports.

**`PLDATransform`** runs `pldaRhoModel` in `maxBatchSize=32` chunks, with a warmup prediction on init. Output shape: 128-dim `[Double]`.

**`WeightInterpolation.InterpolationCoefficients`** precomputes left/right indices and blend weights for resampling an `inputLength`→`outputLength` curve using the half-pixel formula `(idx + 0.5) / scale - 0.5`, clamped to `[0, inputLength-1]`. The precomputed coefficients enable repeated resampling of multiple mask rows without recomputing the mapping.

## Key Code
- [`OfflineEmbeddingExtractor.swift:81`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Extraction/OfflineEmbeddingExtractor.swift#L81) — `init(fbankModel:embeddingModel:pldaTransform:config:)` — dynamic model-description introspection to resolve input/output names.
- [`OfflineEmbeddingExtractor.swift:7`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Extraction/OfflineEmbeddingExtractor.swift#L7) — `CachedSpeakerEmbedding` carries the mask that generated each cached embedding (for skip-strategy mask-similarity comparisons).
- [`PLDATransform.swift:26`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Extraction/PLDATransform.swift#L26) — `transform(_:[[Float]])` batched async transform.
- [`WeightInterpolation.swift:19`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Extraction/WeightInterpolation.swift#L19) — `InterpolationCoefficients(inputLength:outputLength:)` precomputation.

## Edge Cases & Failure Modes
- `OfflineEmbeddingExtractor.init` calls `preconditionFailure` if the FBANK model is missing its `audio` MLMultiArray input — this is fatal at startup.
- The `embedding` model batch limit is derived from the model's batch dimension; falls back to `config.embedding.batchSize`.
- Skip strategy compares against the *cached* mask (not the previous mask) to prevent drift. The docstring explicitly notes this.
- `PLDATransform.transform` skips warmup on error (logs at debug level) and proceeds with the real transform.
- `WeightInterpolation` `init` requires `inputLength > 0` and `outputLength > 0` (precondition).
- `clamped = min(max(position, 0), Float(inputLength - 1))` — out-of-range positions snap to the nearest valid index.
- The skip strategy reuses the cached `embedding256` but re-runs PLDA on it. [REVIEW: re-running PLDA on the same embedding is deterministic; could cache `rho128` too for further savings.]

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/Offline/EmbeddingExtractorMemoryTests.swift` — memory profile.
- `Tests/FluidAudioTests/Diarizer/Offline/WeightInterpolationTests.swift` — scipy.ndimage.zoom parity.

## Changelogs
