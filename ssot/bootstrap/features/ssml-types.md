---
id: ssml-types
name: SSML Types and Shared Utilities
repo: FluidAudio
status: active
linked_features: [ssml-processor, ssml-tag-parser, kokoro-pipeline]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# SSML Types and Shared Utilities

## TL;DR
Public `SSMLProcessingResult`, internal `SSMLParsedTag`, and the shared word-boundary / number-spelling utilities consumed by both SSML processing and `TtsTextPreprocessor`: `phoneticApostropheCharacters`, `isEmoji(_:)`, `isWordCharacter(_:apostrophes:)`, `spellOutFormatter`, `digitWords`, `digitToWord(_:)`.

## Motivation — why it exists

## Context

## What It Does
`SSMLProcessingResult` is `public Sendable`: `text: String` + `phoneticOverrides: [TtsPhoneticOverride]`. It's the single return type of `SSMLProcessor.process(_:)`.

`SSMLParsedTag` is internal: a `TagType` enum + a `Range<String.Index>` pointing into the original input. `TagType` has three cases (`phoneme`, `sub`, `sayAs`) — all `Sendable`.

Shared utilities live at file scope so both `SSMLProcessor` and `TtsTextPreprocessor` use the same word-boundary semantics:
- `phoneticApostropheCharacters` — straight `'`, curly `'`, modifier letter apostrophe `ʼ`, etc.
- `isEmoji(_:)` — character is an emoji if any scalar has `isEmojiPresentation` or `isEmoji`.
- `isWordCharacter(_:apostrophes:)` — letter, number, member of `apostrophes`, or emoji.
- `spellOutFormatter` — a cached `NumberFormatter` with `numberStyle = .spellOut`, `Locale("en_US_POSIX")`, max 0 fraction digits, rounding `.down`. Cached because `NumberFormatter` is expensive to construct.
- `digitWords` — `["zero", "one", …, "nine"]`.
- `digitToWord(_:)` — single-digit char → word, returns nil for non-digits.

## Key Code
- [`Sources/FluidAudio/TTS/SSML/SSMLTypes.swift:4`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLTypes.swift#L4) — `public struct SSMLProcessingResult`.
- [`Sources/FluidAudio/TTS/SSML/SSMLTypes.swift:15`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLTypes.swift#L15) — `struct SSMLParsedTag` + `TagType`.
- [`Sources/FluidAudio/TTS/SSML/SSMLTypes.swift:30`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLTypes.swift#L30) — `phoneticApostropheCharacters`.
- [`Sources/FluidAudio/TTS/SSML/SSMLTypes.swift:42`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLTypes.swift#L42) — `isWordCharacter(_:apostrophes:)`.
- [`Sources/FluidAudio/TTS/SSML/SSMLTypes.swift:51`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLTypes.swift#L51) — `spellOutFormatter` (cached).

## Edge Cases & Failure Modes
- `spellOutFormatter`'s `Locale("en_US_POSIX")` means non-English fallback to English digit words — fine for the current beta scope (en-only).
- `isWordCharacter` treats emoji as word characters — phoneme overrides on emoji boundaries can land at unexpected indices.
- [REVIEW: the file-scope `let` declarations are not in any namespace — confirm they don't collide with similarly-named symbols elsewhere.]

## Test Coverage
- `Tests/FluidAudioTests/TTS/SSMLTests.swift` (uses these types).

## Changelogs
