---
id: magpie-synthesize
name: Magpie TTS Synthesize
trigger_type: user-action
trigger: Caller invokes `MagpieTtsManager.synthesize(text:speaker:language:options:)` or `synthesizeStream(...)`.
end_state: 22 kHz PCM audio is returned (single buffer) or yielded as `MagpieAudioChunk`s on a stream; tokenizer + model caches remain warm for the next call.
involves_features: [magpie-tts-manager, magpie-pipeline, magpie-local-transformer, multilingual-g2p, audio-post-processor]
linked_flows: [model-download-and-load, ssml-prosody-pipeline]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Magpie TTS Synthesize

## TL;DR
NVIDIA Magpie TTS Multilingual 357M encoder-decoder transformer with NanoCodec vocoder. Per-language tokenizer → text_encoder → 110-step prefill → 8-codebook AR loop up to 500 steps → NanoCodec fp32 PCM 22 kHz. Experimental — slow on Apple Silicon (RTFx ≈ 0.04 warm).

## Trigger & Preconditions
`MagpieTtsManager.initialize()` (downloads four CoreML graphs, materializes tokenizer for `preferredLanguages`, runs `synthesizer.warmup()` with EOS forbidden). Languages: 8 supported (ja excluded — requires OpenJTalk + MeCab ~102 MB). Default compute units `.cpuAndNeuralEngine`.

## Stages
1. **Per-language tokenize** — `MagpieTokenizer` dispatches to `MagpieCharTokenizer` / `MagpieMandarinTokenizer` / `MagpiePhonemeTokenizer` from `tokenizer_data/<lang>/`. Pads/truncates to `MagpieConstants.maxTextLength = 256` and produces `MagpieTokenizedText(paddedIds, mask, realLength)`: [`Sources/FluidAudio/TTS/Magpie/Pipeline/Preprocess/MagpieTokenizer.swift:30`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/Pipeline/Preprocess/MagpieTokenizer.swift#L30).
2. **IPA override (optional)** — when `options.allowIpaOverride == true`, `MagpieIpaOverride` parses `|...|` regions for direct IPA token lookup against the language's `token2id` map; outside those regions, normal language tokenization runs.
3. **Streaming chunker (if `synthesizeStream`)** — `MagpieChunker` splits long input; first chunk capped to ~50 codec frames (≈2.3 s) for low time-to-first-audio, later chunks pack at normal capacity.
4. **Text encoder** — runs the `text_encoder` CoreML graph on the padded IDs + mask.
5. **Optional CFG** — additional zero-context encoder pair for classifier-free guidance.
6. **Prefill** — `MagpiePrefill` runs the 110-step prefill against `decoder_prefill`, populating `MagpieKvCache` (capped at `maxCacheLength = 512`).
7. **8-codebook AR loop** — `MagpieSynthesizer` loops up to 500 steps: embed current 8 codes → `decoder_step` → local-transformer sample (`MagpieSampler` in pure Swift/BLAS so 8-codebook sampling stays cache-resident) → new codes. `minFrames` gates EOS to avoid premature termination: [`Sources/FluidAudio/TTS/Magpie/Pipeline/Synthesize/MagpieSynthesizer.swift:16`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Magpie/Pipeline/Synthesize/MagpieSynthesizer.swift#L16).
8. **NanoCodec decode** — `MagpieNanocodec` renders the codebook stream to fp32 PCM 22 kHz.
9. **Peak normalize (optional)** — when not streaming and `options.peakNormalize == true`, scales the buffer to peak 0.9. Force-disabled in streaming mode.
10. **Emit** — `synthesize` returns the whole `Data`; `synthesizeStream` yields one `MagpieAudioChunk` per chunker chunk as its NanoCodec decode finishes.

## Outputs
- Batch: `Data` 22 kHz PCM (single utterance).
- Stream: `AsyncThrowingStream<MagpieAudioChunk, Error>`; first chunk arrives shortly after first NanoCodec decode completes.

## Error Modes
- `MagpieError.notInitialized` if `synthesize*` called before `initialize()`.
- Japanese tokenizer intentionally absent.
- Speaker 0 has a single trailing-word artifact attributable to fp16 sampler-trajectory drift (not structural).
- Streaming cancellation cancels in-flight synthesis cleanly.
- Pad/truncate to 256 tokens silently discards overflow.
- KV cache hard-capped at 512 positions — long outputs truncated.
- AR loop minFrames > maxSteps disables EOS (used only by warmup).
- Cold-start: first synth ≈ 30 s due to CoreML model load + ANE compile.
- [REVIEW] Docstring inconsistency on RTFx (0.04 warm vs 0.41 agg) within the same file.

## Prompts and Models Used

## Usage Metrics

## Changelogs
