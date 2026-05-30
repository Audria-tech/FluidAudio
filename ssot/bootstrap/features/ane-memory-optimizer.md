---
id: ane-memory-optimizer
name: ANE Memory Optimizer
repo: FluidAudio
status: active
linked_features: [mlarray-cache, zerocopy-feature-provider]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# ANE Memory Optimizer

## TL;DR
ANE-aware allocator + buffer-pool for `MLMultiArray`s used by the diarizer and ASR hot paths. Allocates 64-byte-aligned, tile-padded backing memory via `posix_memalign`, computes ANE-friendly strides, supports zero-copy views and stride-aware copies, and offers vDSP-accelerated bulk copies for Float32 collections.

## Motivation — why it exists

## Context

## What It Does
`ANEMemoryUtils` is the low-level allocator. `createAlignedArray(shape:dataType:zeroClear:)` allocates a `posix_memalign`'d buffer at the `aneAlignment = 64` boundary, sized to the stride-padded total elements (innermost dimension is padded up to a multiple of `aneTileSize = 16` when not already aligned). The buffer is handed to `MLMultiArray.init(dataPointer:shape:dataType:strides:deallocator:)` with a `deallocator` that calls `Darwin.free` — using `UnsafeMutableRawPointer.deallocate()` would trap because the memory came from `posix_memalign`, not `Allocator.allocate`. Zero-clear is opt-in (defaults to true) and uses `memset` over the whole aligned region.

Strides are computed last-dimension-first with currentStride bumped by `paddedSize = ceilToTile(dim)` when the innermost dim isn't a multiple of `aneTileSize`. This produces a stride layout that aligns each "row" to the ANE's DMA tile boundary — important for the Neural Engine's matrix-tile pipeline. `getElementSize` maps `MLMultiArrayDataType` to byte sizes (fp16=2, fp32=4, fp64=8, int32=4, unknown → `MemoryLayout<Float>.stride`).

`ANEMemoryOptimizer` wraps the allocator with a per-instance buffer pool (`[String: MLMultiArray]` guarded by `NSLock`) keyed by string ID — `getPooledBuffer(key:shape:dataType:)` returns a cached array if the shape/dtype matches, otherwise allocates a fresh one. `optimizedCopy` is the bulk-write helper: it checks the source's runtime type (`[Float]`, `ArraySlice<Float>`, `ContiguousArray<Float>`) and dispatches to `vDSP_mmov` when the source is contiguous, falling back to an element-by-element copy otherwise. Optional `pad: true` zeros the remainder of the destination with `vDSP_vclr`.

Two view APIs are provided. `ANEMemoryUtils.createZeroCopyView` does stride-aware bounds checking (using `backingElements = strides[0] * shape[0]`, accounting for ANE padding) before slicing into the source pointer. `ANEMemoryOptimizer.createZeroCopyView` is a thinner variant that uses logical element counts (not stride-aware). `ANEMemoryUtils.convertToFloat16` produces an aligned fp16 copy of a Float32 array via `vImageConvert_PlanarFtoPlanar16F` (Accelerate's planar fp16 conversion). `strideAwareCopy` walks two arrays with potentially different stride layouts, copying the innermost dimension as contiguous `memcpy` blocks and iterating outer dims with an odometer-style multi-index — with a single-`memcpy` fast path when source and destination strides match exactly.

The file also defines `MLMultiArray.prefetchToNeuralEngine()` (reads the first and last element to trigger a DMA prefetch) and `ZeroCopyDiarizerFeatureProvider` — a feature-provider implementation specific to the diarizer pipeline, with helpers for chaining a single output to a single input or for building a batch of `(waveform, mask)` providers.

## Key Code
- [`Sources/FluidAudio/Shared/ANEMemoryUtils.swift:10`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ANEMemoryUtils.swift#L10) — `aneAlignment = 64` (DMA boundary), `aneTileSize = 16` (matrix tile)
- [`Sources/FluidAudio/Shared/ANEMemoryUtils.swift:23`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ANEMemoryUtils.swift#L23) — `createAlignedArray` — `posix_memalign` + stride-aware sizing + custom `Darwin.free` deallocator
- [`Sources/FluidAudio/Shared/ANEMemoryUtils.swift:68`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ANEMemoryUtils.swift#L68) — deallocator comment: `deallocate()` would trap, must use `Darwin.free`
- [`Sources/FluidAudio/Shared/ANEMemoryUtils.swift:77`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ANEMemoryUtils.swift#L77) — `calculateOptimalStrides` — innermost dim padded to `aneTileSize` multiple
- [`Sources/FluidAudio/Shared/ANEMemoryUtils.swift:100`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ANEMemoryUtils.swift#L100) — `getElementSize` — dtype → byte size lookup
- [`Sources/FluidAudio/Shared/ANEMemoryUtils.swift:116`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ANEMemoryUtils.swift#L116) — `createZeroCopyView` — stride-aware bounds check before `dataPointer.advanced(by:)`
- [`Sources/FluidAudio/Shared/ANEMemoryUtils.swift:148`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ANEMemoryUtils.swift#L148) — `convertToFloat16` — `vImageConvert_PlanarFtoPlanar16F` into pre-aligned fp16 buffer
- [`Sources/FluidAudio/Shared/ANEMemoryUtils.swift:187`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ANEMemoryUtils.swift#L187) — `strideAwareCopy` — fast `memcpy` path + odometer iteration for mismatched strides
- [`Sources/FluidAudio/Shared/ANEMemoryOptimizer.swift:12`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ANEMemoryOptimizer.swift#L12) — buffer pool dict + `NSLock`
- [`Sources/FluidAudio/Shared/ANEMemoryOptimizer.swift:44`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ANEMemoryOptimizer.swift#L44) — `getPooledBuffer` — lock-and-check shape/dtype; allocate-and-store on miss
- [`Sources/FluidAudio/Shared/ANEMemoryOptimizer.swift:86`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ANEMemoryOptimizer.swift#L86) — `createZeroCopyView` — logical-element bounds (note: not stride-aware unlike `ANEMemoryUtils` version)
- [`Sources/FluidAudio/Shared/ANEMemoryOptimizer.swift:116`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ANEMemoryOptimizer.swift#L116) — `optimizedCopy` — runtime-type-dispatched `vDSP_mmov` for `[Float]`/`ArraySlice`/`ContiguousArray`
- [`Sources/FluidAudio/Shared/ANEMemoryOptimizer.swift:186`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ANEMemoryOptimizer.swift#L186) — `prefetchToNeuralEngine` — touch first/last element to trigger DMA
- [`Sources/FluidAudio/Shared/ANEMemoryOptimizer.swift:197`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/ANEMemoryOptimizer.swift#L197) — `ZeroCopyDiarizerFeatureProvider` — diarizer-specific feature provider with `chain` + `batchProvider`

## Edge Cases & Failure Modes
- `posix_memalign` failure → `ANEMemoryError.allocationFailed` (wrapped to `DiarizerError.memoryAllocationFailed` at the optimizer layer — odd cross-module coupling for a shared util). [REVIEW: `ANEMemoryOptimizer.createAlignedArray` rethrows ANE errors as `DiarizerError` regardless of caller]
- `aneTileSize` padding inflates allocations whose innermost dim isn't a multiple of 16 — e.g. shape `[1, 1]` allocates `64` bytes of fp32 (16 elements) instead of 4. Comment says "at least one alignment unit" enforces minimum.
- `optimizedCopy` silently no-ops for non-`.float32` destinations (`guard destination.dataType == .float32 else { return }`).
- `optimizedCopy` `pad: true` only clears destination tail; it does **not** clear bytes preceding the offset.
- `getPooledBuffer` returns the cached buffer **as-is** — callers must not assume zero-initialized contents on a hit. [REVIEW: pool returns non-zeroed buffer on cache hit — caller-side init contract is implicit]
- Pool keying is by user-provided string only — shape/dtype are checked but a mismatched cache entry is replaced silently with a freshly-allocated buffer; old buffer is released.
- `createZeroCopyView` on `ANEMemoryOptimizer` uses logical-element bounds; on `ANEMemoryUtils` uses stride-aware backing-element bounds. The optimizer variant under-rejects views that overrun stride-padded backing memory but stay within logical element count.
- `prefetchToNeuralEngine` reads via `self[0]` and `self[count-1]`, which on float16 arrays involves a subscript-time conversion — not zero-cost.
- `strideAwareCopy` preconditions on `innerStride == 1` for both arrays — any layout with non-contiguous innermost dim traps.
- `convertToFloat16` only handles `.float32 → .float16`; other dtypes throw `unsupportedDataType`.
- `ZeroCopyDiarizerFeatureProvider.chain` returns `nil` if the named output is missing — silent failure that callers must check.

## Performance / Concurrency Notes
- All allocations route through `posix_memalign` — no use of Swift's default allocator. Frees use `Darwin.free` (matching pair).
- `bufferLock` is `NSLock` (not `os_unfair_lock`) and held across allocation in `getPooledBuffer` — under contention the new allocation blocks other readers.
- `vDSP_mmov` is used for both the bulk Float copy and the pad-clear (`vDSP_vclr`); both are SIMD-accelerated.
- `vImageConvert_PlanarFtoPlanar16F` uses hardware fp16 conversion on Apple Silicon.
- `strideAwareCopy` is `O(outerCount)` `memcpy` calls; outerCount = product of all dims except the last.
- The buffer pool is per-instance; the diarizer typically holds one optimizer for its full lifetime and clears the pool only via `clearBufferPool`.

## Test Coverage
`Tests/FluidAudioTests/Shared/ANEMemoryOptimizerTests.swift`, `ANEMemoryUtilsEdgeCaseTests.swift`, and `ANEOptimizerTests.swift` cover allocator, stride calculations, pooled-buffer behaviour, and zero-copy view bounds.

## Changelogs
