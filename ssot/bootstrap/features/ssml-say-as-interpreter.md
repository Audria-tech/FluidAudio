---
id: ssml-say-as-interpreter
name: SSML say-as Interpreter
repo: FluidAudio
status: active
linked_features: [ssml-processor, ssml-tag-parser, ssml-types]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# SSML say-as Interpreter

## TL;DR
Interprets `<say-as>` tag content based on its `interpret-as` attribute, converting specialized content (cardinals, ordinals, digits, dates, times, telephones, fractions, characters / spell-out) to speakable text. Returns input unchanged for unknown `interpret-as` types.

## Motivation — why it exists

## Context

## What It Does
`SayAsInterpreter` is a `public enum` with one entry point `interpret(content:interpretAs:format?:)` that dispatches on a lowercased / trimmed key:
- `characters`, `spell-out` → `interpretCharacters` (space-joined per-character).
- `cardinal`, `number` → `interpretCardinal` (filter to digits / minus sign, format via `spellOutFormatter`).
- `ordinal` → `interpretOrdinal` (table for 1-19, computed for larger).
- `digits` → `interpretDigits` (each digit as a word).
- `date` → `interpretDate(content:format?:)` (uses cached `digitPattern` to extract components).
- `time` → `interpretTime` (clock-style HH:MM).
- `telephone` → `interpretTelephone` (digit-by-digit).
- `fraction` → `interpretFraction` (numerator/denominator).
- unknown → content unchanged.

Static cached resources: `ordinalWords` table for 1-19, `monthNames` array (1-based via leading empty), `digitPattern` / `minutePattern` / `secondPattern` regexes.

## Key Code
- [`Sources/FluidAudio/TTS/SSML/SayAsInterpreter.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SayAsInterpreter.swift#L5) — `public enum SayAsInterpreter`.
- [`Sources/FluidAudio/TTS/SSML/SayAsInterpreter.swift:10`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SayAsInterpreter.swift#L10) — `ordinalWords` table.
- [`Sources/FluidAudio/TTS/SSML/SayAsInterpreter.swift:40`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SayAsInterpreter.swift#L40) — `interpret(...)` dispatch table.
- [`Sources/FluidAudio/TTS/SSML/SayAsInterpreter.swift:75`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SayAsInterpreter.swift#L75) — `interpretCardinal` (digit filter + spell-out).

## Edge Cases & Failure Modes
- Unknown `interpret-as` returns the original content unchanged (no error surfaced).
- `interpretCardinal` filters input down to digits + minus sign before parse — non-numeric content rejected silently and returned unchanged.
- `interpretOrdinal` only has a hardcoded table for 1-19 — larger ordinals fall through to computed forms; verify "twenty-first" etc.
- `monthNames` uses 1-based indexing with a leading `""` — guards against off-by-one but caller must remember.
- [REVIEW: locale is hardcoded en-US (`spellOutFormatter` uses `en_US_POSIX`); en-only is the documented beta scope.]
- [REVIEW: date / time regex extraction is simple `\d+` matching — does not validate component ranges (month 13, hour 25, etc.).]

## Test Coverage
- `Tests/FluidAudioTests/TTS/SSMLTests.swift` (covers via `SSMLProcessor` and `SayAsInterpreter`).

## Changelogs
