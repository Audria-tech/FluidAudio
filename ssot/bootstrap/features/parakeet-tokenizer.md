---
id: parakeet-tokenizer
name: Parakeet Streaming Tokenizer
repo: FluidAudio
status: active
linked_features:
  - eou-detector
  - nemotron-streaming
  - streaming-asr-utils
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Parakeet Streaming Tokenizer

## TL;DR
`Tokenizer` is the minimal SentencePiece-style ID → text decoder used by the Parakeet streaming pipelines (EOU and Nemotron). It loads a `[String: String]` JSON vocabulary (numeric string keys), builds an `[Int: String]` lookup, and concatenates pieces during `decode(ids:)`, replacing the SentencePiece word boundary marker (`\u{2581}`) with a space.

## Context

## What It Does
The initializer reads the JSON, force-casts to `[String: String]` (so format mismatches throw at load time), and populates the int-keyed dictionary. `decode(ids:)` walks the IDs, appends each known piece (silently skipping unknowns), replaces `\u{2581}` with `" "`, and trims surrounding whitespace.

## Key Code
- [`Tokenizer.swift:7`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/Tokenizer.swift#L7) — `init(vocabPath:)` JSON load + dictionary build.
- [`Tokenizer.swift:19`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ASR/Parakeet/Streaming/Tokenizer.swift#L19) — `decode(ids:)` SentencePiece detokenization.

## Edge Cases & Failure Modes
- The JSON parse uses `as!` — any non-`[String: String]` payload crashes rather than throwing. [REVIEW: brittle cast — should be a guarded conditional with a thrown error.]
- Unknown token IDs are silently skipped; output may be shorter than the ID count.
- The class is not `Sendable`; safe because it is only stored inside actor-isolated streaming managers.

## Test Coverage
- `Tests/FluidAudioTests/ASR/Parakeet/Streaming/TokenizerTests.swift`.

## Changelogs
