# FluidAudio — Doc Audit

Generated 2026-05-29 by Claude (Cowork autonomous session) at HEAD `94e54782a613d6ce2ba627e17e930f3107652530` on branch `ssot-bootstrap`.

## Repository scope

- **Language**: Swift 6 (Swift Package).
- **Files**: 414 Swift files across `Sources/`, `Tests/`, `Scripts/`.
- **LOC**: ~109,464 across all Swift sources.
- **Build targets**: 4 — `FluidAudio` (main library), `FluidAudioCLI` (benchmark + CLI), `FastClusterWrapper` (C++ bridge), `MachTaskSelfWrapper` (C bridge).
- **Models**: All Core ML, auto-downloaded from HuggingFace (`FluidInference/*`). Inference targets the Apple Neural Engine (ANE).

## Existing in-repo documentation (used as ground truth)

| Path | What it covers | Used for |
|---|---|---|
| `README.md` | Top-level capability summary, showcase, model list. | Subsystem identification. |
| `AGENTS.md` | Build/test commands, architecture summary, agent rules. | Repo-wide invariants. |
| `CLAUDE.md` | Critical dev rules, repo-specific conventions (`@MainActor`, no `@unchecked Sendable`). | Cross-cutting concerns. |
| `CONTRIBUTING.md` | Contribution guidelines. | Conventions. |
| `benchmarks.md` | Performance numbers per pipeline. | Performance metrics on relevant features. |
| `Documentation/Architecture.md` | High-level system architecture. | Overview anchor. |
| `Documentation/API.md` | Public-API surface. | Public-feature surface. |
| `Documentation/Models.md` | Model catalog + RTFx. | Model registry feature. |
| `Documentation/ASR/*.md` | Per-ASR-model getting-started, post-processing, custom vocab. | ASR features. |
| `Documentation/Diarization/*.md` | Diarizer concepts: timeline, speaker manager, LS-EEND, Sortformer. | Diarizer features + flows. |
| `Documentation/VAD/*.md` | Silero VAD + segmentation. | VAD feature. |
| `Documentation/TTS/*.md` | TTS engine docs: Kokoro, Magpie, PocketTTS, SSML, CosyVoice3. | TTS features. |
| `Documentation/Guides/AudioConversion.md` | Audio format conversion. | Shared audio utilities. |
| `Documentation/CtcDecoderGuide.md` | CTC decoder walkthrough. | ASR Parakeet CTC feature. |
| `Documentation/ModelConversion.md` | Conversion pipeline (out-of-scope but referenced). | Out of scope. |
| `Documentation/CLI.md` | CLI subcommands. | CLI feature stubs. |

## Code-tree structure

```
Sources/
├── FluidAudio/                      # main library (~56k LOC)
│   ├── ASR/                         # 63 files, 15.2k LOC
│   │   ├── Cohere/                  # Cohere Transcribe 03-2026 pipeline
│   │   ├── Parakeet/                # TDT/CTC + streaming EOU + Nemotron
│   │   │   ├── SlidingWindow/       # batch transcription
│   │   │   │   ├── CTC/             # CTC decoder
│   │   │   │   ├── CustomVocabulary/# BK-tree + word-spotting + rescorer
│   │   │   │   └── TDT/             # TDT decoder
│   │   │   ├── Streaming/           # streaming ASR
│   │   │   │   ├── EOU/             # end-of-utterance
│   │   │   │   └── Nemotron/        # Nemotron streaming
│   │   │   └── TokenDeduplication/  # cross-chunk dedup
│   │   ├── Qwen3/                   # Qwen3-ASR variant + RoPE + streaming
│   │   └── Shared/                  # PunctuationCommitLayer
│   ├── Diarizer/                    # 34 files, 13.4k LOC
│   │   ├── Clustering/              # SpeakerManager (online)
│   │   ├── Core/                    # DiarizerManager
│   │   ├── Extraction/              # EmbeddingExtractor
│   │   ├── LS-EEND/                 # LS-EEND online diarizer
│   │   ├── Sortformer/              # Sortformer online diarizer
│   │   ├── Segmentation/            # SlidingWindow + AudioValidation
│   │   └── Offline/                 # Offline batch pipeline
│   │       ├── Clustering/          # AHC, KMeans, VBx clustering
│   │       ├── Core/                # OfflineDiarizerManager
│   │       ├── Extraction/          # offline embedding extractor
│   │       ├── Segmentation/        # offline segmentation
│   │       └── Utils/               # helpers
│   ├── VAD/                         # 4 files, 961 LOC — Silero VAD
│   ├── TTS/                         # 107 files, 21.4k LOC
│   │   ├── CosyVoice3/              # CosyVoice3 0.5B
│   │   ├── G2P/                     # multilingual G2P (charsiu-byt5)
│   │   ├── Kokoro/                  # Kokoro 82M
│   │   ├── KokoroAne/               # Kokoro ANE-optimized variant
│   │   ├── Magpie/                  # Magpie 357M multilingual TTS
│   │   ├── PocketTTS/               # PocketTTS streaming
│   │   ├── SSML/                    # SSML parser + interpreter
│   │   ├── Shared/                  # AudioPostProcessor + compute-unit presets
│   │   └── StyleTTS2/               # StyleTTS-2
│   ├── ITN/                         # 1 file, 403 LOC — TextNormalizer
│   ├── Shared/                      # 22 files, 3.6k LOC — ANE utils, AudioConverter, mel-spec, registry
│   ├── DownloadUtils.swift          # HuggingFace download
│   ├── ModelNames.swift             # Repo enum
│   ├── ModelRegistry.swift          # registry URL config (HF + override)
│   └── FluidAudioSwift.swift        # umbrella + back-compat aliases
├── FluidAudioCLI/                   # 25.2k LOC — benchmarking + CLI
├── FastClusterWrapper/              # C++ bridge for fast clustering
└── MachTaskSelfWrapper/             # C bridge for mach_task_self
Tests/FluidAudioTests/               # XCTest suite
```

## Documentation gaps observed (to be captured during feature.md generation)

- `Sources/FluidAudio/ASR/Cohere/` has no per-model doc in `Documentation/ASR/` (only a `Documentation/ASR/Cohere.md` exists, which is at root of the ASR docs).
- `Sources/FluidAudio/Diarizer/Sortformer/` has a long-form doc but no per-API reference.
- `Sources/FluidAudio/TTS/StyleTTS2/` has source but no `Documentation/TTS/StyleTTS2.md` entry — confirm in self-review.
- `Sources/FluidAudio/Shared/ANEMemoryOptimizer.swift` is critical to performance but has no top-level doc beyond inline comments.
- `Sources/FluidAudio/ITN/TextNormalizer.swift` references `text-processing-rs` externally but has no in-repo design doc.

## Bootstrap depth strategy

FluidAudio (414 files / ~109k LOC) is on the same scale as audria-backend (~81k LOC). Per the audria-backend bootstrap precedent, we apply the **three-tier depth strategy**:

- **Full depth (~150 lines, onDeviceASR style)**: the architectural bedrock — manager classes that are the entry points for each subsystem (`DiarizerManager`, `OfflineDiarizerManager`, `SortformerDiarizer`, `LSEENDDiarizer`, `VadManager`, `SlidingWindowAsrManager`, `StreamingAsrManager`, `Qwen3AsrManager`, `CoherePipeline`, `KokoroTtsManager`, `MagpieTtsManager`, `PocketTtsManager`, `CosyVoice3TtsManager`, plus `ModelRegistry`, `DownloadUtils`, `AudioConverter`, `SpeakerManager`). ~17 features.
- **Compact (~50 lines)**: per-pipeline support modules — decoders (TDT/CTC/RNNT), clustering (AHC/KMeans/VBx), G2P, SSML, EmbeddingExtractor, TokenDeduplication, EncoderCacheManager, AudioMelSpectrogram, ANE memory utils, ProgressEmitter, AssetDownloader, etc. ~55 features.
- **Stub (~20 lines)**: thin wrappers, types-only files, and the CLI which is bundled into one `fluidaudio-cli-bundle` feature. ~13 features.

## Total expected output

- ~85 feature.md files.
- ~20 flow.md files.
- 1 overview.
- 1 self-review.
- This `_setup/` set.

Pinned commit: `94e54782a613d6ce2ba627e17e930f3107652530`.
