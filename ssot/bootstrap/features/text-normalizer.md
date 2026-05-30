---
id: text-normalizer
name: ITN Text Normalizer
repo: FluidAudio
status: active
linked_features: []
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# ITN Text Normalizer

## TL;DR
Inverse Text Normalization for post-processing ASR output: converts spoken-form text to written form ("two hundred thirty two" → "232", "five dollars and fifty cents" → "$5.50"). Bridges a Rust/C nemo native library via `dlsym`, falls back gracefully to identity when the lib is absent. Uses Apple `NaturalLanguage.NLTagger` to disambiguate "period"/"dash"/"colon" as words vs. punctuation commands.

## Motivation — why it exists

## Context

## What It Does
`TextNormalizer` is a `public final class: Sendable` with a `static let shared` singleton. It binds to a native `nemo_*` C library lazily during `init()` via `dlopen(nil, RTLD_NOW)` + `dlsym`, then stores typed C function pointers in immutable stored properties (so the class can be `Sendable`). The minimum required symbols are `nemo_normalize`, `nemo_free_string`, and `nemo_version`; optional symbols (`nemo_normalize_sentence`, `nemo_normalize_sentence_with_max_span`, `nemo_add_rule`, `nemo_remove_rule`, `nemo_clear_rules`, `nemo_rule_count`) are resolved opportunistically so older library builds still work. If the minimum set is missing, `isNativeAvailable = false` and every API short-circuits to identity.

Three normalization modes:
1. `normalize(_:)` — single-expression mode. Calls `nemo_normalize`, copies the returned `CChar*` via `String(cString:)`, then frees via `nemo_free_string`.
2. `normalizeSentence(_:)` — sliding-window sentence mode. Applies `filterAmbiguousWords` first.
3. `normalizeSentence(_:maxSpanTokens:)` — sentence mode with caller-controlled max span size.

A fourth helper `normalize(result: ASRResult) -> ASRResult` runs sentence-mode normalization and re-wraps the ASR result preserving timings/confidence; it skips allocation when the text is unchanged.

Ambiguous-word handling: a hard-coded set `{period, dash, colon, pipe, slash, dot, plus, hash, percent}` is checked first as a cheap presence filter. When any are present, `NLTagger(tagSchemes: [.lexicalClass])` tags each occurrence; nouns/verbs/adjectives/adverbs are treated as natural language and preserved, others fall through to the normalizer. (Note: the current `filterAmbiguousWords` implementation appends the original word in both branches — the logic is wired through but the actual punctuation-suppression behavior is not yet emitting different output.)

Custom-rule API: `addRule(spoken:written:)`, `removeRule(spoken:)`, `clearRules()`, `ruleCount` — each delegates to the corresponding `nemo_*` symbol if present. Custom rules have the highest priority per the docstring (checked before built-in taggers). Matching is case-insensitive on the spoken form.

`version` returns the native lib's version string when available; useful for diagnostics.

## Key Code
- [`Sources/FluidAudio/ITN/TextNormalizer.swift:20`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ITN/TextNormalizer.swift#L20) — `public final class TextNormalizer: Sendable`.
- [`Sources/FluidAudio/ITN/TextNormalizer.swift:33`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ITN/TextNormalizer.swift#L33) — `ambiguousWords` set: `period, dash, colon, pipe, slash, dot, plus, hash, percent`.
- [`Sources/FluidAudio/ITN/TextNormalizer.swift:75`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ITN/TextNormalizer.swift#L75) — `init()`: `dlopen` + `dlsym` resolve, optional-symbol fallbacks.
- [`Sources/FluidAudio/ITN/TextNormalizer.swift:185`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ITN/TextNormalizer.swift#L185) — `normalize(_:)` single-expression entry.
- [`Sources/FluidAudio/ITN/TextNormalizer.swift:209`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ITN/TextNormalizer.swift#L209) — `normalizeSentence(_:)` sentence mode with ambiguous-word filter.
- [`Sources/FluidAudio/ITN/TextNormalizer.swift:224`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ITN/TextNormalizer.swift#L224) — `normalizeSentence(_:maxSpanTokens:)` configurable span.
- [`Sources/FluidAudio/ITN/TextNormalizer.swift:239`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ITN/TextNormalizer.swift#L239) — `normalize(result:)` ASR-result wrapper.
- [`Sources/FluidAudio/ITN/TextNormalizer.swift:267`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ITN/TextNormalizer.swift#L267) — `addRule(spoken:written:)` custom rule API.
- [`Sources/FluidAudio/ITN/TextNormalizer.swift:302`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ITN/TextNormalizer.swift#L302) — `version` property bridges `nemo_version`.
- [`Sources/FluidAudio/ITN/TextNormalizer.swift:323`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ITN/TextNormalizer.swift#L323) — `filterAmbiguousWords(in:)` NLTagger-based disambiguation.

## Edge Cases & Failure Modes
- When `dlsym` cannot find any of `nemo_normalize`, `nemo_free_string`, `nemo_version`, `isNativeAvailable` is `false` and every method returns the input unchanged. The cleanup branch at lines 78–104 carefully nils every optional pointer.
- Sentence-mode falls back to `normalize(_:)` (whole-input single expression) when `nemo_normalize_sentence` isn't exported.
- `normalize(result:)` short-circuits and returns the original `ASRResult` when text didn't change, avoiding allocation.
- Custom-rule methods silently no-op if the optional symbols aren't present (no error surfaced — caller can check `ruleCount` to detect this).
- [REVIEW: `filterAmbiguousWords` is a no-op — the `if isNaturalLanguage && words.count > 1` branch and the `else` branch both append `String(word)` (lines 359–365). Intended passthrough-marker logic appears to be unimplemented despite the docstring saying "wraps them in a passthrough marker so the Rust normalizer skips them".]
- [REVIEW: native library distribution — the class assumes `dlopen(nil, ...)` will find the symbols in the loaded image (statically linked or dlopen'd dependency). There's no fallback path that dlopens a specific library file by name. Confirm packaging.]

## Performance / Concurrency Notes
- `Sendable` conformance is load-bearing: the class declares immutable C-function-pointer stored properties so cross-actor sharing is safe. There is no shared mutable Swift state, but the underlying native library may keep mutable global state (custom rules) — `addRule`/`removeRule` calls are not Swift-locked.
- NLTagger work happens only when ambiguous words are actually present, courtesy of the `hasAmbiguous` cheap pre-check.
- Each FFI call copies the result string out and frees the original — no shared buffer reuse.
- Singleton `shared` is the recommended access pattern; init cost is paid once per process.

## Test Coverage
- `Tests/FluidAudioTests/TTS/TextNormalizerTests.swift`

## Changelogs
