---
id: mlarray-cache
name: MLArray Cache
repo: FluidAudio
status: active
linked_features: [ane-memory-optimizer]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# MLArray Cache

## TL;DR
`actor`-isolated pool of `MLMultiArray` instances keyed by `(shape, dataType)`, intended to amortise the cost of ANE-aligned allocation across repeated prediction calls. Exposed as a single process-wide `sharedMLArrayCache`.

## Motivation — why it exists

## Context

## What It Does
Cache entries are `[MLMultiArray]` stacks keyed by `CacheKey(shape: [Int], dataType: MLMultiArrayDataType)`. `getArray` pops the back of the stack if non-empty, otherwise allocates a fresh array via `ANEMemoryUtils.createAlignedArray`. `returnArray` pushes the array back if the per-key count is below `maxCacheSize / max(cache.count, 1)`, calling `resetData(to: 0)` on it first to give the next caller a zero-initialised buffer. `prewarm` allocates up to `min(5, maxCacheSize / max(shapes.count, 1))` arrays per requested shape. `clear` empties everything.

## Key Code
- [`Sources/FluidAudio/Shared/MLArrayCache.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/MLArrayCache.swift#L5) — `actor MLArrayCache` — isolation guarantees thread safety
- [`Sources/FluidAudio/Shared/MLArrayCache.swift:19`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/MLArrayCache.swift#L19) — `getArray` — pop-or-allocate via `ANEMemoryUtils.createAlignedArray`
- [`Sources/FluidAudio/Shared/MLArrayCache.swift:35`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/MLArrayCache.swift#L35) — `returnArray` — `resetData(to: 0)` before re-pooling, bounded per-key
- [`Sources/FluidAudio/Shared/MLArrayCache.swift:78`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/MLArrayCache.swift#L78) — `let sharedMLArrayCache = MLArrayCache()` — file-private global instance

## Edge Cases & Failure Modes
- `resetData(to:)` is defined as an extension on `MLMultiArray` in `Sources/FluidAudio/ASR/Parakeet/SlidingWindow/TDT/Decoder/TdtDecoderState.swift` — not in `Shared/`. The cache silently depends on that extension being in scope at the compile unit. [REVIEW: `resetData(to:)` lives in TDT decoder file but is consumed by the shared cache — fragile cross-module dependency]
- Per-key size cap `maxCacheSize / max(cache.count, 1)` shrinks as the number of distinct shapes grows; high shape diversity defeats the cache.
- `prewarm` swallows allocation errors silently — failed shapes simply have an empty pool.
- `returnArray` ignores arrays whose per-key bucket is full — the dropped array is freed by ARC immediately.
- No internal use sites of `sharedMLArrayCache` outside this file as of this SHA — the cache may be effectively dormant. [REVIEW: `sharedMLArrayCache` has no callers in the repo at this SHA — verify if it's still wired into any hot path]

## Test Coverage
`Tests/FluidAudioTests/Shared/MLArrayCacheTests.swift` covers basic get/return and clear behaviour.

## Changelogs
