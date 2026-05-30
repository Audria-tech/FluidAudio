# FluidAudio — Graph Report

Generated 2026-05-29 by Claude (LLM-driven code-graph analysis) at HEAD `94e54782a613d6ce2ba627e17e930f3107652530`. Note: Graphify is a skill, not a CLI; this report is the LLM-analyzed equivalent.

## Top-level modules

```
FluidAudio (library)
├── ASR
│   ├── Parakeet  (SlidingWindow + Streaming)
│   ├── Cohere
│   ├── Qwen3
│   └── Shared    (PunctuationCommitLayer)
├── Diarizer
│   ├── Core      (DiarizerManager)
│   ├── Sortformer
│   ├── LS-EEND
│   ├── Clustering / Extraction / Segmentation
│   └── Offline   (independent batch pipeline)
├── VAD           (Silero)
├── TTS
│   ├── Kokoro / KokoroAne / PocketTTS / Magpie / CosyVoice3 / StyleTTS2
│   ├── SSML
│   └── G2P       (multilingual)
├── ITN           (TextNormalizer)
└── Shared        (ANE utils, AudioConverter, Mel, ModelRegistry, DownloadUtils)

FluidAudioCLI    (depends on FluidAudio)
FastClusterWrapper / MachTaskSelfWrapper   (C/C++ bridges)
```

## High-traffic edges (most-called modules)

| Module | Called by | Why |
|---|---|---|
| `Shared/ModelRegistry` | every manager that downloads a model | HF URL construction. |
| `Shared/DownloadUtils` | every model loader | parallel HF model download. |
| `Shared/AudioConverter` | every audio entry point | resample to 16 kHz mono Float32. |
| `Shared/AudioMelSpectrogram` | Parakeet, Cohere, Qwen3 (Whisper-style mel), VAD | front-end mel features. |
| `Shared/ANEMemoryOptimizer` | every Core ML predict path | zero-copy input/output. |
| `Shared/MLModel+Prediction` | every manager | async predict wrappers. |
| `Diarizer/Extraction/EmbeddingExtractor` | online + offline diarizers | speaker embeddings. |
| `ASR/Parakeet/Streaming/RnntDecoder` | streaming Parakeet + Nemotron | RNNT step. |
| `ASR/Parakeet/Streaming/Tokenizer` | every Parakeet variant | SentencePiece. |
| `Diarizer/Clustering/SpeakerManager` | online diarizers + speaker ID | speaker registry. |

## Per-subsystem entry points

- **ASR/SlidingWindow**: `SlidingWindowAsrManager.transcribe(audio:)` → `SlidingWindowAsrSession` → `TDT/CTC` decoder → tokenizer → text + timings.
- **ASR/Streaming**: `StreamingAsrManager.feedAudio()` → encoder cache → `RnntDecoder` → token dedup → emit partial/final.
- **ASR/Cohere**: `CoherePipeline.transcribe(audio:)` → CoreML inference → text.
- **ASR/Qwen3**: `Qwen3AsrManager.transcribe(audio:)` / streaming → RoPE → Whisper mel → text.
- **Diarizer (online)**: `DiarizerManager.processChunk(audio:)` → segmentation → embedding → `SpeakerManager` → timeline.
- **Diarizer (Sortformer)**: `SortformerDiarizer.process(audio:)` → encoder → state updater → timeline.
- **Diarizer (LS-EEND)**: `LSEENDDiarizer.diarize(audio:)` → preprocessor → inference → timeline.
- **Diarizer (offline)**: `OfflineDiarizerManager.diarize(audio:)` → segmentation → embedding → clustering (AHC/KMeans/VBx) → timeline.
- **VAD**: `VadManager.detectSegments(audio:)` / streaming → Silero CoreML.
- **TTS**: per-engine manager (`KokoroTtsManager.synthesize`, `MagpieTtsManager.synthesize`, `PocketTtsManager.synthesize`, etc.) → G2P → engine pipeline → AudioPostProcessor.
- **ITN**: `TextNormalizer.normalize(text:)` (text-only, no ML).

## Concurrency model

- Heavy use of Swift 6 actors and `@MainActor`. No `@unchecked Sendable` (per repo invariant).
- Per-manager actor-isolated state; predict paths are `async` wrappers around `MLModel.prediction(from:)`.
- `nonisolated(unsafe)` is used in `ModelRegistry` for a single mutable static (the customizable base URL) under a "set-once at startup" assumption — flag this in self-review as a cross-cutting concern.

## Memory model (notable)

- `ANEMemoryOptimizer` / `ANEMemoryUtils` enforce 64-byte alignment for ANE inputs.
- `MLArrayCache` reuses `MLMultiArray` allocations across predict calls.
- `ZeroCopyFeatureProvider` avoids dictionary copies between predict() calls.
- `MagpieMemoryMonitor` (used in Magpie autoregressive TTS) watches RSS to throttle generation.

## External dependencies (Swift PM)

Per `Package.swift`:
- No third-party Swift dependencies for the library target; all heavy ML is Core ML.
- `FastClusterWrapper` brings a vendored C++ fastcluster source via `cxxLanguageStandard`.

## Implications for feature.md generation

- Per-engine TTS files have similar shape (manager + pipeline + assets) → use compact tier with cross-references rather than duplicating boilerplate.
- Decoders (TDT, CTC, RNNT) are stateful and share scaffolding (encoder cache, tokenizer) → compact features with explicit edge cases.
- Shared utilities (ANE, MLArrayCache, AudioConverter, AudioMelSpectrogram) deserve full-depth features because they're the cross-cutting performance layer.
