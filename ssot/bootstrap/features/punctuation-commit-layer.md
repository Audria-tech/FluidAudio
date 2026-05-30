---
id: punctuation-commit-layer
name: Punctuation Commit Layer
repo: FluidAudio
status: active
linked_features:
  - streaming-asr-manager
  - eou-detector
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Punctuation Commit Layer

## TL;DR
`PunctuationCommitLayer` is the actor that wraps streaming-ASR partial text and segments it into "committed" (finalized) and "ghost" (still-volatile) portions. Text is promoted to committed when it ends with `.`, `!`, or `?`, when an explicit EOU signal arrives, when the caller commits manually, or when a configurable debounce timeout expires mid-sentence.

## Context

## What It Does
The actor owns `committedText`, `ghostText`, a debounce timer Task, and an optional `@Sendable (CommitLayerUpdate) -> Void` callback. `processPartialText(_:)` scans for the last punctuation mark in the new text: everything up to and including that mark (plus surrounding whitespace) is appended to `committedText`; the trailing fragment becomes the new `ghostText`. When no punctuation is found, the whole text becomes ghost and a new debounce timer is started. `processEOU()` commits whatever ghost text remains and emits `CommitReason.endOfUtterance`. `processManualCommit()` (defined later in the file) does the same with `CommitReason.manualCommit`. Every state transition produces a `CommitLayerUpdate` (committed, ghost, total, reason, timestamp) and fires the registered callback.

The contract is that callers wire this in *between* a streaming engine and the UI: `engine.setPartialTranscriptCallback { partial in await commitLayer.processPartialText(partial) }`, then react to committed-text deltas for stable rendering while showing ghost text as a faded-out preview. The actor isolation ensures concurrent partial-text updates and EOU signals are serialized.

## Key Code
- [`PunctuationCommitLayer.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Shared/PunctuationCommitLayer.swift#L5) — `CommitReason` enum.
- [`PunctuationCommitLayer.swift:16`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Shared/PunctuationCommitLayer.swift#L16) — `CommitLayerUpdate` value type.
- [`PunctuationCommitLayer.swift:92`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Shared/PunctuationCommitLayer.swift#L92) — `public actor PunctuationCommitLayer`.
- [`PunctuationCommitLayer.swift:150`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Shared/PunctuationCommitLayer.swift#L150) — `processPartialText(_:)` punctuation split + whitespace preservation.
- [`PunctuationCommitLayer.swift:225`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Shared/PunctuationCommitLayer.swift#L225) — `processEOU()` commits all ghost text.

## Edge Cases & Failure Modes
- Default debounce is 3.0 s — too aggressive in noisy environments; tune via the initializer.
- `commitOnTimeout: false` keeps text in ghost state forever until punctuation or EOU arrives.
- The punctuation set is `Character`-based (`.`, `!`, `?` by default). Multi-byte sentence terminators (e.g. Chinese `。`) need to be added explicitly.
- `lastUpdateTime` is updated on every `processPartialText` but not surfaced — debounce only reschedules itself.
- Concurrent calls into the same actor are safe but block on actor isolation; high-frequency partial updates back up if the callback is slow.

## Test Coverage
- `Tests/FluidAudioTests/ASR/PunctuationCommitLayerTests.swift`.

## Changelogs
