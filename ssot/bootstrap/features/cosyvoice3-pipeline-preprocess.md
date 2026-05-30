---
id: cosyvoice3-pipeline-preprocess
name: CosyVoice3 Pipeline (Preprocess)
repo: FluidAudio
status: active
linked_features: [cosyvoice3-tts-manager, cosyvoice3-pipeline-synthesize, cosyvoice3-models, cosyvoice3-shared]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# CosyVoice3 Pipeline (Preprocess)

## TL;DR
CosyVoice3 frontend: Mandarin minimal normalizer, Qwen2 BPE tokenizer (HF vocab.json + merges.txt + 281 runtime special tokens), runtime safetensors text embeddings, prompt-asset bundle loader (`CosyVoice3PromptAssets`), prompt-mel mmap reader, the text/speech-token chunker (estimates token budget under 250-token Flow cap), and the Phase 1 fixture loader.

## Motivation — why it exists

## Context

## What It Does
**`CosyVoice3ChineseNormalizer`** — minimal Chinese normalizer with a `containsChinese(_:)` cheap predicate that gates whether normalization runs at all. Used as a fallback when wetext isn't pre-applied server-side.

**`Qwen2BpeTokenizer`** — loads HF Qwen2 assets (`vocab.json` + `merges.txt`) plus a `[String: Int32]` special-token map (281 runtime-added tokens including `<|endofprompt|>` = 151646). Standard BPE merge-priority tokenization.

**`CosyVoice3TextEmbeddings`** — mmap of `embeddings-runtime-fp32.safetensors` containing Qwen2 `text_embedding` and CosyVoice3 `speech_embedding` rows at runtime dtype. Reads via the shared `SafetensorsFile`.

**`CosyVoice3PromptAssets`** — loadable bundle: prompt text (with `<|endofprompt|>`), precomputed `llm_prompt_speech_ids`, `prompt_mel`, `spk_embedding`. Static `load(from:)`.

**`CosyVoice3PromptMel`** — fp32 mel mmap reader.

**`CosyVoice3TextChunker`** — splits long input under the 250-token Flow cap; greedy on hard sentence enders + soft clause separators when the running speech-token estimate exceeds budget. `estimateSpeechTokens(_:)` is the budget oracle.

**`CosyVoice3TextFrontend`** — `assemble(promptText:ttsText:promptSpeechIds:)` produces the `lm_input_embeds` matrix and `tPre` (prompt length) for the synthesizer.

**`CosyVoice3FrontendFixture`** — Phase 1 parity fixture loader / in-memory adapter used by both the fixture-replay path and the text-mode chunked path.

## Key Code
- [`Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/CosyVoice3TextChunker.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/CosyVoice3TextChunker.swift) — 250-token budget chunker.
- [`Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/CosyVoice3TextFrontend.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/CosyVoice3TextFrontend.swift) — `assemble(...)` lm_input_embeds builder.
- [`Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/Qwen2BpeTokenizer.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/Qwen2BpeTokenizer.swift) — vocab + merges + special tokens.
- [`Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/CosyVoice3PromptAssets.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/CosyVoice3PromptAssets.swift) — prompt bundle loader.
- [`Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/CosyVoice3ChineseNormalizer.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/CosyVoice3ChineseNormalizer.swift) — minimal Chinese normalizer.

## Edge Cases & Failure Modes
- Normalization is skipped automatically for SSML-like input (`<|...|>`), non-Chinese text, or `prenormalized: true`.
- Chunker's 250-token cap is hard; long chunks split on hard enders first, then soft clause separators.
- Tokenizer requires *both* HF vocab and the 281-token runtime special-token map; missing either throws at construction.
- [REVIEW: minimal normalizer — may miss edge cases that wetext handles; doc explicitly suggests pre-normalizing server-side via wetext.]

## Test Coverage
- `Tests/FluidAudioTests/TTS/CosyVoice3ChineseNormalizerTests.swift`, `CosyVoice3PromptMelTests.swift`, `CosyVoice3TextChunkerTests.swift`.

## Changelogs
