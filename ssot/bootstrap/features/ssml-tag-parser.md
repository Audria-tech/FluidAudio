---
id: ssml-tag-parser
name: SSML Tag Parser
repo: FluidAudio
status: active
linked_features: [ssml-processor, ssml-types, ssml-say-as-interpreter]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# SSML Tag Parser

## TL;DR
Regex-based parser that extracts `<phoneme>`, `<sub>`, and `<say-as>` tags from SSML input and returns `SSMLParsedTag` values sorted in reverse document order — so the caller (`SSMLProcessor`) can replace each tag with its content using still-valid `String.Index` ranges.

## Motivation — why it exists

## Context

## What It Does
`SSMLTagParser` is an internal `enum` namespace. Three compiled regex patterns (`phonemePattern`, `subPattern`, `sayAsPattern`) target the three supported tags; each is case-insensitive and captures `(attributes, content)`. Attribute values are extracted by `extractAttribute(_:from:)` (which compiles a per-call regex for the attribute name — see [REVIEW] below).

`parse(_:)` enumerates matches for each pattern, builds `SSMLParsedTag` values via the `TagType` enum (`phoneme(alphabet:ph:content:)`, `sub(alias:content:)`, `sayAs(interpretAs:format?:content:)`), and returns them sorted by descending `range.lowerBound`. For `<phoneme>`, the `alphabet` attribute defaults to `"ipa"` when absent. For `<say-as>`, `format` is optional. Tags without their required attribute (`ph`, `alias`, `interpret-as`) are silently dropped.

## Key Code
- [`Sources/FluidAudio/TTS/SSML/SSMLTagParser.swift:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLTagParser.swift#L5) — `enum SSMLTagParser`.
- [`Sources/FluidAudio/TTS/SSML/SSMLTagParser.swift:11`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLTagParser.swift#L11) — `phonemePattern`.
- [`Sources/FluidAudio/TTS/SSML/SSMLTagParser.swift:31`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLTagParser.swift#L31) — `extractAttribute(_:from:)` (per-call regex compile).
- [`Sources/FluidAudio/TTS/SSML/SSMLTagParser.swift:48`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLTagParser.swift#L48) — `parse(_:)` main entry.
- [`Sources/FluidAudio/TTS/SSML/SSMLTagParser.swift:82`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLTagParser.swift#L82) — descending-lowerBound sort.

## Edge Cases & Failure Modes
- Content regex `[^<]*` rejects nested tags entirely — by design.
- Attribute quotes accept `"` or `'`; whitespace between `=` and value is tolerated.
- Tags missing the required attribute (`ph` / `alias` / `interpret-as`) are silently dropped, not flagged.
- Unicode attribute values are passed through whole; no entity decoding is applied.
- [REVIEW: `extractAttribute(_:from:)` compiles a new `NSRegularExpression` on every call (lines 32-35) — for SSML-heavy inputs this is wasteful; consider a per-name cache.]

## Test Coverage
- `Tests/FluidAudioTests/TTS/SSMLTests.swift` (covers parser via `SSMLProcessor`).

## Changelogs
