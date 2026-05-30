---
id: whisper-mel-spectrogram
name: Whisper Mel Spectrogram (Qwen3-ASR)
repo: FluidAudio
status: active
linked_features:
  - qwen3-asr-manager
  - qwen3-asr-config
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Whisper Mel Spectrogram (Qwen3-ASR)

## TL;DR
`WhisperMelSpectrogram` is the Qwen3-ASR mel-spectrogram extractor that matches HuggingFace's `WhisperFeatureExtractor` bit-for-bit: 400-sample periodic Hann window, 160-sample hop, 128 HTK-Slaney mel bins, log10 output, dynamic-range compression + per-utterance normalization. Because Apple's vDSP FFT does not support length 400, the DFT is computed via precomputed cos/sin tables using vDSP vector operations — slower than a power-of-two FFT but bit-exact against `torch.stft(n_fft=400)`.

## Context

## What It Does
Init precomputes a 400-sample periodic Hann window (`np.hanning(401)[:-1]`), the HTK-Slaney mel filterbank `[128, 201]` flattened row-major, and the DFT cos/sin tables: `cos(-2π k n / 400)` and `sin(-2π k n / 400)` for `k in [0, 201)`, `n in [0, 400)`. Per-frame compute multiplies the windowed frame by `dftCos`/`dftSin` via `vDSP_dotpr` to get the real/imag parts, sums of squares give power, mel-filter projection gives the 128 mel-energy values, log10 + dynamic range compression + normalization finishes the frame.

The class explicitly documents that it is **not** `Sendable` and **not** thread-safe because of reusable buffers (`windowedFrame`, `realPart`, `imagPart`, `powerSpec`, `melFrame`). Each thread/task must use its own instance — `Qwen3AsrManager` holds one as a let-bound stored property and relies on actor isolation for serialization.

## Key Code
- [`WhisperMelSpectrogram.swift:22`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/WhisperMelSpectrogram.swift#L22) — class declaration with thread-safety warning.
- [`WhisperMelSpectrogram.swift:28`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/WhisperMelSpectrogram.swift#L28) — config constants (`nFFT: 400`, `hopLength: 160`).
- [`WhisperMelSpectrogram.swift:43`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/WhisperMelSpectrogram.swift#L43) — DFT cos/sin tables (row-major `[numFreqBins * nFFT]`).
- [`WhisperMelSpectrogram.swift:55`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Qwen3/WhisperMelSpectrogram.swift#L55) — reusable buffers.

## Edge Cases & Failure Modes
- Not `Sendable` — concurrent calls from multiple actors will race on reusable buffers.
- Audio shorter than `nFFT - hopLength = 240` samples produces zero frames (the manager throws `Qwen3AsrError.generationFailed`).
- DFT via dot-products is `O(N * K)` per frame vs `O(N log N)` for FFT — measurably slower for long audio but bit-exact against PyTorch (worth it for parity).
- `nFFT = 400` is documented as the reason for the custom DFT — Apple's vDSP requires `f * 2^n` with `f in {1, 3, 5, 15}`, which excludes 400.

## Test Coverage
- No dedicated test for `WhisperMelSpectrogram` itself; parity is verified indirectly through `Qwen3AsrManager` test paths.

## Changelogs
