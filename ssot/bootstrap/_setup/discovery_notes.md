# FluidAudio — Discovery Notes

Generated 2026-05-29 at commit `94e54782a613d6ce2ba627e17e930f3107652530`.

## What this file is

Working notes from the feature/flow enumeration pass that don't belong in `features.txt` or `flows.txt` themselves but should be preserved for the SSOT maintainer.

## Scale calibration

FluidAudio is a **414-file / ~109k-LOC Swift package**, comparable to audria-backend (~81k LOC) and far beyond onDeviceASR (13 files / 2k LOC). Per the audria-backend precedent, this bootstrap uses **three-tier depth**:

- **Full depth (~17 features)**: top-level managers and the architectural bedrock (ModelRegistry, DownloadUtils, AudioConverter, ANE memory layer, the entry-point manager for each pipeline).
- **Compact (~58 features)**: support modules — decoders, clustering algorithms, G2P, SSML, embedding extraction, per-pipeline support files.
- **Stub (~3 features)**: thin/back-compat files plus the CLI bundle (25k LOC under one `fluidaudio-cli-bundle` entry).

An onDeviceASR-equivalent per-method bootstrap would produce ~250+ feature.md files, not feasible in a single session.

## Subsystem boundaries

Six subsystems are clearly separable:

1. **ASR** — Parakeet (TDT/CTC + streaming + EOU + Nemotron), Cohere, Qwen3.
2. **Diarizer** — Online (DiarizerManager, SpeakerManager) + Sortformer + LS-EEND + Offline batch.
3. **VAD** — Silero.
4. **TTS** — Kokoro, KokoroAne, Magpie, PocketTTS, CosyVoice3, StyleTTS2 + Shared (TtsBackend, AudioPostProcessor, SSML, multilingual G2P).
5. **ITN** — Text normalization.
6. **Shared** — ANE memory, AudioConverter, AudioMelSpectrogram, ModelRegistry, DownloadUtils, MLArrayCache, ZeroCopyFeatureProvider, AssetDownloader, PerformanceMetrics, SystemInfo, etc.

Cross-cutting: every model-loading manager depends on `ModelRegistry` + `DownloadUtils`. Every predict path depends on `ANEMemoryOptimizer` + `MLModel+Prediction`. Every audio entry point depends on `AudioConverter`.

## Key concurrency observations

- The repo invariant (in `AGENTS.md` / `CLAUDE.md`) **forbids `@unchecked Sendable`**. Verify this holds across all sources during the feature pass — any `@unchecked Sendable` is a self-review finding.
- `ModelRegistry` uses `nonisolated(unsafe) private static var _customBaseURL` (Swift 6 escape hatch). This is documented as a "set-once at startup" assumption in the source comment, but there's no runtime guard. Flag in self-review as a controlled-but-unenforced invariant.

## Known TODOs / inline review markers in source (sampling)

- `ModelNames.swift` has a comment noting that Japanese CTC-only inference was removed in commit `846924a1d`; the preprocessor+encoder are reused from the Japanese model. Worth a feature.md note under `parakeet-language-models`.
- `Diarizer/Sortformer/SortformerStateUpdater` has state that lives outside actor isolation in places — verify thread-safety claims during the full-depth pass.
- `TTS/Magpie/MagpieTtsManager` documented as "quite slow (~0.04 RTFx, ~25× slower than realtime) and needs further perf work" in the README. Flag as a performance review item.

## Per-engine TTS shape similarity

Every TTS engine follows a near-identical surface:

```
<Engine>TtsManager
  ├── synthesize(text:) async throws -> Audio
  ├── load() async throws
  └── unload() async
<Engine>Pipeline/
<Engine>Models
<Engine>Constants
<Engine>Error
Assets/   (JSON config)
```

This is deliberately uniform per the `AGENTS.md` rule "make sure that the API is consistent with the other model managers". Implication for documentation: per-engine compact features can cross-reference a shared `tts-backend-protocol` feature.md rather than duplicating the surface description.

## Decoder shape similarity (ASR)

The TDT decoder, CTC decoder, and RNNT decoder share the same skeleton — encoder output → step loop → token emission. Where they differ is the step rule (TDT: durations; CTC: blank collapsing; RNNT: predictor + joint). Per-decoder compact features capture the step rule precisely; cross-reference rather than duplicate the encoder-cache shape.

## Diarizer split

There are **four distinct diarizer implementations** in one folder:

1. **DiarizerManager** (online) — segmentation + speaker embedding + speaker manager.
2. **SortformerDiarizer** — Sortformer encoder + state updater.
3. **LSEENDDiarizer** — LS-EEND inference + preprocessor.
4. **OfflineDiarizerManager** — offline batch with AHC/KMeans/VBx clustering.

They share nothing except the embedding extractor in some paths. All four merit full-depth manager features.

## Out-of-scope (per `exclusion_list.md`)

- Tests, Scripts, ThirdPartyLicenses, Documentation/*.md, Package.swift, FluidAudio.podspec, Asset JSON blobs.
- FluidAudioCLI is bundled into one stub; per-subcommand expansion deferred.

## What this bootstrap intentionally does NOT cover

- Per-decoder-method code walkthrough (deferred per scale calibration).
- Per-TTS-engine pipeline step audit (covered at compact-tier shape, not per stage).
- Conversion pipeline (`Documentation/ModelConversion.md` — out of scope, separate repo concerns).
- C++ FastCluster algorithm internals (only the Swift bridge is documented).

## Next steps after this enumeration

1. Generate full-depth feature.md for the 17 listed.
2. Generate compact feature.md for the 58 listed.
3. Generate stub feature.md for the 3 listed.
4. Generate 23 flow.md files.
5. Generate `FluidAudio_overview.md`.
6. Generate `_self_review.md`.
7. File Jira tickets for findings.
