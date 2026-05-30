---
id: embedding-extractor
name: Embedding Extractor
repo: FluidAudio
status: active
linked_features: [diarizer-manager, diarizer-models]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Embedding Extractor

## TL;DR
`EmbeddingExtractor` runs the WeSpeaker 256-dim speaker-embedding CoreML model with ANE-aligned `MLMultiArray` buffers and zero-copy feature providers. It accepts raw 16 kHz audio (any `RandomAccessCollection<Float>`) plus per-speaker activity masks, and returns one 256-dim embedding per speaker, with inactive speakers producing zero vectors.

## Motivation — why it exists

## Context

## What It Does
The model has a fixed input shape of `[3, 160_000]` (batch=3 speakers per call, 160k float samples = 10s at 16 kHz). On `init`, the extractor stores the model and an `ANEMemoryOptimizer`. `getEmbeddings(audio:masks:minActivityThreshold:)` is the only public method:

1. Allocates two ANE-aligned `MLMultiArray`s — `waveformBuffer` shape `[3, 160_000]` and `maskBuffer` shape `[3, firstMask.count]`.
2. Fills the waveform buffer once via `fillWaveformBuffer`, which copies the input via `memoryOptimizer.optimizedCopy` then repeat-pads (doubling via `vDSP_mmov`) until 160k samples are reached. The padding fills *only the first speaker slot* — slots 1 and 2 of the batch dimension are left as whatever was in the buffer (the same audio, because we wrote into slot 0 and then doubled it past slot 0's boundary). [REVIEW: relies on repeat-padding overwriting slots 1 and 2 with copies of slot 0's audio; this is implicit and depends on slot stride.]
3. For each speaker mask, computes `speakerActivity = sum(mask)`. Below `minActivityThreshold` (default 10.0), it emits a zero-vector embedding and skips the model call entirely.
4. Otherwise, calls `fillMaskBufferOptimized` to vDSP-zero the mask buffer, copy this speaker's mask into slot 0, and repeat-pad to the full mask length.
5. Builds a `ZeroCopyDiarizerFeatureProvider` with the waveform + mask MLFeatureValues, calls `prefetchToNeuralEngine()` on both buffers, runs the model, and reads `embedding` from the output.
6. Extraction uses `memoryOptimizer.createZeroCopyView` when possible (offset-into-buffer view, then `memcpy` of 256 floats); falls back to `vDSP_mmov` if the view fails.

The `numMasksInChunk` calculation is `min((firstMask.count * audio.count + 80_000) / 160_000, firstMask.count)` — a rounded ratio of audio length to model frame count, clamped to the mask length to prevent buffer overflow when audio exceeds 160k samples.

## Key Code
- [`EmbeddingExtractor.swift:27`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Extraction/EmbeddingExtractor.swift#L27) — `getEmbeddings<C>(audio:masks:minActivityThreshold:)` is the sole public API; `C` is constrained to `RandomAccessCollection<Float>` with `Index == Int` to enable Array/ArraySlice/ContiguousArray callers.
- [`EmbeddingExtractor.swift:63`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Extraction/EmbeddingExtractor.swift#L63) — `numMasksInChunk` clamp; the inline comment explicitly notes this prevents heap-buffer-overflow when `audio.count > 160_000`.
- [`EmbeddingExtractor.swift:73`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Extraction/EmbeddingExtractor.swift#L73) — Activity filter: speakers below `minActivityThreshold` get a zero embedding without running the model.
- [`EmbeddingExtractor.swift:96`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Extraction/EmbeddingExtractor.swift#L96) — `prefetchToNeuralEngine()` is called on both buffers before every prediction to keep the ANE warm.
- [`EmbeddingExtractor.swift:118`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Extraction/EmbeddingExtractor.swift#L118) — `fillWaveformBuffer` uses repeat-doubling padding via `vDSP_mmov` to avoid energy distortion vs. zero padding.
- [`EmbeddingExtractor.swift:201`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Extraction/EmbeddingExtractor.swift#L201) — `extractEmbeddingOptimized` tries `createZeroCopyView` first, falls back to `vDSP_mmov`.

## Edge Cases & Failure Modes
- Empty `masks` array → returns `[]` immediately, no model call.
- Empty `audio` (`sampleCount == 0`) → `fillWaveformBuffer` returns early without padding, leaving the buffer in its initial state (zeros from ANE alignment).
- Missing `embedding` output → falls back to a zero `[Float](repeating: 0.0, count: 256)` embedding (no error thrown).
- Audio with `audio.count > 160_000` — the clamp in `numMasksInChunk` prevents buffer overflow but the model still only sees the first 160k samples (the `optimizedCopy` is bounded by buffer size).
- The waveform buffer is shared across all speakers in a single `getEmbeddings` call; the mask buffer is rewritten between speakers but the waveform is fixed. This assumes the model is masked on the input side via the mask channel.
- `if !copied` fallback path in the caller's `processChunkWithSpeakerTracking` doesn't apply here — `EmbeddingExtractor` always copies via `memoryOptimizer.optimizedCopy`, which handles non-contiguous Collections.

## Performance / Concurrency Notes
- The extractor is a `public final class` with all mutable state in `let` properties; it is effectively read-only after init and safe to call concurrently *if and only if* CoreML's `prediction` is reentrant on the held model — which is the case for compiled `MLModel` instances.
- ANE alignment + repeat-padding + zero-copy feature providers all aim to maximize Neural Engine throughput; the embedding model is the bottleneck in the offline pipeline at ~6.5ms per chunk on ANE per the offline config notes.
- Each `getEmbeddings` call allocates two fresh ANE-aligned buffers; there is no pool, so high-frequency calls will churn ANE memory. [REVIEW: contrast with `OfflineEmbeddingExtractor` which uses `getPooledBuffer` from `ANEMemoryOptimizer`.]

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/Extraction/EmbeddingExtractorOverflowTests.swift` covers the `numMasksInChunk` clamp and the `audio.count > 160_000` boundary.
- `Tests/FluidAudioTests/Diarizer/Offline/EmbeddingExtractorMemoryTests.swift` exercises adjacent ANE-aligned buffer memory behavior shared with the offline extractor.

## Changelogs
