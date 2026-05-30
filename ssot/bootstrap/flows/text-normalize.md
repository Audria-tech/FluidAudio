---
id: text-normalize
name: ITN Text Normalize
trigger_type: user-action
trigger: Caller invokes `TextNormalizer.shared.normalize(_:)` / `normalizeSentence(_:)` / `normalize(result:)` on an ASR transcript.
end_state: A written-form string (or `ASRResult` wrapper preserving timings) is returned; when the native nemo library is unavailable, input is returned unchanged.
involves_features: [text-normalizer]
linked_flows: []
sync_async: sync
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# ITN Text Normalize

## TL;DR
Inverse Text Normalization: spoken-form → written form ("two hundred thirty two" → "232", "five dollars and fifty cents" → "$5.50"). Bridges a Rust/C nemo native library via `dlsym` with graceful identity-fallback when symbols aren't present. Uses Apple `NaturalLanguage.NLTagger` to disambiguate "period"/"dash"/"colon" as words vs. punctuation commands.

## Trigger & Preconditions
`TextNormalizer.shared` (singleton). `init()` resolves `nemo_normalize` / `nemo_free_string` / `nemo_version` via `dlopen(nil, RTLD_NOW)` + `dlsym`. If any of those three is missing, `isNativeAvailable = false` and every API short-circuits.

## Stages
1. **Short-circuit on no native** — every API checks `isNativeAvailable` and returns input unchanged when false: [`Sources/FluidAudio/ITN/TextNormalizer.swift:185`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ITN/TextNormalizer.swift#L185).
2. **Ambiguous-word filter (sentence mode)** — `normalizeSentence(_:)` calls `filterAmbiguousWords(in:)` first; cheap presence check against `{period, dash, colon, pipe, slash, dot, plus, hash, percent}`, then `NLTagger(tagSchemes: [.lexicalClass])` tags occurrences; nouns/verbs/adjectives/adverbs are treated as natural language: [`Sources/FluidAudio/ITN/TextNormalizer.swift:323`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ITN/TextNormalizer.swift#L323).
3. **FFI call** — `nemo_normalize` (single-expression) or `nemo_normalize_sentence` / `nemo_normalize_sentence_with_max_span` (sentence mode with optional max-span tokens). Sentence-mode falls back to single-expression mode when the symbol isn't exported.
4. **Copy result + free** — `String(cString:)` copies the returned `CChar*`; `nemo_free_string` releases the native buffer.
5. **ASR-result wrapper** — `normalize(result: ASRResult)` runs sentence-mode and re-wraps preserving timings/confidence; short-circuits and returns the original `ASRResult` when the text didn't change: [`Sources/FluidAudio/ITN/TextNormalizer.swift:239`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ITN/TextNormalizer.swift#L239).

## Outputs
- Normalized `String` (or original on fallback).
- `ASRResult` envelope variant preserves timings/confidence.

## Error Modes
- When `dlsym` cannot find any of the three minimum symbols, `isNativeAvailable = false` and every API is a passthrough.
- `addRule` / `removeRule` / `clearRules` / `ruleCount` silently no-op if optional `nemo_add_rule` etc. aren't exported.
- [REVIEW] `filterAmbiguousWords` is currently a passthrough — both branches append `String(word)` (the documented passthrough-marker logic appears unimplemented).
- [REVIEW] Native lib distribution: assumes `dlopen(nil, ...)` finds the symbols in the loaded image; no fallback to dlopen by filename.

## Prompts and Models Used

## Usage Metrics

## Changelogs
