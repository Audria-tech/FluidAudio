---
id: ssml-prosody-pipeline
name: SSML Prosody Pipeline
trigger_type: event
trigger: A TTS manager's preprocessor receives input text and calls `SSMLProcessor.process(_:)` to strip tags and surface phonetic overrides.
end_state: Cleaned text and an ordered `[TtsPhoneticOverride]` list (keyed by word index) are returned to the preprocessor; downstream G2P and synthesis honor the overrides.
involves_features: [ssml-processor, ssml-tag-parser, ssml-types, ssml-say-as-interpreter]
linked_flows: [kokoro-synthesize, magpie-synthesize]
sync_async: sync
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# SSML Prosody Pipeline

## TL;DR
Pure-Swift, regex-based SSML pre-processor for the TTS frontends. Strips `<phoneme>`, `<sub>`, `<say-as>` tags from the input, returns the cleaned text plus a list of `TtsPhoneticOverride` keyed by word index, and applies `<say-as>` interpretation inline. Tagless input bypasses parsing entirely.

## Trigger & Preconditions
Called by `TtsTextPreprocessor.preprocessDetailed(...)` for Kokoro and by Magpie's text preprocessor. Input is a `String`; output is consumed by the manager's phoneme mapper.

## Stages
1. **Fast path** — if the input contains no `<`, `SSMLProcessor.process(_:)` returns immediately with unchanged text and an empty override list: [`Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift:12`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift#L12).
2. **Tag parse** — `SSMLTagParser.parse(...)` enumerates the three regex patterns (`phoneme`, `sub`, `say-as`), builds `SSMLParsedTag` values, sorts in *descending* lowerBound order so each replacement uses still-valid `String.Index` ranges: [`Sources/FluidAudio/TTS/SSML/SSMLTagParser.swift:48`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLTagParser.swift#L48).
3. **Reverse-order replacement loop** — `process(_:)` walks tags in reverse document order: [`Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift:23`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift#L23).
4. **`<phoneme>`** — `countWordsBeforeIndex(...)` computes the word index in the cleaned-text prefix using the `isWordCharacter` predicate, the tag is replaced with its inner content, a `TtsPhoneticOverride` is appended with both `tokenizePhonemes(...)` output (space-split or single-blob fallback) and the unicode-scalars-split fallback: [`Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift:25`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift#L25).
5. **`<sub>`** — replaced with the `alias` text; no override emitted (alias flows through normal word-boundary detection): [`Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift:45`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift#L45).
6. **`<say-as>`** — content dispatched to `SayAsInterpreter.interpret(content:interpretAs:format?:)` (one of: `characters`/`spell-out`, `cardinal`/`number`, `ordinal`, `digits`, `date`, `time`, `telephone`, `fraction`; unknown returns content unchanged): [`Sources/FluidAudio/TTS/SSML/SayAsInterpreter.swift:40`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SayAsInterpreter.swift#L40).
7. **Sort overrides** — `phoneticOverrides` (appended in reverse doc order) are re-sorted ascending by word index before returning: [`Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift:61`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/SSML/SSMLProcessor.swift#L61).
8. **Return result** — `SSMLProcessingResult { cleanedText: String, phoneticOverrides: [TtsPhoneticOverride] }`.

## Outputs
- Cleaned text `String`.
- `[TtsPhoneticOverride]` sorted ascending by word index.

## Error Modes
- Nested tags not supported (regex `[^<]*` rejects them).
- Tags missing required attribute (`ph` / `alias` / `interpret-as`) are silently dropped.
- Unknown `interpret-as` returns content unchanged (no error).
- Empty phoneme attribute produces `[""]` token array — `PhonemeMapper` must tolerate.
- No XML well-formedness check, no entity decoding (lax-by-design).
- [REVIEW] `extractAttribute(_:from:)` recompiles a regex per call (line 32-35 of `SSMLTagParser`).
- [REVIEW] Adjacent-word phoneme tags: word-index counted in post-replacement text but tags processed in reverse — index of later-in-document tag may shift by content-vs-tag length difference.

## Prompts and Models Used

## Usage Metrics

## Changelogs
