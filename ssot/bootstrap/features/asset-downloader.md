---
id: asset-downloader
name: Asset Downloader
repo: FluidAudio
status: active
linked_features: [download-utils]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Asset Downloader

## TL;DR
Generic single-asset fetcher used by TTS / G2P paths to download individual JSON / binary sidecar files (voice packs, vocab files, runtime embeddings) outside the main `DownloadUtils.loadModels` flow. Supports both `Data`-mode (in-memory then atomic write) and file-mode (`URLSession.download` + move) transfers, with skip-if-exists by default.

## Motivation — why it exists

## Context

## What It Does
A caller fills in a `Descriptor` (`description`, `remoteURL`, `destinationURL`, `skipIfExists`, `transferMode`) and calls `ensure(_:)`. The helper checks for the destination file, creates parent directories, performs the transfer, and validates the HTTP status code (`200..<300`). `TransferMode.data(DataWriter)` defaults to atomic write; `TransferMode.file(FileMover)` defaults to a "remove-existing then move" pattern. `fetchData(from:description:)` is a fire-and-forget byte fetcher that does not write to disk — used by callers that immediately parse the response.

## Key Code
- [`Sources/FluidAudio/Shared/AssetDownloader.swift:11`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AssetDownloader.swift#L11) — `defaultDataWriter` — atomic `Data.write(to:options:.atomic)`
- [`Sources/FluidAudio/Shared/AssetDownloader.swift:15`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AssetDownloader.swift#L15) — `defaultFileMover` — remove-existing + `moveItem`
- [`Sources/FluidAudio/Shared/AssetDownloader.swift:60`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AssetDownloader.swift#L60) — `ensure(_:session:logger:)` — main entry with skip-if-exists + status check
- [`Sources/FluidAudio/Shared/AssetDownloader.swift:99`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AssetDownloader.swift#L99) — `fetchData` — in-memory fetch, no write

## Edge Cases & Failure Modes
- Status check rejects any non-2xx as `invalidResponse(description:statusCode:)` — no retry, no exponential backoff (unlike `DownloadUtils.fetchHuggingFaceFile`).
- Reuses `DownloadUtils.sharedSession` by default — inherits the HuggingFace base URL + proxy configuration from `ModelRegistry`. Callers passing a custom session bypass that.
- `skipIfExists = true` skips even if the existing file is partial / corrupted — no checksum validation.
- HTML-error-page sniffing (`DownloadUtils.validateJSONResponse`) is **not** applied here, so a `200 OK` HTML page from HuggingFace under rate limit would be written to disk unchanged. [REVIEW: `AssetDownloader.ensure` does not sniff HTML error responses — could persist garbage when HF is rate-limiting]
- No `HF_TOKEN` auth header on these requests — relies on the shared session's default config.

## Test Coverage
No dedicated tests; exercised by TTS resource downloaders.

## Changelogs
