---
id: audio-mel-spectrogram
name: Audio Mel Spectrogram
repo: FluidAudio
status: active
linked_features: [asr-constants]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Audio Mel Spectrogram

## TL;DR
Native vDSP/Accelerate mel-spectrogram extractor that reproduces NVIDIA NeMo's `AudioToMelSpectrogramPreprocessor` numerically (Slaney mel scale, symmetric Hann window, preemphasis filter, log-magnitude). Replaces the CoreML preprocessor for paths that need exact PyTorch parity or fully native Swift control (Parakeet realtime EOU 120m, Nemotron streaming, others).

## Motivation — why it exists

## Context

## What It Does
At init the extractor (a) builds a Hann window of length `winLength` (symmetric by default, periodic optional — symmetric matches NeMo, periodic matches librosa STFT), (b) builds a `[nMels, numFreqBins]` mel filterbank with Slaney normalization (`norm = 2 / (fRight − fLeft)`) using the librosa Slaney hz↔mel conversion (linear below 1000 Hz with `fSp = 200/3 ≈ 66.67`, logarithmic above with `logStep = log(6.4)/27`), (c) flattens that filterbank to a row-major `[Float]` for vDSP matrix multiply, (d) creates a `vDSP_DFT_zop` setup for the chosen `nFFT`, and (e) pre-allocates every reusable buffer (`realIn`, `imagIn`, `realOut`, `imagOut`, `powerSpec`, `imagSq`, `frame`) so the hot path can run allocation-free.

The hot path is `computeFlatTransposed(audio:lastAudioSample:paddingMode:expectedFrameCount:)`. It optionally seeds the preemphasis filter with `lastAudioSample` (the sample preceding the current chunk; allows seamless streaming) then applies the filter `y[n] = x[n] − preemph × x[n−1]` via `vDSP_vsma` into a pre-padded buffer in one shot. For each STFT frame it (1) clears the FFT input buffer with `vDSP_vclr`, (2) windows `winLength` samples from `paddedAudio` into the centred slot of `frame` using `vDSP_vmul`, (3) runs the real→complex FFT via `vDSP_DFT_Execute`, (4) computes the power spectrum as `real² + imag²` via two `vDSP_vsq` calls + one `vDSP_vadd`, (5) projects onto the mel filterbank with `vDSP_mmul`, and (6) applies the log floor (`.additive` adds `logFloor` before `log`, `.clamped` clips). Output is a flat `[T × nMels]` row-major buffer; an alternative `computeFlat` variant emits `[nMels × T]` for callers that expect that layout (e.g. MLMultiArray shapes `[1, nMels, T]`).

Two padding modes are supported. `.center` zero-pads `nFFT/2` samples on each side of the audio before windowing — matches NeMo's `center=True` / `pad_mode='constant'` behaviour and produces `1 + (paddedCount − winLength) / hopLength` frames. `.prePadded` assumes the caller already supplied left/right padding (used by streaming paths that hand-manage padding to keep frame counts deterministic across chunks). `padTo` ceils the emitted frame count to a multiple of the given divisor — useful when a downstream encoder expects fixed-bucket inputs. `expectedFrameCount` lets streaming callers pin the exact frame span they need, regardless of the formulaic count.

A legacy `compute(audio:)` method returns a `[[[Float]]]` shaped `[1, nMels, T]` — same math but uses non-vDSP `computePowerSpectrum` / `applyMelFilterbank` helpers and is therefore much slower. Debug accessors (`getFilterbank`, `getHannWindow`) are exposed for parity-checking against PyTorch dumps.

## Key Code
- [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:18`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L18) — class doc — full NeMo `nvidia/parakeet_realtime_eou_120m-v1` config
- [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:19`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L19) — `PaddingMode` (`center` matches NeMo; `prePadded` for streaming)
- [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:24`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L24) — `LogFloorMode` (`additive` = `log(x + floor)`, `clamped` = `log(max(x, floor))`)
- [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:59`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L59) — designated init — pre-allocates everything, sets up vDSP DFT
- [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:107`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L107) — `vDSP_DFT_zop_CreateSetup` (`.FORWARD`)
- [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:123`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L123) — `deinit` destroys the vDSP setup (mandatory — leaks otherwise)
- [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:185`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L185) — `computeFlat` — `[nMels × numPaddedFrames]` row-major output
- [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:219`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L219) — preemphasis via `vDSP_vsma` (`x[n] − preemph × x[n−1]` in one vector op)
- [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:299`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L299) — `computeFlatTransposed` — `[T × nMels]` layout
- [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:325`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L325) — `computeFlatTransposed(_:paddingMode:expectedFrameCount:)` — streaming-aware variant
- [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:363`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L363) — preemph-disabled fast path: plain `memcpy` into padded buffer when `preemph == 0`
- [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:459`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L459) — `computePowerSpectrumInPlace` — FFT + `vsq`+`vsq`+`vadd` power computation
- [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:553`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L553) — `createHannWindow(length:periodic:)` — symmetric vs periodic divisor
- [`Sources/FluidAudio/Shared/AudioMelSpectrogram.swift:564`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AudioMelSpectrogram.swift#L564) — `createMelFilterbank` — Slaney mel scale + triangular filters + Slaney `norm = 2/(fRight−fLeft)`

## Edge Cases & Failure Modes
- `compute(audio:)` returns `([[[Float]]](), 0)` if `numFrames ≤ 0` (audio shorter than `winLength`).
- `computeFlatTransposed` returns a single zero-filled frame (`mel: [0…]`, `melLength: 0`, `numFrames: 1`) when audio is too short or empty — callers must check `melLength`, not `numFrames`, to detect "no real data". [REVIEW: zero-data response uses `numFrames: 1` not `0` — callers reading `numFrames` see a phantom frame]
- The class is not `Sendable` — internal mutable buffers (`realIn`, `imagIn`, ...) are reused frame-to-frame. Concurrent calls to `computeFlat*` on the same instance corrupt the FFT buffers. [REVIEW: `AudioMelSpectrogram` mutates reusable buffers without sync — must not be shared across threads]
- `vDSP_DFT_zop_CreateSetup` returns `nil` for unsupported FFT sizes; `computePowerSpectrumInPlace` then no-ops via `guard let setup`. Mel output for the entire call would be all zeros.
- Streaming `lastAudioSample` defaults to 0; if a non-zero previous sample isn't passed at chunk boundaries, the first frame of each chunk gets a wrong preemphasis result that audibly mismatches the offline mel.
- `additive` log mode produces `log(0 + logFloor) ≈ log(2^-24) ≈ -16.6` for empty bins; `clamped` produces the same floor but only when input is below `logFloor`. Switching between modes mid-pipeline changes embeddings.
- `padTo` defaults to `max(1, padTo)` — passing `0` does not produce 0-padding; it silently coerces to 1.
- Slaney `fSp` constant is hard-coded; using the extractor with a non-16 kHz sample rate works but the mel bins drift away from librosa's calibration curve at extreme rates.
- `vImagePixelCount` casts on `audio.count` ultimately bounds the input to ~2³² samples in the convert path — practical limit is far below this but no explicit check.
- `windowOffset = (nFFT − winLength) / 2` assumes `nFFT ≥ winLength`; otherwise the offset goes negative and `vDSP_vmul` would underflow. No assertion.
- Filterbank construction silently produces zero filters if `fMax > sampleRate/2` (Nyquist violation) because mel-to-Hz mapping pushes points beyond the FFT bin range.

## Performance / Concurrency Notes
- All reusable buffers live on the instance; the hot path makes zero heap allocations per frame.
- vDSP is used for every per-frame op: `vclr`, `vmul` (window), `DFT_Execute`, `vsq` ×2, `vadd`, `mmul` (mel projection).
- Preemphasis runs once over the entire chunk via `vDSP_vsma` — `O(N)` vector op rather than a per-sample loop.
- `memcpy` fast path activates when `preemph == 0` — saves the `vsma` pass entirely.
- `compute(audio:)` (legacy) uses naive Swift loops for power spectrum and mel projection — significantly slower; only kept for the alternative output shape and parity tests.
- Concurrent calls require **separate instances** — the FFT setup and reusable buffers are not thread-safe. ASR pipelines typically own one extractor per inference task.

## Test Coverage
No dedicated `AudioMelSpectrogramTests.swift` — parity is validated through ASR end-to-end tests (`Tests/FluidAudioTests/ASR/Parakeet/*`) and CI compile checks. Manual parity checking is done via the `getFilterbank` / `getHannWindow` debug accessors against PyTorch reference dumps.

## Changelogs
