---
id: audio-source-abstraction
name: Audio Source Abstraction
repo: FluidAudio
status: active
linked_features: [audio-converter]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Audio Source Abstraction

## TL;DR
Three-part abstraction for ASR / Diarizer batched-inference input. `AudioSource` enumerates capture types (`microphone` / `system`). `AudioSampleSource` is a `Sendable` protocol exposing `sampleCount` + a pointer-write API. `AudioSourceFactory` streams an audio file to a temporary RAW file, memory-maps it, and returns a `DiskBackedAudioSampleSource` — letting long files be processed without holding all samples in `[Float]`.

## Motivation — why it exists

## Context

## What It Does
`AudioSampleSource.copySamples(into:offset:count:)` is the read interface — the consumer passes an `UnsafeMutablePointer<Float>` along with an offset and count, and the source memcpys the requested slice into the destination. Two concrete implementations exist: `ArrayAudioSampleSource` wraps a `[Float]` for in-memory use, and `DiskBackedAudioSampleSource` wraps `Data(contentsOf:options:[.mappedIfSafe])` so the OS handles paging.

`AudioSourceFactory.makeDiskBackedSource(from:targetSampleRate:)` is the streaming converter: it opens the source file, builds an `AVAudioConverter` to mono Float32 at the requested rate, reads input in 16 384-frame chunks via `AVAudioConverter.convert(to:error:withInputFrom:)`, and writes each converted output buffer to a unique `fluidaudio-streaming-<uuid>.raw` temp file via `FileHandle.write(contentsOf:)`. The temp file is then mapped read-only via `Data(contentsOf:options:[.mappedIfSafe])` and handed to a `DiskBackedAudioSampleSource`. `cleanup()` deletes the temp file. The factory returns the source plus elapsed load duration.

## Key Code
- [`Sources/FluidAudio/Shared/AudioSource.swift:3`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioSource.swift#L3) — `AudioSource` enum (`microphone`, `system`) — `Sendable`
- [`Sources/FluidAudio/Shared/AudioSampleSource.swift:3`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioSampleSource.swift#L3) — `AudioSampleSource` protocol — pointer-write API, `Sendable`
- [`Sources/FluidAudio/Shared/AudioSampleSource.swift:12`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioSampleSource.swift#L12) — `ArrayAudioSampleSource` — in-memory `[Float]` wrapper
- [`Sources/FluidAudio/Shared/AudioSampleSource.swift:42`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioSampleSource.swift#L42) — `DiskBackedAudioSampleSource` — mmap-backed source
- [`Sources/FluidAudio/Shared/AudioSampleSource.swift:74`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioSampleSource.swift#L74) — `cleanup()` — best-effort temp file deletion
- [`Sources/FluidAudio/Shared/AudioSourceFactory.swift:11`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioSourceFactory.swift#L11) — `makeDiskBackedSource(from:targetSampleRate:)` — stream-and-mmap factory
- [`Sources/FluidAudio/Shared/AudioSourceFactory.swift:94`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioSourceFactory.swift#L94) — `streamConvert` — input-block-driven `AVAudioConverter` loop with `OSAllocatedUnfairLock` for completion/error flags

## Edge Cases & Failure Modes
- `AudioSourceError.processingFailed(String)` is the single error case — wraps both buffer allocation failures and per-frame conversion failures.
- `nonisolated(unsafe) let capturedInputBuffer` is captured by the `AVAudioConverterInputBlock` — safe because the block runs synchronously inside `convert(...)`, but the annotation is required by Swift 6. [REVIEW: `AudioSourceFactory.streamConvert` uses `nonisolated(unsafe)` for the input buffer — correctness depends on the synchronous-callback contract holding]
- Temp file location is `FileManager.default.temporaryDirectory` + uuid — cleanup is best-effort and ignores errors; long-running processes may leak temp files if `cleanup()` is forgotten.
- `Data(contentsOf:options:[.mappedIfSafe])` falls back to a regular read if mmap isn't safe (e.g. network volumes) — the source still works but loses the streaming-memory benefit silently.
- A sample-count mismatch between the temp file's mapped data length and the tracked `totalSamples` only logs a warning — no truncation or refusal.
- `copySamples` on either implementation silently returns when `count == 0`, `sampleCount == 0`, or the offset is out of bounds; partial reads (`available < count`) leave the tail of the destination unwritten — caller must zero or check return.

## Test Coverage
No direct unit tests for `AudioSourceFactory` or `DiskBackedAudioSampleSource`; covered indirectly through ASR file-based integration tests.

## Changelogs
