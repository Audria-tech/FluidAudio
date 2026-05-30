---
id: cosyvoice3-synthesize
name: CosyVoice3 Synthesize (Preprocess + Synthesize)
trigger_type: user-action
trigger: Caller invokes `CosyVoice3TtsManager.synthesize(text:promptAssets:options:prenormalized:)` (Phase 2) or `synthesizeFromFixture(...)` (Phase 1 parity).
end_state: fp32 PCM 22.05 kHz audio is returned in `CosyVoice3SynthesisResult` along with `decodedTokens` and `finishedOnEos`; voice + frontend state remain loaded for the next call.
involves_features: [cosyvoice3-tts-manager, cosyvoice3-pipeline-preprocess, cosyvoice3-pipeline-synthesize, cosyvoice3-models]
linked_flows: [model-download-and-load]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# CosyVoice3 Synthesize (Preprocess + Synthesize)

## TL;DR
Mandarin zero-shot voice cloning. Text → optional Chinese normalize → 250-token chunker → per chunk: Qwen2 BPE tokenize → assemble `lm_input_embeds` → Qwen2 LM prefill + decode (external KV cache) with RAS sampling → Flow CFM (fp32 CPU/GPU) → HiFT vocoder → PCM. Multi-chunk results are stitched with a cosine equal-power cross-fade.

## Trigger & Preconditions
`CosyVoice3TtsManager(...).initialize()` (loads 4 CoreML models + speech embeddings + optional text frontend). For Phase 2 text-driven synthesis both `synthesizer` and `textFrontend` must be loaded; fixture-only constructor throws at runtime if text path is invoked. `loadVoice(_:)` materializes the prompt-asset bundle.

## Stages
1. **Optional normalize** — `CosyVoice3ChineseNormalizer` runs only when `prenormalized == false`, input contains CJK characters, and isn't SSML-like (`<|...|>`): [`Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/CosyVoice3ChineseNormalizer.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/CosyVoice3ChineseNormalizer.swift).
2. **Chunk** — `CosyVoice3TextChunker` greedily splits at hard enders + soft clause separators when `estimateSpeechTokens(...)` exceeds the structural 250-token Flow cap: [`Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/CosyVoice3TextChunker.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/CosyVoice3TextChunker.swift).
3. **Per chunk: tokenize** — `Qwen2BpeTokenizer` (vocab.json + merges.txt + 281 runtime special tokens including `<|endofprompt|>` = 151646): [`Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/Qwen2BpeTokenizer.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/Qwen2BpeTokenizer.swift).
4. **Assemble lm_input_embeds** — `CosyVoice3TextFrontend.assemble(promptText:ttsText:promptSpeechIds:)` produces the embedding matrix and `tPre` (prompt length): [`Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/CosyVoice3TextFrontend.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Preprocess/CosyVoice3TextFrontend.swift).
5. **Wrap as fixture adapter** — `synthesizeChunk(...)` packages the chunked output into an in-memory `CosyVoice3FrontendFixture` so the single `synthesize(fixture:options:)` synthesizer entry serves both Phase 1 and Phase 2: [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift:261`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift#L261).
6. **LM prefill** — `CosyVoice3Synthesizer` consumes `lmInputEmbeds`, runs the `prefill` CoreML model, returns initial external KV cache (no `MLState` — keeps iOS 17 / macOS 14 floor).
7. **LM decode loop** — stateless decode against `decode` CoreML, threading external KV cache. Per step the `CosyVoice3RasSampler` honors `seed` and `maxNewTokens`, emits decoded tokens and `finishedOnEos` flag: [`Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Synthesize/CosyVoice3RasSampler.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/Pipeline/Synthesize/CosyVoice3RasSampler.swift).
8. **Speech token embed** — `CosyVoice3SpeechEmbeddings` mmap returns the per-token embedding rows for the Flow CFM input.
9. **Flow CFM** — forced fp32 CPU/GPU (fp16+ANE NaNs through fused `layer_norm`); produces mel spectrogram.
10. **HiFT vocoder** — renders mel to fp32 PCM at 22.05 kHz; ~12 sinegen/windowing ops fall back to CPU.
11. **Cross-fade stitch** — `mergeChunkedResults(...)` cross-fades adjacent chunks with `concatWithCrossfade(...)` (default 8 ms cosine equal-power, fade window `min(fadeMs, prev.tail/2, next.head/2)`): [`Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift:303`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/CosyVoice3/CosyVoice3TtsManager.swift#L303).
12. **Aggregate EOS** — `finishedOnEos` is true only when every chunk ended naturally.

## Outputs
- `CosyVoice3SynthesisResult { samples: [Float], sampleRate: Int, generatedTokenCount: Int, decodedTokens: [Int], finishedOnEos: Bool }`.

## Error Modes
- `CosyVoice3Error.notInitialized` before `initialize()`.
- Phase 2 with fixture-only constructor: throws at runtime when text path is invoked.
- `options.replayDecodedTokens` must be `false` in text mode (only valid in fixture mode).
- `CosyVoice3Error.invalidShape` if `flattenLmEmbeds` sees mismatched dtype/shape.
- Cross-fade gracefully degrades on very short chunks (window collapses to 0 → plain concat).
- Normalization auto-bypassed for SSML-like input or no-CJK text.
- [REVIEW] RTFx < 1.0 typical on M-series; Flow forced fp32, HiFT partially CPU.

## Prompts and Models Used

## Usage Metrics

## Changelogs
