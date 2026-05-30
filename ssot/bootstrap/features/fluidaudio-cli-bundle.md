---
id: fluidaudio-cli-bundle
name: FluidAudio CLI Bundle
repo: FluidAudio
status: active
linked_features: [kokoro-tts-manager, magpie-tts-manager, pocket-tts-manager, cosyvoice3-tts-manager, kokoro-ane, style-tts2, tts-compute-unit-preset]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# FluidAudio CLI Bundle

## TL;DR
~25k LOC `swift run fluidaudiocli` command-line tool that exercises every FluidAudio backend (ASR, diarization, VAD, TTS, dataset parsing, benchmarks). Subcommand semantics live in `Documentation/CLI.md`; this stub points at the source tree.

## Motivation — why it exists

## Context

## What It Does
The CLI lives under `Sources/FluidAudioCLI/` and ships subcommands for transcription (`transcribe`, `qwen3-transcribe`, `parakeet-eou`, `multi-stream`), TTS (`tts`, `tts-asr-verify`, `tts-benchmark`, plus backend-specific commands `MagpieCommand`, `StyleTTS2Command`, `CosyVoice3/`), benchmarking (`asr-benchmark`, `qwen3-benchmark`, `fleurs-benchmark`, `g2p-benchmark`, `ctc-earnings-benchmark`, `DiarizationBenchmark`, `LSEENDBenchmark`, `SortformerBenchmark`, `VadBenchmark`), diarization (`SortformerCommand`, `LSEENDCommand`), dataset parsing (`DatasetParsers/`), and asset downloads (`DownloadCommand`).

## Key Code
- [`Sources/FluidAudioCLI/FluidAudioCLI.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudioCLI/FluidAudioCLI.swift) — top-level entry point.
- [`Sources/FluidAudioCLI/Commands/TTSCommand.swift`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudioCLI/Commands/TTSCommand.swift) — primary `tts` subcommand.
- [`Documentation/CLI.md`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Documentation/CLI.md) — canonical subcommand reference.

## Edge Cases & Failure Modes
- See `Documentation/CLI.md` for per-subcommand flags and exit codes.

## Test Coverage
- `Tests/FluidAudioTests/CLI/` covers argument parsing + a handful of integration paths.

## Changelogs
