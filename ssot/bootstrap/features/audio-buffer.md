---
id: audio-buffer
name: Audio Buffer (Actor-Isolated Ring Buffer)
repo: FluidAudio
status: active
linked_features:
  - sliding-window-asr-manager
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Audio Buffer (Actor-Isolated Ring Buffer)

## TL;DR
`AudioBuffer` is the internal actor-isolated circular ring buffer used by the long-form chunked TDT pipeline. It supports overflow-by-discarding-oldest, full and partial chunk retrieval, and keeps a short trace of the last 10 processed chunk offsets for debugging.

## Context

## What It Does
The buffer is a fixed-size `[Float]` with `readPosition`, `writePosition`, and `count` tracking. `append(_:)` throws `AudioBufferError.bufferOverflow` if a single batch is larger than capacity; otherwise it makes room by advancing the read position, writes the new samples, and — when overflow happened — repositions read to the start of the freshly-written region (so the most recent audio is what gets read next). `getChunk(size:)` returns `nil` until the requested amount is available; `getChunkWithPartial(requestedSize:allowPartial:)` falls back to whatever is available when `allowPartial` is true. `peekAvailable()` returns the live contents without advancing read, and `utilization()` returns the fill percentage.

## Key Code
- [`AudioBuffer.swift:17`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/AudioBuffer.swift#L17) — internal `actor AudioBuffer` declaration.
- [`AudioBuffer.swift:40`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/AudioBuffer.swift#L40) — `append(_:)` overflow handling.
- [`AudioBuffer.swift:73`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/AudioBuffer.swift#L73) — `getChunk(size:)` returns `nil` if not enough samples.
- [`AudioBuffer.swift:122`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/AudioBuffer.swift#L122) — `getChunkWithPartial(requestedSize:allowPartial:)` partial drain at end-of-stream.

## Edge Cases & Failure Modes
- A single appended batch larger than `capacity` always throws; smaller batches that *combined* exceed capacity silently discard oldest and warn via the logger.
- The post-overflow reposition can effectively discard *all* prior samples — the comment is explicit that the goal is to prioritize the freshly written region.
- `processedChunks` history is capped at 10 entries with `removeFirst()` on overflow.
- `AudioBuffer` is `internal`, not part of the public API; callers cannot directly instantiate it.

## Test Coverage
- No dedicated test file; behavior is exercised by `Tests/FluidAudioTests/ASR/Parakeet/SlidingWindow/TDT/ChunkProcessorTests.swift` and adjacent chunking tests.

## Changelogs
