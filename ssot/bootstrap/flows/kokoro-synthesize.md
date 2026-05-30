---
id: kokoro-synthesize
name: Kokoro TTS Synthesize
trigger_type: user-action
trigger: Caller invokes `KokoroTtsManager.synthesize(text:voice:speakerId:deEss:)` or one of its variants (`synthesizeDetailed`, `synthesizeToFile`).
end_state: Raw 24 kHz PCM audio (or a written WAV file) is returned; voice embeddings and lexicon caches are warm for the next call.
involves_features: [kokoro-tts-manager, kokoro-pipeline, kokoro-custom-lexicon, multilingual-g2p, audio-post-processor]
linked_flows: [model-download-and-load, ssml-prosody-pipeline, g2p-multilingual-graph]
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Kokoro TTS Synthesize

## TL;DR
Kokoro 82M single-graph CoreML TTS. Input text → SSML strip → number/abbreviation expansion → phoneme mapping (lexicon → MultilingualG2P fallback) → voice-pack-conditioned single-graph synthesis → optional biquad de-essing → 24 kHz PCM. Per-call `@TaskLocal` context injection lets the static `KokoroSynthesizer` see the manager's model cache, lexicon assets, and custom lexicon.

## Trigger & Preconditions
`KokoroTtsManager.synthesize(...)`. `initialize(...)` must have downloaded the model bundle, loaded the simple phoneme dictionary, and preloaded the requested voice embeddings. Beta status: only American English voices are QA'd.

## Stages
1. **Text preprocess** — `TtsTextPreprocessor.preprocessDetailed(text)` runs SSML strip (via `SSMLProcessor` — see `ssml-prosody-pipeline.md`), expands numbers/abbreviations, builds word index → phonetic-override map: [`Sources/FluidAudio/TTS/Kokoro/Pipeline/Preprocess/TtsTextPreprocessor.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/Pipeline/Preprocess/TtsTextPreprocessor.swift).
2. **Sanitize** — `KokoroSynthesizer.sanitizeInput(...)` strips invalid characters.
3. **Resolve voice** — `voiceName(for:)` falls back to `availableVoices[abs(speakerId) % count]` when no `voice` provided; `normalizeVoice` trims whitespace, defaults to `TtsConstants.recommendedVoice = "af_heart"`: [`Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift:246`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift#L246).
4. **Ensure voice embedding on disk** — `TtsResourceDownloader.ensureVoiceEmbedding(...)` fetches the voice pack JSON if missing.
5. **Enter task-local context** — three nested `@TaskLocal` scopes (`withLexiconAssets`, `withModelCache`, `withCustomLexicon`) so the static synthesizer can see per-call deps without manager refs: [`Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift:148`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Kokoro/KokoroTtsManager.swift#L148).
6. **Chunk + map to phonemes** — `KokoroChunker` splits long input into capacity-aware sentence-aware chunks (via NaturalLanguage); per word, `TtsCustomLexicon.phonemes(for:)` is consulted first (exact → lowercase → normalized fallback), then the built-in simple-phoneme dictionary, then `MultilingualG2PModel.phonemize(word:language:)` (CharsiuG2P ByT5) — see `g2p-multilingual-graph.md`.
7. **Phoneme → vocab IDs** — `PhonemeMapper` walks IPA tokens to Kokoro vocab IDs.
8. **Pick variant** — `variantPreference` and `KokoroModelCache` lazy-load the 5 s or 15 s graph variant depending on chunk length.
9. **Single-graph synthesis** — `KokoroSynthesizer` runs the variant model with `voiceEmbedding`, `phonemeIds`, etc. The graph emits 24 kHz PCM Float32.
10. **De-ess (optional)** — when `deEss: true` (default), `AudioPostProcessor.deEss(...)` applies a biquad high-shelf filter (default cutoff 6 kHz, -3 dB, Q 0.707) in place: [`Sources/FluidAudio/TTS/Shared/AudioPostProcessor.swift:15`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/Shared/AudioPostProcessor.swift#L15).
11. **Wrap/write** — `synthesize` returns Data; `synthesizeDetailed` returns a result envelope with framing/timing; `synthesizeToFile` deletes any existing output URL first, then writes a 24 kHz WAV via `AudioWAV.data(from:sampleRate:)`.

## Outputs
- `Data` raw 24 kHz PCM (or WAV via `AudioWAV`).
- `SynthesisResult { audio: Data, ...framing }` for the detailed entry.
- Side effect: voice embeddings cached in `ensuredVoices`.

## Error Modes
- `TTSError.modelNotFound("Kokoro model not initialized")` if `synthesize` is called before `initialize`.
- Voice embedding download failure surfaces from `TtsResourceDownloader.ensureVoiceEmbedding`.
- `synthesizeToFile` destructively deletes any existing file at `outputURL`.
- Non-actor class — concurrent `synthesize` from multiple tasks may race on `ensuredVoices`/`defaultVoice`/`customLexicon`/`isInitialized` [REVIEW noted].
- Non-en voices flagged "experimental, not tested" in `TtsConstants`.
- iOS 26+ ANE compiler regression ("Cannot retrieve vector from IRValue format int32") — docstring recommends `.cpuAndGPU` on iOS 26+.

## Prompts and Models Used

## Usage Metrics

## Changelogs
