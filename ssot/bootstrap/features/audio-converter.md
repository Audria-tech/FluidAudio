---
id: audio-converter
name: Audio Converter
repo: FluidAudio
status: active
linked_features: [audio-stream]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Audio Converter

## TL;DR
Converts arbitrary `AVAudioPCMBuffer`s, `CMSampleBuffer`s, audio files, and raw `[Float]` arrays into mono Float32 at the target sample rate (default 16 kHz) — the canonical format every FluidAudio model consumes. Uses `AVAudioConverter` with mastering-quality resampling for ≤2 channels, falls back to a hand-rolled mixer + linear interpolation for higher channel counts.

## Motivation — why it exists

## Context

## What It Does
The converter is `Sendable` and immutable; construction picks the target format (16 kHz / mono / Float32 / non-interleaved by default, overridable to any `AVAudioFormat` or any sample rate). All entry points return a `[Float]` ready for the model frontends.

Four input shapes are supported. `resample(_:from:)` takes a mono Float32 array at an arbitrary sample rate and resamples it via an `AVAudioConverter` built around mono Float32 input/output formats. `resampleBuffer(_:)` takes an `AVAudioPCMBuffer`, short-circuits if it already matches the target format (`extractFloatArray` via `vDSP_mmov`), otherwise delegates to `convertBuffer`. `resampleAudioFile(_:)` streams a file in `chunkSize ≈ sampleRate` frames, extracts each chunk to mono Float32 via an intermediate `AVAudioConverter`, accumulates, then runs the rate conversion at the end. `resampleSampleBuffer(_:)` extracts an `AVAudioPCMBuffer` from a `CMSampleBuffer` via `CMSampleBufferCopyPCMDataIntoAudioBufferList` (with `extractAVAudioPCMBuffer` factored out as a public helper for callers that need the buffer rather than samples).

`convertBuffer` is the core path. For 1–2 channels it builds an `AVAudioConverter` between input and target formats, sets `sampleRateConverterAlgorithm = AVSampleRateConverterAlgorithm_Mastering` and `quality = .max`, estimates first-pass output capacity by sample-rate ratio, then drains the converter in a loop until `endOfStream`. The input block uses an `OSAllocatedUnfairLock<Bool>` (not a captured `var`) to satisfy Swift 6's isolation rules while still tracking whether the input has been delivered once. Each output buffer is flattened via `extractFloatArray` and appended to the aggregate.

For >2 channels (e.g. Safari speaker mode, multi-channel hardware) `linearResample` mixes channels down to mono by averaging, then linear-interpolates at the new sample rate. The comment is explicit that accuracy is worse than `AVAudioConverter`'s mastering algorithm — this path exists because `AVAudioConverter` has known limitations above 2 channels.

The file also publishes `AudioWAV.data(from:sampleRate:)` — a small WAV-encoder used by both TTS and ASR. It normalises the Float samples by their absolute max, clips to ±1, converts to little-endian 16-bit PCM, and builds a single-chunk WAV (RIFF/fmt/data) byte-by-byte.

## Key Code
- [`Sources/FluidAudio/Shared/AudioConverter.swift:14`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L14) — `final public class AudioConverter: Sendable` — immutable state, no mutation after init
- [`Sources/FluidAudio/Shared/AudioConverter.swift:23`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L23) — default-format init: 16 kHz / mono / Float32 / non-interleaved
- [`Sources/FluidAudio/Shared/AudioConverter.swift:60`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L60) — `resample(_:from:)` — `[Float]`-in, `[Float]`-out resampler
- [`Sources/FluidAudio/Shared/AudioConverter.swift:77`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L77) — `resampleBuffer` — fast path via `isTargetFormat` + `extractFloatArray`
- [`Sources/FluidAudio/Shared/AudioConverter.swift:91`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L91) — `resampleAudioFile(_:)` — chunked read loop, accumulates mono samples
- [`Sources/FluidAudio/Shared/AudioConverter.swift:171`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L171) — `extractMonoFloat32` — fast path for mono-fp32-non-interleaved; channel-mix via `AVAudioConverter` otherwise
- [`Sources/FluidAudio/Shared/AudioConverter.swift:236`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L236) — `resampleSampleBuffer` (and `extractAVAudioPCMBuffer` helper) — handles `CMSampleBuffer` (ScreenCaptureKit / AVCaptureSession)
- [`Sources/FluidAudio/Shared/AudioConverter.swift:283`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L283) — `convertBuffer` — channel-count switch + drain-to-EOS loop
- [`Sources/FluidAudio/Shared/AudioConverter.swift:312`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L312) — `OSAllocatedUnfairLock<Bool>` input-delivered flag — Swift 6 isolation-safe replacement for `var provided`
- [`Sources/FluidAudio/Shared/AudioConverter.swift:357`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L357) — `configure(converter:)` sets `Mastering` algorithm + `quality = .max`
- [`Sources/FluidAudio/Shared/AudioConverter.swift:372`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L372) — `linearResample` — >2-channel path with average-mix + linear interpolation
- [`Sources/FluidAudio/Shared/AudioConverter.swift:429`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L429) — `extractFloatArray` — `vDSP_mmov` into `unsafeUninitializedCapacity` Array
- [`Sources/FluidAudio/Shared/AudioConverter.swift:458`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioConverter.swift#L458) — `AudioWAV.data(from:sampleRate:)` — float-to-16bit PCM + manual RIFF header

## Edge Cases & Failure Modes
- `AudioConverterError.failedToCreateConverter` — input format unsupported by `AVAudioConverter` (rare; usually only when the source format is malformed).
- `AudioConverterError.sampleBufferFormatMissing` — `CMSampleBuffer` has no audio format description; typically means the caller passed a video frame by mistake.
- `AudioConverterError.failedToCreateSourceFormat` — `AVAudioFormat(streamDescription:)` rejected the sample buffer's basic description.
- `AudioConverterError.sampleBufferCopyFailed(OSStatus)` — `CMSampleBufferCopyPCMDataIntoAudioBufferList` returned non-zero. OSStatus is opaque.
- `AudioConverterError.conversionFailed(Error?)` — the underlying `AVAudioConverter.convert` returned `.error`.
- `extractFloatArray` asserts `channelCount == 1` — in release builds an interleaved or multi-channel buffer would read only channel 0 and silently produce wrong audio.
- `linearResample` uses naive averaging for downmix (all channels weighted equally), not standard `Lt/Rt` matrix downmix — surround-encoded stereo signals will lose their decoder cues.
- `linearResample` linear interpolation has audible aliasing on high-rate downsamples; the comment acknowledges this trade-off.
- `AudioWAV` normalises by absolute max **per buffer** — concatenating multiple `data(from:)` outputs at different gains will produce loudness discontinuities.
- `resampleAudioFile(_:)` reads in `chunkSize ≈ sampleRate` (1 second) frames; very long files allocate `monoSamples` with `reserveCapacity(Int(audioFile.length))` upfront (potentially many MB).
- A new `AVAudioConverter` is constructed per buffer in `convertBuffer` — the doc comment notes the converter is "stateless"; this is deliberate but adds per-call setup cost.
- The fast-path equality check `isTargetFormat` compares sample rate, channel count, common format, and interleaved flag — but not bit depth or channel layout, so a format that differs only in channel layout would skip conversion incorrectly. [REVIEW: `isTargetFormat` doesn't compare channelLayout — non-default layouts could mis-route through the fast path]
- For >2-channel `convertBuffer` the fallback to `linearResample` runs in process, not via `AVAudioConverter`, so the `Mastering` algorithm quality settings do not apply.

## Performance / Concurrency Notes
- Converter is `Sendable` and stateless; safe to share across threads, though each `convert*` call constructs an `AVAudioConverter` so contention is at AVFoundation's layer.
- `extractFloatArray` uses `[Float](unsafeUninitializedCapacity:)` + `vDSP_mmov` — single allocation, single memcpy-equivalent.
- `convertBuffer` first-pass output buffer is sized by sample-rate ratio (ceiling); drain loop uses 4096-frame buffers until EOS. Aggregation does `reserveCapacity(estimatedOutputFrames)` to avoid re-allocs in the common case.
- `OSAllocatedUnfairLock` is used both in `convertBuffer` and `extractMonoFloat32` to track input-delivered state — input block runs synchronously inside `AVAudioConverter.convert`, but the closure is `@Sendable` so it cannot capture a mutable `var`.
- WAV builder allocates one `Data` per output sample (`pcm.append`) — for very long audio this is the slow path; should be batched via `withUnsafeBytes` of a contiguous Int16 buffer if it ever becomes hot.

## Test Coverage
`Tests/FluidAudioTests/Shared/AudioConverterTests.swift` covers format detection, mono extraction, sample-rate conversion, and the >2-channel `linearResample` path. `AudioStreamTests.swift` exercises the converter indirectly via `AudioStream.write(from:AVAudioPCMBuffer)`.

## Changelogs
