---
id: cosyvoice3-tts-manager
name: CosyVoice3 TTS Manager
repo: FluidAudio
status: active
linked_features: [cosyvoice3-pipeline-preprocess, cosyvoice3-pipeline-synthesize, cosyvoice3-models, cosyvoice3-shared, tts-backend-protocol]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# CosyVoice3 TTS Manager

## TL;DR
Public actor entry point for the CosyVoice3 (Mandarin) zero-shot voice cloning pipeline: Qwen2 LM (prefill + decode) → Flow CFM → HiFT vocoder. Exposes both a Phase 1 fixture replay (`synthesizeFromFixture`) and a Phase 2 text-driven path (`synthesize(text:promptAssets:)`). Currently slow — RTFx < 1.0 typical.

## Motivation — why it exists

## Context

## What It Does
`CosyVoice3TtsManager` is a `public actor` that owns a `CosyVoice3ModelStore` plus optional text-frontend resources (Qwen2 tokenizer directory, a runtime safetensors embedding file, and a special-tokens JSON map). Two `init` overloads exist: a fixture-only constructor (Phase 1 parity harness) and the full text-mode constructor. `downloadAndCreate(...)` is the convenience factory — it fetches ~5.8 GB of models from `FluidInference/CosyVoice3-0.5B-coreml`, ensures the text-frontend assets, and optionally pre-fetches the default `cosyvoice3-default-zh` voice bundle.

`initialize()` is idempotent: it loads the four CoreML models (`prefill`, `decode`, `flow`, `hift`) into a `CosyVoice3Synthesizer`, mmaps the speech-embedding safetensors into a `CosyVoice3SpeechEmbeddings`, and (if frontend resources are configured) loads `Qwen2BpeTokenizer` + `CosyVoice3TextEmbeddings` + 281 runtime special tokens to build a `CosyVoice3TextFrontend`. Voices are loaded on demand via `loadVoice(_:)` which materializes a `CosyVoice3PromptAssets` (prompt text + precomputed `llm_prompt_speech_ids` + `prompt_mel` + `spk_embedding`).

Phase 2 `synthesize(text:promptAssets:options:prenormalized:)` runs the optional `CosyVoice3ChineseNormalizer` (bypassed for SSML-like input, non-Chinese, or `prenormalized: true`), auto-chunks long input via `CosyVoice3TextChunker` to fit a structural 250-token Flow cap, then per chunk: frontend `assemble(...)` builds `lm_input_embeds`, an in-memory `CosyVoice3FrontendFixture` adapter routes the chunk back through the Phase 1 synthesizer path. Multi-chunk results are stitched with a short cosine equal-power cross-fade (default 8 ms) at boundaries to mask DC/phase mismatch. `finishedOnEos` is true only when every chunk ended naturally.

`flattenLmEmbeds` handles non-compact strides when reading the `[1, tPre, 896]` fp32 multi-array out of CoreML into a flat `[Float]`. `loadSpecialTokens` accepts both the `tokenizer_fixture.json` nested form (`{"special_tokens": {...}, "cases": [...]}`) and a flat map.

## Key Code
- [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift:44`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift#L44) — `public actor CosyVoice3TtsManager` declaration.
- [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift:97`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift#L97) — `downloadAndCreate(...)` factory: fetches models, frontend, default voice.
- [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift:122`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift#L122) — `loadVoice(_:)` lazy voice-bundle materialization.
- [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift:140`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift#L140) — `initialize()`: loads 4 models + speech embeddings + (optional) text frontend.
- [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift:171`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift#L171) — `synthesizeFromFixture(...)` parity replay path.
- [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift:193`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift#L193) — `synthesize(text:promptAssets:options:prenormalized:)` end-to-end text path.
- [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift:261`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift#L261) — `synthesizeChunk(...)` shared per-chunk path that uses the in-memory fixture adapter.
- [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift:303`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift#L303) — `mergeChunkedResults(...)` cross-fade stitching + EOS aggregation.
- [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift:330`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift#L330) — `concatWithCrossfade(...)` cosine equal-power overlap-add.
- [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift:368`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift#L368) — `flattenLmEmbeds(...)` stride-aware MLMultiArray flatten.
- [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift:400`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift#L400) — `loadSpecialTokens(...)` JSON parsing (nested or flat map).

## Edge Cases & Failure Modes
- Calls to `synthesize*` before `initialize()` throw `CosyVoice3Error.notInitialized`.
- Phase 2 `synthesize(text:promptAssets:)` requires both `synthesizer` and `textFrontend` to be loaded — fixture-only constructor will throw at runtime if the text path is invoked.
- Normalization is automatically skipped when the input contains SSML-like markers (`<|...|>`) or when there are no CJK characters at all — mirroring Python's bypass.
- `options.replayDecodedTokens` must be `false` in text mode; the synthesizer rejects replay-mode for text-driven input.
- Cross-fade degrades gracefully: fade window = `min(fadeMs, prev.tail/2, next.head/2)`, so very short chunks fall back to a plain concatenation rather than consuming the entire chunk.
- `flattenLmEmbeds` throws `CosyVoice3Error.invalidShape` if dtype/shape don't match `[1, tPre, embedDim] fp32`.
- `loadSpecialTokens` throws `invalidShape` if the parsed map is empty.
- [REVIEW: performance — RTFx < 1.0 typical on M-series; Flow stage is forced fp32 CPU/GPU because fp16 + ANE produces NaNs through fused `layer_norm`. HiFT has ~12 sinegen/windowing ops that fall back to CPU. Investigation ongoing whether this is a model or conversion issue.]
- [REVIEW: experimental API — Swift API, model layout, and prompt-asset format may change in subsequent releases without deprecation aliases.]

## Performance / Concurrency Notes
- Actor isolation serializes all synthesis calls; model store is also actor-isolated.
- Initialization is split: synthesizer loads always, text frontend loads only when configured — fixture-only callers don't pay the tokenizer cost.
- Auto-chunking budget = 250 speech tokens; `CosyVoice3TextChunker.estimateSpeechTokens` drives the split decision and is also logged per chunk.
- The shared per-chunk path constructs an in-memory `CosyVoice3FrontendFixture` so a single `synthesize(fixture:)` code path serves both Phase 1 and Phase 2.

## Test Coverage
- `Tests/FluidAudioTests/TTS/CosyVoice3ChineseNormalizerTests.swift`
- `Tests/FluidAudioTests/TTS/CosyVoice3ModelNameTests.swift`
- `Tests/FluidAudioTests/TTS/CosyVoice3PromptMelTests.swift`
- `Tests/FluidAudioTests/TTS/CosyVoice3TextChunkerTests.swift`

## Changelogs
