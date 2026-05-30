---
id: ssml-processor
name: SSML Processor
repo: FluidAudio
status: active
linked_features: [ssml-tag-parser, ssml-types, ssml-say-as-interpreter, kokoro-tts-manager, kokoro-pipeline]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# SSML Processor

## TL;DR
Pure-Swift, regex-based SSML pre-processor that strips `<phoneme>`, `<sub>`, and `<say-as>` tags from input text, returns the cleaned text plus a list of `TtsPhoneticOverride` objects keyed by word index, and applies say-as interpretation inline. Hot path optimization: tagless input bypasses parsing entirely.

## Motivation — why it exists

## Context

## What It Does
`SSMLProcessor.process(_:)` is the single public entry point. Step 0: if the input contains no `<`, return immediately with the unchanged text and an empty override list. Otherwise, `SSMLTagParser.parse(...)` produces a list of `SSMLParsedTag` values sorted in reverse document order, so each replacement uses still-valid indices into the working text.

Three tag types are handled by `process`:

1. `<phoneme alphabet="ipa" ph="…">word</phoneme>` — word index in the cleaned text is computed via `countWordsBeforeIndex` before the swap; the tag is replaced with its inner content; a `TtsPhoneticOverride` is appended carrying both tokenized phonemes (`tokenizePhonemes` splits on whitespace when present, otherwise keeps the whole IPA blob as a single token) and the unicodeScalars-split fallback. Downstream `PhonemeMapper` consumes both forms.
2. `<sub alias="…">…</sub>` — replaced with the alias text; no override emitted.
3. `<say-as interpret-as="…" format="…">…</say-as>` — content is run through `SayAsInterpreter.interpret(...)` (cardinal/ordinal/digits/date/time/telephone/fraction/characters/spell-out) and the tag is replaced with the interpreted string.

After processing, `phoneticOverrides` are sorted by ascending word index (they were appended in reverse document order during replacement) and returned in a `SSMLProcessingResult`.

`countWordsBeforeIndex` uses the shared `isWordCharacter` predicate (letters, numbers, apostrophe variants, emoji) to count completed words in the prefix — a partial in-progress word at the boundary is intentionally not counted, so the index points at the word that the phoneme override applies to.

## Key Code
- [`Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift:10`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift#L10) — `process(_:)` single public entry.
- [`Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift:12`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift#L12) — fast path: tagless input bypasses parsing.
- [`Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift:23`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift#L23) — reverse-document-order replacement loop.
- [`Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift:25`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift#L25) — `<phoneme>` handling: word-index computation + override emission.
- [`Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift:45`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift#L45) — `<sub>` handling: alias swap.
- [`Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift:49`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift#L49) — `<say-as>` handling: dispatch to `SayAsInterpreter`.
- [`Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift:61`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift#L61) — override sort by ascending word index.
- [`Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift:70`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift#L70) — `countWordsBeforeIndex` (partial-word handling).
- [`Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift:93`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift#L93) — `tokenizePhonemes` (space-split vs single-blob fallback).

## Edge Cases & Failure Modes
- Input without `<` skips all regex work — important for the common no-SSML case.
- Tags are processed in *reverse document order* but overrides are accumulated in encounter order then re-sorted ascending — the API guarantees forward word-index ordering downstream.
- Nested tags are not supported (each regex matches `[^<]*` content, so a tag inside a tag breaks the outer match).
- `<sub>` does not emit an override — alias text flows through normal word boundary detection.
- An empty phoneme attribute produces a single-element token array `[""]` — `PhonemeMapper` must tolerate this.
- Unknown `interpret-as` types pass content through unchanged (see `SayAsInterpreter.interpret` `default` branch).
- [REVIEW: regex parsing is intentionally lax — no XML well-formedness check, no entity decoding, no attribute order independence beyond what the regex allows. Malformed input degrades gracefully but silently.]
- [REVIEW: word index counted in the post-replacement working text, but tags processed in reverse — confirm interaction when two phoneme tags target adjacent words; index of the later-in-document one may shift by content-vs-tag length difference.]

## Performance / Concurrency Notes
- Pure value-type API; no shared mutable state.
- Regex compiled lazily inside `SSMLTagParser` (one per tag kind, then reused).
- `tokenizePhonemes` returns a single-element array for the no-space case so `PhonemeMapper` can do the work — keeps the SSML layer model-agnostic.

## Test Coverage
- `Tests/FluidAudioTests/TTS/SSMLTests.swift`

## Changelogs
