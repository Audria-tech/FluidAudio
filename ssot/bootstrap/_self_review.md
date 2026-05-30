# SSOT Bootstrap — Self Review (FluidAudio)

Generated 2026-05-29 by Claude (Cowork autonomous session) at HEAD `94e54782a613d6ce2ba627e17e930f3107652530` on branch `ssot-bootstrap`.

## Scope acknowledgment

FluidAudio is **414 Swift files / ~109,464 LOC** — slightly larger than audria-backend (~81k LOC) and roughly 55× the size of onDeviceASR. A per-method-granularity bootstrap at the depth used for onDeviceASR would produce 250+ feature.md files and is not feasible in a single autonomous session.

This bootstrap is deliberately calibrated at **three depth tiers** (mirroring the audria-backend strategy):

- **Full depth** (~120-180 line files matching onDeviceASR style): the architectural bedrock — top-level managers and the cross-cutting perf layer. Includes `model-registry`, `download-utils`, `model-names`, `ane-memory-optimizer`, `audio-converter`, `audio-mel-spectrogram`, `sliding-window-asr-manager`, `streaming-asr-manager`, `cohere-asr-pipeline`, `qwen3-asr-manager`, `diarizer-manager`, `speaker-manager`, `embedding-extractor`, `sortformer-diarizer`, `lseend-diarizer`, `offline-diarizer-manager`, `vad-manager`, `kokoro-tts-manager`, `magpie-tts-manager`, `pocket-tts-manager`, `cosyvoice3-tts-manager`, `tts-backend-protocol`, `multilingual-g2p`, `ssml-processor`, `text-normalizer`. ~25 features.
- **Compact** (~40-60 lines with key code anchors, error modes, edge cases): every per-pipeline support module — decoders (TDT/CTC/RNNT), clustering (AHC/KMeans/VBx), G2P, SSML, embedding extractor, encoder cache, audio post-processor, etc. ~80 features.
- **Stub** (~15-25 lines anchoring to source + Code link): thin types, back-compat aliases, and the entire `FluidAudioCLI` target (25k LOC, ~80 subcommand files) bundled into one `fluidaudio-cli-bundle`. ~5 features.

## Inventory totals

- **110 feature.md files** in `ssot/bootstrap/features/`
- **25 flow.md files** in `ssot/bootstrap/flows/`
- `_setup/` artifacts: `doc_audit.md`, `exclusion_list.md`, `.graphifyignore`, `features.txt`, `flows.txt`, `discovery_notes.md`, `graphify_output/GRAPH_REPORT.md`
- SHA pinned: `94e54782a613d6ce2ba627e17e930f3107652530`

All code anchors are SHA-pinned permalinks against the commit above.

## Confidence assessment

### Full-depth features
**High** — code read end-to-end (managers, pipelines), edge cases concrete, REVIEW markers grounded in observed code.

### Compact features
**Medium-high** — module purpose accurate, key code anchors correct, error modes capture the obvious failure paths. Some edge cases / deeper concerns flagged with `[REVIEW:]` markers rather than fully resolved.

### Stub features
**Medium** — accurate at the module level. The CLI bundle stub is intentionally not split; per-subcommand follow-up split needed.

### Flows
**High** — flow shape accurate, per-stage code anchors present for every step. Streaming-ASR variants and offline-diarization are at full detail.

## REVIEW markers — recurring themes

Across the 110 features, the most-repeated `[REVIEW:]` markers cluster around:

1. **`nonisolated(unsafe)` static mutables with set-once contracts not runtime-enforced** — `ModelRegistry._customBaseURL`, `AppLogger.defaultSubsystem`, `OfflineDiarizerManager.models`. All documented in source as "set once at startup before concurrent access," but no compile- or runtime-time guard.
2. **Thread safety relying on hand-rolled locking, not actor isolation** — `SortformerDiarizer` uses an `NSLock` wrapping all public methods; future methods that forget `lock.withLock` would race silently. `SortformerStateUpdater` is a value-type struct with no isolation; safety lives entirely at the caller boundary. `KokoroTtsManager` is a class (not actor) while every other backend manager is an actor.
3. **Stringly-typed dispatchers with silent fall-through** — `ModelNames.getRequiredModelNames` takes a `variant: String?`; wrong sentinel silently returns a single-filename default. `Qwen3AsrManager` and `Qwen3AsrInt8` both route to `requiredModelsFull` — `Qwen3ASR.requiredModels` is dispatcher-unreachable.
4. **Compatibility / version checks behind `assert(...)` only** — `Sortformer.bundle(for:)` uses `assert` for variant compatibility; release builds silently load incompatible variants.
5. **Force-cast and force-unwrap in hot init paths** — `parakeet-tokenizer.swift` uses `as!` for JSON cast; `MLModelConfigurationUtils.defaultModelsDirectory` force-unwraps `FileManager.urls(...)`.
6. **Cross-module / cross-file dependencies that look local** — `MLArrayCache.swift` calls `resetData(to:)` defined as an `MLMultiArray` extension inside the TDT decoder file. Naming collision risk with the Shared `reset(to:)` which silently no-ops on non-fp32/int32 dtypes.
7. **Behaviors that don't match docstrings** —
   - `Qwen3StreamingManager` docstring says "sliding window" but re-runs the full offline transcribe on a growing buffer.
   - `SpeakerOperations.reassignSegment` docstring promises to recalculate main embeddings; implementation does not.
   - `SpeakerManager.assignSpeaker` accepts a `confidence` parameter that is never used.
   - `TextNormalizer.filterAmbiguousWords` is documented as wrapping in passthrough markers, but both branches return the original word (no-op).
8. **Empty / phantom results that callers may misread** — `AudioMelSpectrogram` returns `numFrames: 1` (not 0) on empty/short input. `EmbeddingExtractor.fillWaveformBuffer` writes only slot 0 of a 3-speaker batch; slots 1 and 2 retain whatever the padding overlapped.
9. **Performance flags** — Magpie at 0.04 RTFx (25× slower than realtime, README-acknowledged "needs perf work"). StyleTTS2 at ~44% WER vs Kokoro 1.3%. CosyVoice3 RTFx < 1.0 on M-series.
10. **Network fetch hardening gaps** — `AssetDownloader.ensure` does not sniff HTML error bodies; HuggingFace rate-limit HTML could persist as model data.

## Where reviewers should focus first

Highest-leverage findings that warrant Jira tickets, ordered by severity:

1. **`TextNormalizer.filterAmbiguousWords` is a no-op** — both branches return the original word, despite docstring promising disambiguation. Source: [ITN/TextNormalizer.swift](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ITN/TextNormalizer.swift).
2. **`Qwen3StreamingManager` does not stream** — re-runs `Qwen3AsrManager.transcribe` on a growing buffer with a rolling cap, despite docstring describing a sliding-window streaming approach. This has both a correctness implication (different output trajectory than true streaming) and a perf implication (quadratic work).
3. **`SortformerDiarizer` thread safety is hand-rolled `NSLock`-on-public-methods** — easy to break in future PRs. Should be moved to actor isolation per the repo's `@unchecked Sendable` ban.
4. **`SpeakerOperations.reassignSegment` does not recalculate main embeddings** — docstring promises it does. This is a data-integrity issue: speakers' main embeddings drift from their true centroid after reassignment.
5. **`SpeakerManager.assignSpeaker` ignores its `confidence` parameter** — dead parameter; callers may assume the value affects behavior.
6. **`AssetDownloader.ensure` does not validate response Content-Type / signature** — a HuggingFace 429/HTML response can persist as garbage in the model cache, causing cryptic load failures on next run.
7. **`ModelNames.getRequiredModelNames` dispatcher has dispatcher-unreachable cases** — `Qwen3ASR.requiredModels` (the 3-model set) is never reached because both `qwen3Asr` and `qwen3AsrInt8` route to `requiredModelsFull`.
8. **`Sortformer.bundle(for:)` variant-compatibility check is `assert`-only** — release builds silently load incompatible variants.
9. **`nonisolated(unsafe)` static mutables (`ModelRegistry._customBaseURL`, `AppLogger.defaultSubsystem`) have no runtime guard against late mutation** — set-once contract documented in comments but unenforced; concurrent late writes would race silently in Swift 6.
10. **`KokoroTtsManager` is a class (not actor) with mutable state** — diverges from the every-other-backend pattern.

(Performance items — Magpie 0.04 RTFx, StyleTTS2 elevated WER, CosyVoice3 RTFx < 1.0 — are acknowledged in source/docs and are not separately ticketed.)

## LEAVE-BLANK section discipline

All feature.md and flow.md files emit only the section header (no body) for `Motivation — why it exists`, `Context`, `Changelogs` (both), plus `Prompts and Models Used` and `Usage Metrics` (flows only). Reserved for human + sync-pipeline.

## Repo-modification confirmation

Every file written by this bootstrap lives under `ssot/bootstrap/`:
- `ssot/bootstrap/features/*.md` × 110
- `ssot/bootstrap/flows/*.md` × 25
- `ssot/bootstrap/_setup/...`
- `ssot/bootstrap/_self_review.md` (this file)
- `ssot/bootstrap/FluidAudio_overview.md`

No source code, `Package.swift`, `Documentation/`, or existing top-level markdown was modified.

## What is intentionally out of scope

- **Per-CLI-subcommand features**. The 25k LOC `FluidAudioCLI` target is bundled as `fluidaudio-cli-bundle`. Splitting would more than double the feature.md count; deferred until a specific subcommand warrants its own page. See `Documentation/CLI.md` for the user-facing CLI.
- **Tests** (`Tests/FluidAudioTests/`). Listed in `Test Coverage` sections of the relevant features rather than as standalone.
- **One-off scripts** (`Scripts/`). Out per `exclusion_list.md`.
- **Long-form docs** (`Documentation/*.md`). Referenced when relevant; not feature-documented.
- **FastCluster C++ algorithm internals**. Only the Swift bridge is documented (`fastcluster-bridge`).
- **`mach_task_self()` C bridge** — documented as part of `system-info` shared feature, not separately.

## Trade-offs the reader should know

This bootstrap optimizes for **coverage breadth + actionability of Jira findings** over per-method depth. A second pass focused on any single subsystem (ASR streaming pipeline, offline diarizer, or any single TTS engine) would produce per-method granularity.

The full-depth features (the 25 listed above) are at the same quality as onDeviceASR's 33. The compact tier is honest about what's documented vs flagged. The stub tier is explicit — pointers + intent + key code anchor.

## Comparison with prior bootstraps

| Aspect | onDeviceASR | audria-backend | FluidAudio |
|---|---|---|---|
| Repo size | 13 files / 2k LOC | 280 files / ~81k LOC | 414 files / ~109k LOC |
| Language | Swift | Python | Swift |
| Features generated | 33 | ~85 | 110 |
| Flows generated | 10 | ~23 | 25 |
| Per-file average depth | ~150 lines | varies: 50–180 | varies: 25–180 |
| All features at full depth | Yes | No (3 tiers) | No (3 tiers) |
| Jira tickets to file | 6 | 7 | ~10 planned |

## Next steps for the SSOT maintainer

1. Import `ssot/bootstrap/features/*.md` to `audria_ssot/features/FluidAudio/`.
2. Import `ssot/bootstrap/flows/*.md` to `audria_ssot/flows/FluidAudio/`.
3. Import `FluidAudio_overview.md` to `audria_ssot/overview/FluidAudio.md`.
4. File the Jira tickets from the "Where reviewers should focus first" list (top ~10).
5. Schedule a second pass on whichever subsystem needs per-method depth (recommended: Streaming ASR or Offline Diarizer).
