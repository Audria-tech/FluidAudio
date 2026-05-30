---
id: kokoro-custom-lexicon
name: Kokoro Custom Lexicon
repo: FluidAudio
status: active
linked_features: [kokoro-tts-manager, kokoro-pipeline]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Kokoro Custom Lexicon

## TL;DR
A `Sendable` value-type pronunciation override dictionary parsed from a simple `word=phonemes` line format. Entries take precedence over all built-in dictionaries and G2P when injected into Kokoro synthesis via `KokoroTtsManager.setCustomLexicon(_:)`.

## Motivation — why it exists

## Context

## What It Does
`TtsCustomLexicon` stores three lookup tables: case-sensitive `entries`, case-insensitive `lowercaseEntries`, and a private `normalizedEntries` table keyed by a lookup-normalized form (lowercase, letters/digits/apostrophes only, apostrophe variants folded). The init runs a candidate-selection pass per bucket so when several user keys collide (e.g. `"UN"` and `"un"`), the canonical form wins (case-sensitive equals lowercase) and ties break on lexicographic order. `phonemes(for:)` tries exact → lowercase → normalized in order. Static `load(from: URL)` and `parse(_:)` produce a lexicon from `word=phonemes` lines (with `#` comments and blank-line skipping); each phoneme string is split per Swift `Character`, with whitespace runs collapsed to a single `" "` separator token so multi-word expansions like `UN=junˈITᵻd nˈAʃənz` work. Convenience: `empty`, `merged(with:)` (other wins), `count`, `isEmpty`.

## Key Code
- [`Sources/FluidAudio/TTS/Kokoro/TtsCustomLexicon.swift:8`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/TtsCustomLexicon.swift#L8) — `public struct TtsCustomLexicon: Sendable`.
- [`Sources/FluidAudio/TTS/Kokoro/TtsCustomLexicon.swift:21`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/TtsCustomLexicon.swift#L21) — init with bucketed candidate selection.
- [`Sources/FluidAudio/TTS/Kokoro/TtsCustomLexicon.swift:96`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/TtsCustomLexicon.swift#L96) — `parse(_:)` line format with `=` separator and `#` comments.
- [`Sources/FluidAudio/TTS/Kokoro/TtsCustomLexicon.swift:150`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/TtsCustomLexicon.swift#L150) — `phonemes(for:)` exact → lowercase → normalized fallback.
- [`Sources/FluidAudio/TTS/Kokoro/TtsCustomLexicon.swift:182`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/TtsCustomLexicon.swift#L182) — `parsePhonemes` per-Character splitting + whitespace separator.
- [`Sources/FluidAudio/TTS/Kokoro/TtsCustomLexicon.swift:241`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/TtsCustomLexicon.swift#L241) — `merged(with:)` (other takes precedence).

## Edge Cases & Failure Modes
- `parse` throws `TTSError.processingFailed` for missing `=`, empty word, empty phonemes, or `parsePhonemes` returning an empty array.
- Multiple keys differing only in case quietly collapse to one canonical entry per lookup table.
- Per-`Character` phoneme split handles combining diacritics, but pre-split space-separated input is *not* accepted as one token per element — whitespace is reserved for the word separator.
- [REVIEW: word separator `" "` is a special token — downstream `PhonemeMapper` must recognize it; confirm wiring.]

## Test Coverage
- `Tests/FluidAudioTests/TTS/TtsCustomLexiconTests.swift`

## Changelogs
