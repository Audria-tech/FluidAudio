---
id: audio-validation
name: Audio Validation
repo: FluidAudio
status: active
linked_features: [diarizer-manager, diarizer-types]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Audio Validation

## TL;DR
`AudioValidation` is an internal struct with two checks: `validateAudio<C: Collection<Float>>` returns an `AudioValidationResult` listing any issues (too short, empty, too quiet), and `validateEmbedding` enforces non-empty, all-finite, magnitude > 0.1.

## Motivation — why it exists

## Context

## What It Does
`validateAudio` accepts any `Collection<Float>` and computes `duration = count / 16000`, appending issues for `duration < 1.0s` ("Audio too short"), empty input, and `rmsEnergy < 0.01` ("Audio too quiet or silent"). RMS energy is computed sequentially via `samples.reduce(0)` over squared values then `sqrt(squaredSum / count)`. `validateEmbedding` checks non-empty + `allSatisfy { $0.isFinite }` + L2 magnitude > 0.1 (sqrt of sum of squares, computed via map+reduce — not vDSP).

## Key Code
- [`AudioValidation.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Segmentation/AudioValidation.swift#L5) — `validateAudio<C>` builds the issues array and returns `AudioValidationResult(isValid: issues.isEmpty, ...)`.
- [`AudioValidation.swift:31`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Segmentation/AudioValidation.swift#L31) — `validateEmbedding` magnitude gate at 0.1.
- [`AudioValidation.swift:42`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Segmentation/AudioValidation.swift#L42) — `calculateRMSEnergy<C>` — naïve scalar fold, no vDSP.

## Edge Cases & Failure Modes
- The 16 kHz sample-rate divisor is hard-coded in `validateAudio` — passing audio at any other rate produces incorrect `durationSeconds`.
- `calculateRMSEnergy` is O(n) scalar; consider vDSP_svesq for large inputs. [REVIEW: contrast with `SpeakerOperations.validateEmbedding` which does use vDSP.]
- `validateEmbedding` here duplicates `SpeakerUtilities.validateEmbedding` but uses scalar arithmetic instead of vDSP. [REVIEW: two implementations of the same check in different files.]
- Empty audio triggers both "Audio too short" *and* "No audio data" issues.

## Test Coverage
- No dedicated tests; exercised through `Tests/FluidAudioTests/Diarizer/SpeakerEnrollmentTests.swift` via `DiarizerManager.validateAudio` / `validateEmbedding`.

## Changelogs
