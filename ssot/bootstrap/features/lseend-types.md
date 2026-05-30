---
id: lseend-types
name: LS-EEND Types
repo: FluidAudio
status: active
linked_features: [lseend-diarizer, lseend-inference, lseend-preprocessor]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# LS-EEND Types

## TL;DR
Value types for the LS-EEND pipeline: `LSEENDMetadata` (model config loaded from the CoreML `creatorDefinedKey`), `LSEENDState` (six `MLMultiArray` recurrent-state buffers, `~Copyable`), and `LSEENDError`. Also re-exports `LSEENDVariant` and `LSEENDStepSize` from `ModelNames.LSEEND`.

## Motivation — why it exists

## Context

## What It Does
`LSEENDMetadata` is `Codable` and stores: `chunkSize`, `frameDurationSeconds`, `maxSpeakers`, `sampleRate`, `maxNspks`, `hopLength`, `winLength`, `nMels`, `contextSize`, `subsampling`, `convDelay`, plus model structure fields `nUnits`, `nHeads`, `encNLayers`, `decNLayers`, `convKernelSize`. Two computed derivations: `headDim = nUnits / nHeads`, `melFrames = (chunkSize - 1) * subsampling + 2 * contextSize + 1`, and `nFFT = 1 << (Int.bitWidth - (winLength - 1).leadingZeroBitCount)` (next power of 2 ≥ winLength).

`LSEENDState` is `~Copyable` (move-only) and owns six `MLMultiArray`s: `encRetKv` `[Lenc, 1, H, hd, hd]`, `encRetScale` `[Lenc, 1]`, `encConvCache` `[Lenc, 1, K, D]`, `cnnWindow` `[1, D, Kcnn]` where `Kcnn = 2*convDelay`, `decRetKv` `[Ldec, nSpk, H, hd, hd]`, `decRetScale` `[Ldec, 1]`. All allocated via `ANEMemoryUtils.createAlignedArray` for ANE-aligned access. `copy()` clones each buffer via `ANEMemoryUtils.strideAwareCopy`; `copy(to:)` copies the contents of self into a borrowed destination (no allocation). `reset()` memsets every buffer to zero.

`LSEENDError` covers `initializationFailed(String)`, `inferenceFailed(String)`, `invalidInputSize(String)`, `notInitialized` (no localized description on this last case).

## Key Code
- [`LSEENDTypes.swift:10`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDTypes.swift#L10) — `LSEENDMetadata` declaration.
- [`LSEENDTypes.swift:53`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDTypes.swift#L53) — `melFrames` derivation formula.
- [`LSEENDTypes.swift:62`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDTypes.swift#L62) — `~Copyable struct LSEENDState` (move-only ownership).
- [`LSEENDTypes.swift:112`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/LS-EEND/LSEENDTypes.swift#L112) — `copy()` allocates fresh ANE-aligned buffers and stride-aware-copies into them.

## Edge Cases & Failure Modes
- `~Copyable` means callers must explicitly `consume` or move the state — the compiler enforces this; misuse is a compile error, not a runtime bug.
- `reset()` is `nonmutating` despite zeroing buffer contents because the buffers themselves are heap-allocated `MLMultiArray`s. This is intentional but worth flagging — `reset` does *not* require ownership of `self`. [REVIEW: marks `LSEENDState` as effectively reference-typed at the buffer level despite the `~Copyable` value-type structure.]
- `nFFT` computation uses a clever bit-twiddle for the smallest power of 2 ≥ `winLength`; correct for `winLength >= 1` but undefined for `winLength == 0` (which would compute `1 << Int.bitWidth` = UB).
- `LSEENDError.notInitialized` has no `errorDescription` case — `localizedDescription` will fall back to the default Swift error description.

## Test Coverage
- Tests use `LSEENDState` through `LSEENDFeatureProvider` indirectly.

## Changelogs
