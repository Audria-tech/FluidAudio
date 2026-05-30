---
id: pocket-tts-tokenizer
name: PocketTTS SentencePiece Tokenizer
repo: FluidAudio
status: active
linked_features: [pocket-tts-manager, pocket-tts-pipeline]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# PocketTTS SentencePiece Tokenizer

## TL;DR
Minimal SentencePiece *unigram* tokenizer for PocketTTS. Parses a `.model` protobuf to extract the vocabulary, then Viterbi-decodes input text into subword tokens. Pure Swift, no external SentencePiece dependency.

## Motivation — why it exists

## Context

## What It Does
`SentencePieceTokenizer` is a `Sendable` struct holding the parsed pieces (`[SentencePieceProto.Piece]`), a string-to-id lookup, and `maxPieceLength` for early termination in the Viterbi search. Construction takes the raw protobuf `Data` and calls `SentencePieceProto.parse(_:)` which extracts the unigram vocabulary plus per-piece log-probability scores. The static `spaceMarker = "\u{2581}"` ( ▁ ) is the SentencePiece convention for word boundaries.

`SentencePieceProto` is the zero-dependency parser: it reads the proto's field-by-field wire format just enough to pull the `pieces` (each: piece string + log-prob score). The rest of the proto schema is ignored.

Encoding flow (handled inside the tokenizer): replace spaces with `▁`, run Viterbi DP to maximize cumulative log-prob across overlapping piece segmentations bounded by `maxPieceLength`, return token IDs.

## Key Code
- [`Sources/FluidAudio/TTS/PocketTTS/Tokenizer/SentencePieceTokenizer.swift:7`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/Tokenizer/SentencePieceTokenizer.swift#L7) — `public struct SentencePieceTokenizer: Sendable`.
- [`Sources/FluidAudio/TTS/PocketTTS/Tokenizer/SentencePieceTokenizer.swift:17`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/Tokenizer/SentencePieceTokenizer.swift#L17) — `spaceMarker = "\u{2581}"`.
- [`Sources/FluidAudio/TTS/PocketTTS/Tokenizer/SentencePieceTokenizer.swift:19`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/Tokenizer/SentencePieceTokenizer.swift#L19) — `init(modelData:)` proto parse + lookup build.
- [`Sources/FluidAudio/TTS/PocketTTS/Tokenizer/SentencePieceProto.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/Tokenizer/SentencePieceProto.swift) — minimal protobuf parser.

## Edge Cases & Failure Modes
- Unknown characters (no piece coverage) fall back to fragment + byte fallback per SentencePiece convention.
- The parser ignores schema fields beyond `pieces` — model files with non-default training options may still parse but lose those settings.
- [REVIEW: only "unigram" mode supported — BPE-mode `.model` files would parse but encode incorrectly.]

## Test Coverage
- `Tests/FluidAudioTests/TTS/PocketTTS/SentencePieceProtoTests.swift`

## Changelogs
