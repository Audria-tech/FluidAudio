---
id: sliding-window-asr-manager
name: Sliding Window ASR Manager
repo: FluidAudio
status: active
linked_features:
  - sliding-window-asr-session
  - tdt-decoder
  - streaming-asr-manager
  - custom-vocabulary-rescorer
  - custom-vocabulary-word-spotting
  - asr-types
  - audio-buffer
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Sliding Window ASR Manager

## TL;DR
`SlidingWindowAsrManager` is the high-level public actor that turns the offline Parakeet TDT encoder into a pseudo-streaming transcriber by feeding overlapping audio windows (left context + chunk + right lookahead) through `AsrManager` and accumulating tokens with a two-tier (volatile / confirmed) transcript. It is the recommended path for real-time microphone or system audio capture on Parakeet TDT models and supports CTC-based vocabulary boosting on confirmed text.

## Motivation — why it exists

## Context

## What It Does
The manager exposes an `AsyncStream<AVAudioPCMBuffer>` input. Every call to `streamAudio(_:)` yields a buffer that the recognizer task resamples to 16 kHz mono via `AudioConverter`, appends to an absolute-indexed `sampleBuffer`, and then slices into windows of the form `[leftContext, chunk, rightContext]`. Each window is sent through `AsrManager.transcribeChunk(...)` together with a per-session `TdtDecoderState`, accumulated `previousTokens`, and an `isLastChunk` flag. Output frames are re-aligned to a global audio timeline by `applyGlobalFrameOffset`, which adds `windowStartSample / samplesPerEncoderFrame` to every chunk-local timestamp.

Two transcript tiers are maintained in parallel: `volatileTranscript` (the latest chunk's text — may change) and `confirmedTranscript` (text promoted out of volatile once it both clears `confirmationThreshold` confidence and has at least `minContextForConfirmation` seconds of audio behind it). On every chunk the manager yields a `SlidingWindowTranscriptionUpdate` carrying the new text, the `isConfirmed` flag, raw token IDs, and re-aligned token timings. When `finish()` is called the manager either rebuilds the full transcript from `accumulatedTokens` via `AsrManager.convertTokensToText` (lossless dedup path) or concatenates the two tiers when vocabulary rescoring has already mutated text in place.

Vocabulary boosting (optional) is configured via `configureVocabularyBoosting(vocabulary:ctcModels:config:)`, which constructs a `CtcKeywordSpotter` and a `VocabularyRescorer`. When a chunk is being promoted to confirmed, the manager re-runs CTC inference over the same window, then calls `VocabularyRescorer.ctcTokenRescore(...)` with chunk-local token timings and `ContextBiasingConstants`-derived `cbw` / `minSimilarity`. Replacements that pass the acoustic-evidence gate mutate the displayed `ASRResult` via `withRescoring(...)` and are surfaced in the update as `detected` / `applied` term lists.

Error recovery is wired through `attemptErrorRecovery`: model-processing errors trigger `resetDecoderForRecovery()` which rebuilds the `TdtDecoderState` with the correct layer count from `AsrManager.decoderLayerCount`. Cancellation is cooperative — `cancel()` finishes the input stream, cancels the recognizer Task, and finishes the update continuation.

## Key Code
- [`SlidingWindowAsrManager.swift:10`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift#L10) — actor declaration with `AudioConverter`, `AsyncStream` input, decoder state, and vocabulary boosting fields.
- [`SlidingWindowAsrManager.swift:86`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift#L86) — `configureVocabularyBoosting` wires up `CtcKeywordSpotter` + `VocabularyRescorer` with vocab-size-aware config.
- [`SlidingWindowAsrManager.swift:156`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift#L156) — `startStreaming(source:)` creates the per-session `TdtDecoderState` (sized to `decoderLayerCount`) and launches the recognizer Task.
- [`SlidingWindowAsrManager.swift:319`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift#L319) — `appendSamplesAndProcess` slides the window forward by `chunk` samples while preserving `left` for context.
- [`SlidingWindowAsrManager.swift:358`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift#L358) — `flushRemaining` drains the trailing partial chunk at end-of-stream without right context.
- [`SlidingWindowAsrManager.swift:399`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift#L399) — `processWindow` calls `AsrManager.transcribeChunk` and yields the update.
- [`SlidingWindowAsrManager.swift:520`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift#L520) — `updateTranscriptionState` promotion rules (high confidence + sufficient context).
- [`SlidingWindowAsrManager.swift:623`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift#L623) — `applyGlobalFrameOffset` re-aligns chunk-local timestamps onto the absolute audio timeline.
- [`SlidingWindowAsrManager.swift:678`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManager.swift#L678) — `SlidingWindowAsrConfig` (default, `.streaming`, `custom(...)`) and sample-count helpers.

## Edge Cases & Failure Modes
- `finish()` warning: when `vocabBoostingEnabled`, the code intentionally avoids token-based reconstruction so prior rescoring is preserved — token-based decode would undo CTC corrections.
- `startStreaming` throws `ASRError.notInitialized` if `loadModels()` was never called.
- Buffer trimming uses absolute indexing (`bufferStartIndex`) so partial advances stay aligned even after many windows.
- `flushRemaining` produces a final shorter window with `isLastChunk: true` so the TDT decoder can drain pending tokens.
- `attemptErrorRecovery` discards mid-flight state and rebuilds `TdtDecoderState` after `.modelProcessingFailed` only — `.modelsNotLoaded` and `.invalidConfiguration` are non-recoverable.
- `SlidingWindowAsrError.bufferOverflow` is logged but not raised here; backpressure is delegated to the upstream capture stack.
- Vocabulary rescoring is skipped silently when the spotter produces empty log-probs, when token timings are empty, or when the rescorer throws (warning logged).
- [REVIEW: `_ = sampleRate` is read but unused inside `appendSamplesAndProcess`/`flushRemaining` — looks like leftover from refactor.]

## Performance / Concurrency Notes
- The actor isolates all mutable state (`sampleBuffer`, `accumulatedTokens`, `volatileTranscript`, `confirmedTranscript`, decoder state). Concurrent `streamAudio(_:)` calls are safe because `AsyncStream.Continuation.yield` is `Sendable`.
- `AudioConverter` is held as a stored let inside the actor; each conversion is stateless (no flush at end-of-stream).
- Window slicing is O(chunk) per advance with periodic `removeFirst(dropCount)` trims to keep the live buffer bounded by `~leftContext` samples.
- Vocabulary rescoring runs a second CTC pass per confirmed chunk — measurable cost when enabled, especially on large vocabularies (see `ContextBiasingConstants.rescorerConfig(forVocabSize:)`).

## Test Coverage
- `Tests/FluidAudioTests/ASR/Parakeet/SlidingWindow/SlidingWindowAsrManagerTests.swift` — public API surface, config variants, lifecycle.
- `Tests/FluidAudioTests/ASR/Parakeet/SlidingWindow/SlidingWindowAsrSessionTests.swift` — session-level coordination.

## Changelogs
