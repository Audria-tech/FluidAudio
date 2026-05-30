# FluidAudio — Exclusion List

Generated 2026-05-29 at commit `94e54782a613d6ce2ba627e17e930f3107652530`.

The following paths are intentionally **out of scope** for this bootstrap. They are excluded from `features.txt`, `flows.txt`, and the Graphify run.

## Tests
- `Tests/FluidAudioTests/**` — test code. Coverage is summarized inside the relevant feature's `Test Coverage` section rather than as standalone features.

## Build / packaging / vendor
- `Package.swift` — Swift PM manifest; build-system, not application logic.
- `FluidAudio.podspec` — CocoaPods manifest.
- `Sources/FastClusterWrapper/**` — third-party C++ algorithm vendor (FastCluster), exposed via a thin Swift bridge; bridge is documented as part of clustering features but the C++ source itself is not.
- `Sources/MachTaskSelfWrapper/**` — single-call C bridge to `mach_task_self()`; documented as part of `system-info` shared feature.
- `ThirdPartyLicenses/` — license texts only.

## Scripts / dev tooling
- `Scripts/**` — repo dev tooling (formatting, benchmarks, conversion helpers).
- `.swift-format`, `.gitattributes`, `.gitignore`.

## Long-form docs (referenced, not feature-documented)
- `Documentation/Architecture.md`, `Documentation/Models.md`, `Documentation/API.md`, `Documentation/Benchmarks.md`, `Documentation/CLI.md`, `Documentation/ModelConversion.md`, `Documentation/CtcDecoderGuide.md`, `Documentation/ASR/*.md`, `Documentation/Diarization/*.md`, `Documentation/TTS/*.md`, `Documentation/VAD/*.md`, `Documentation/Guides/*.md` — referenced from features but not feature.md targets themselves.
- `benchmarks.md` (root) — referenced from performance sections only.
- `README.md`, `AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md`, `CITATION.cff` — meta-docs, used as ground truth and referenced from `Cross-Cutting Concerns`.

## CLI bundling
- `Sources/FluidAudioCLI/**` — the entire CLI (25.2k LOC, ~80 files of subcommands and benchmark drivers) is captured under a **single** `fluidaudio-cli-bundle` feature stub. Each subcommand could merit its own feature; deferred to a future per-CLI-command pass.

## Asset blobs
- `Sources/FluidAudio/TTS/*/Assets/**` — vendored model assets (JSON config, phoneme dicts). Referenced from the relevant TTS feature; not feature-documented as standalone.

## Deprecated / legacy
- None identified at this commit; will be surfaced in `_self_review.md` if discovered.

## Banner / images
- `banner.png`, `Documentation/assets/**`, `Documentation/VAD/*.png` — images only.
