---
id: audio-stream
name: Audio Stream
repo: FluidAudio
status: active
linked_features: [audio-converter]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Audio Stream

## TL;DR
Sliding-window buffer for real-time audio. Accepts samples (or PCM/CMSampleBuffer) via `write(...)`, emits fixed-duration chunks via a bound callback (sync or async) or a pull-mode `readChunkIfAvailable()`. Configurable chunk duration, skip, alignment strategy (`useMostRecent` / `useFixedSkip`), and startup strategy.

## Motivation — why it exists

## Context

## What It Does
Internally maintains a `ContiguousArray<Float>` of `bufferCapacitySeconds × sampleRate` samples (defaults to `chunkDuration + chunkSkip`), a `writeIndex` (next-write position), a `bufferStartTime` (timestamp of index 0), and a `temporaryChunkSize` (lets `rampUpChunkSize` grow chunks toward `chunkSize`). A concurrent `DispatchQueue` (`FluidAudio.AudioStream.queue`) gates all mutation behind `.barrier` writes; reads use plain `sync`.

`write(...)` accepts `[Float]`, `ArraySlice<Float>`, `ContiguousArray<Float>`, `AVAudioPCMBuffer` (via `AudioConverter.resampleBuffer`), or `CMSampleBuffer` (via `AudioConverter.resampleSampleBuffer`). The float-array path resyncs against the optional `time` argument by appending zeros (if samples should be later than tracked) or rolling back (`rollbackNewest`) if they should be earlier. After each write the loop drains all newly-available chunks to the bound callback.

`readChunkIfAvailable()` checks `writeIndex >= temporaryChunkSize` then extracts a chunk per the chunking strategy — `useMostRecent` reads the trailing window and `forgetOldest` shifts the buffer back by `writeIndex - overlapSize`; `useFixedSkip` reads from the buffer front and shifts by `skipSize`. Startup strategies (`startSilent`, `rampUpChunkSize`, `waitForFullChunk`) parametrise how the first chunk(s) behave before the buffer is fully primed.

## Key Code
- [`Sources/FluidAudio/Shared/AudioStream.swift:33`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioStream.swift#L33) — `hasNewChunk` — `writeIndex >= temporaryChunkSize` under queue.sync
- [`Sources/FluidAudio/Shared/AudioStream.swift:75`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioStream.swift#L75) — `init` — derives `chunkSize`, `skipSize`, buffer capacity; selects startup strategy
- [`Sources/FluidAudio/Shared/AudioStream.swift:139`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioStream.swift#L139) — `bind` (4 overloads: sync `[Float]`, sync generic, async `[Float]`, async generic)
- [`Sources/FluidAudio/Shared/AudioStream.swift:210`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioStream.swift#L210) — `write<C>(from:atTime:)` — type-dispatched fast path for contiguous Float collections
- [`Sources/FluidAudio/Shared/AudioStream.swift:260`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioStream.swift#L260) — `readChunkIfAvailable` — chunking-strategy switch + post-read shift
- [`Sources/FluidAudio/Shared/AudioStream.swift:303`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioStream.swift#L303) — `writeContiguousSamples` — timestamp resync + append + callback drain loop
- [`Sources/FluidAudio/Shared/AudioStream.swift:352`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioStream.swift#L352) — `forgetOldest` — `memmove`-based front-trim
- [`Sources/FluidAudio/Shared/AudioStream.swift:489`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioStream.swift#L489) — strategy enums (`Chunking`, `Startup`) + `AudioStreamError`

## Edge Cases & Failure Modes
- `AudioStreamError.invalidChunkDuration` / `invalidChunkSkip` / `bufferTooSmall` — init-time guards on parameter ranges.
- "Samples may be skipped if the time jumps forward significantly" (docstring): a future `time` triggers `appendZeros` which can push older samples out of the buffer entirely.
- Synchronous callbacks are invoked while holding the queue (via `queue.sync`); a slow callback blocks subsequent writes. Async `bind` overloads dispatch to `Task.detached`.
- `rollbackNewest` can produce negative `writeIndex` mid-call; the helper compensates by advancing `bufferStartTime` and clamping to 0 — but a deeply negative rollback silently discards samples.
- Buffer capacity defaults to `chunkDuration + chunkSkip` seconds; high `bufferCapacitySeconds` reserves significant RAM (`Float × seconds × sampleRate`).
- `useFixedSkip` strategy reads from `buffer.prefix(temporaryChunkSize)` regardless of whether the read range includes samples that haven't been written yet — relies on the `hasNewChunk` guard.
- `read*` and `bind` both populate the chunk arg via `Array(buffer[...])` — full copy on every emission.

## Test Coverage
`Tests/FluidAudioTests/Shared/AudioStreamTests.swift` covers chunking strategies, startup strategies, timestamp resync, and overlap. `RandomAccessCollectionTests.swift` and `ArraySliceTests.swift` cover the input-type dispatch.

## Changelogs
