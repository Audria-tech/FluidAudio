---
id: pocket-tts-manager
name: PocketTTS Manager
repo: FluidAudio
status: active
linked_features: [pocket-tts-pipeline, pocket-tts-tokenizer, tts-backend-protocol, audio-post-processor]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# PocketTTS Manager

## TL;DR
Public actor façade for PocketTTS — a flow-matching language model that generates audio autoregressively at 24 kHz, one 80 ms frame (1920 samples) per step. Supports streaming, persistent sessions with warm voice KV cache, and voice cloning via the Mimi encoder.

## Motivation — why it exists

## Context

## What It Does
`PocketTtsManager` is a `public actor` that owns a `PocketTtsModelStore` (lazy CoreML download/load) and a default-voice string. The language pack (`PocketTtsLanguage`) is fixed at construction — to switch languages, the docstring requires creating a new manager. Default precision is `.fp16` (matching upstream on-disk weights); `.int8` swaps `flowlm_step` for the upstream `flowlm_stepv2` int8-quantized variant.

`initialize()` triggers `modelStore.loadIfNeeded()` (downloads + compiles models if missing). After init, three modes of synthesis are exposed:

1. **One-shot WAV**: `synthesize(text:voice:temperature:deEss:)` returns `Data` (a 24 kHz WAV). A second overload takes pre-computed `PocketTtsVoiceData` for cloned voices.
2. **Streaming**: `synthesizeStreaming(...)` returns `AsyncThrowingStream<PocketTtsSynthesizer.AudioFrame, Error>` — each frame is 1920 Float32 samples, yielded as generated. Enables playback before the utterance finishes.
3. **Session**: `makeSession(...)` returns a `PocketTtsSession` that keeps the voice prefill (~125 tokens) warm across multiple `enqueue()` calls so subsequent utterances only pay the text-prefill cost. Mimi decoder state persists for seamless audio continuity between utterances.

Voice cloning is gated on the optional `mimi_encoder` model: `isVoiceCloningAvailable()` checks presence, `cloneVoice(from: URL | [Float])` produces `PocketTtsVoiceData`, `saveClonedVoice`/`loadClonedVoice` round-trip to a `.bin` file. `cloneVoiceToFile(from:to:)` does both steps.

The synthesizer delegates to `PocketTtsSynthesizer` (a struct with static methods) and uses a `@TaskLocal` model-store context (`withModelStore`) so async closures pick up the dependency without explicit threading.

## Key Code
- [`Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift:16`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift#L16) — `public actor PocketTtsManager` declaration.
- [`Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift:39`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift#L39) — `init(defaultVoice:language:directory:precision:)`.
- [`Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift:59`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift#L59) — `initialize()` lazy download/load.
- [`Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift:73`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift#L73) — `synthesize(text:voice:temperature:deEss:)` → WAV Data.
- [`Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift:178`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift#L178) — `synthesizeStreaming(text:...)` returns `AsyncThrowingStream<AudioFrame, Error>`.
- [`Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift:251`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift#L251) — `makeSession(voice:temperature:seed:)` for persistent warm-cache sessions.
- [`Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift:340`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift#L340) — `isVoiceCloningAvailable()`: checks for `mimi_encoder`.
- [`Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift:358`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift#L358) — `cloneVoice(from: URL)` zero-shot voice cloning.
- [`Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift:381`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/TTS/PocketTTS/PocketTtsManager.swift#L381) — `saveClonedVoice(_:to:)` / `loadClonedVoice(from:)`.

## Edge Cases & Failure Modes
- Pre-init synthesis throws `PocketTTSError.modelNotFound("PocketTTS model not initialized")`.
- `cloneVoice(...)` requires the `mimi_encoder` model; throws `PocketTTSError.modelNotFound` if it's not in the bundle for the configured language.
- `synthesizeToFile` deletes any existing file at `outputURL` first (destructive).
- Language is immutable after construction; switching languages requires a new manager.
- Sessions: `enqueue` while a prior session is running shares Mimi decoder state — concurrent sessions on the same manager may interfere.
- [REVIEW: precision — `.int8` precision relies on upstream `flowlm_stepv2`; confirm that the int8 variant ships for every supported language pack.]
- [REVIEW: streaming overlap — docstring describes "seamless audio continuity" across session utterances, but no test in the read scope explicitly validates phase/sample continuity at utterance boundaries.]

## Performance / Concurrency Notes
- Actor isolation serializes manager-level calls; the model store is also actor-isolated.
- Each generation step emits an 80 ms (1920-sample) frame at 24 kHz — streaming therefore has ~80 ms time-to-first-audio granularity once warm.
- Long text is chunked into sentence-sized pieces (≤50 tokens each) to stay under the 512-position KV cache (per `PocketTtsSynthesizer` docs).
- Session warmup amortizes the ~125-token voice prefill across many utterances; documented in `makeSession` doc comment.
- README claims ~1.5–2× RTFx for streaming Mimi path.

## Test Coverage
- `Tests/FluidAudioTests/TTS/PocketTTS/PocketTtsLanguageTests.swift`
- `Tests/FluidAudioTests/TTS/PocketTTS/PocketTtsSessionTests.swift`
- `Tests/FluidAudioTests/TTS/PocketTTS/PocketTtsStreamingTests.swift`
- `Tests/FluidAudioTests/TTS/PocketTTS/SentencePieceProtoTests.swift`

## Changelogs
