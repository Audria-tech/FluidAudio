---
id: download-utils
name: Download Utils
repo: FluidAudio
status: active
linked_features: [model-registry, model-names, asset-downloader, system-info]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Download Utils

## TL;DR
Top-level downloader that resolves a `Repo` to a local directory of `.mlmodelc` bundles plus auxiliary JSON/BIN files, loads them as `MLModel` instances, and reports byte-weighted progress. Every cross-cutting download in FluidAudio (ASR, VAD, Diarizer, TTS) goes through this entry point.

## Motivation — why it exists

## Context

## What It Does
`DownloadUtils.loadModels` is the public happy-path: it checks the on-disk cache, downloads any missing files via the HuggingFace tree-listing + per-file fetch dance, compiles each `.mlmodelc` into an `MLModel`, and returns a `[name: MLModel]` map. If the first attempt fails it deletes the cached repo folder and retries once — a hard-coded one-shot recovery from corrupted partial downloads.

The downloader walks `Repo.remotePath` via the HF `tree/main` API, recursively listing only directories that intersect the `requiredModels` patterns (resolved from `ModelNames.getRequiredModelNames(for:variant:)`). Each candidate file is filtered against the required-model set plus a hard-coded suffix allowlist (`.json`, `.txt`, `.bin`, `.model`) so README files etc. don't get pulled. Repos with a `subPath` (e.g. `parakeetEou160` → `160ms/`) have the prefix stripped on the local side so the on-disk layout mirrors `requiredModels`.

Progress is byte-weighted when total file sizes are known: the listing phase is the first 0%–0% tick, downloads span 0.0→0.5 of the overall progress range, compilation spans 0.5→1.0. For per-file byte progress the downloader spins up a private `URLSession` with a `DownloadProgressDelegate` that forwards `didWriteData` callbacks; this avoids cross-talk between concurrent downloads sharing the global session.

Rate-limiting (`429`/`503`) and HTML-error-page responses (HuggingFace sometimes returns `200 OK` with an HTML rate-limit page) are both promoted to thrown errors. `fetchHuggingFaceFile` adds bounded retry with exponential backoff (`2^n * minBackoff`, default 4 attempts) for ad-hoc single-file fetches. Auth tokens are pulled from `HF_TOKEN`, `HUGGING_FACE_HUB_TOKEN`, or `HUGGINGFACEHUB_API_TOKEN` (whichever is set) and attached as a `Bearer` header.

## Key Code
- [`Sources/FluidAudio/DownloadUtils.swift:10`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L10) — `sharedSession` — single `URLSession` reused by every download path
- [`Sources/FluidAudio/DownloadUtils.swift:17`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L17) — `huggingFaceToken` — multi-env-var token resolution
- [`Sources/FluidAudio/DownloadUtils.swift:43`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L43) — `validateJSONResponse` rejects HTML payloads as `htmlErrorResponse`
- [`Sources/FluidAudio/DownloadUtils.swift:77`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L77) — `DownloadPhase` (`listing` / `downloading` / `compiling`)
- [`Sources/FluidAudio/DownloadUtils.swift:115`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L115) — `loadModels` — public entry; once-only retry-after-cache-wipe
- [`Sources/FluidAudio/DownloadUtils.swift:167`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L167) — `clearAllModelCaches` — nukes ApplicationSupport + TTS caches
- [`Sources/FluidAudio/DownloadUtils.swift:191`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L191) — `loadModelsOnce` — cache-check → download → compile per-model
- [`Sources/FluidAudio/DownloadUtils.swift:247`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L247) — `coremldata.bin` existence check — sentinel for "compiled bundle, not a stub directory"
- [`Sources/FluidAudio/DownloadUtils.swift:280`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L280) — `downloadRepo` — main listing + per-file download loop
- [`Sources/FluidAudio/DownloadUtils.swift:307`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L307) — `listDirectory` — recursive HF `tree/main` walk with pattern filtering
- [`Sources/FluidAudio/DownloadUtils.swift:401`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L401) — 0-byte file workaround (HF returns 500 for empty files; locally create an empty file)
- [`Sources/FluidAudio/DownloadUtils.swift:497`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L497) — `createDirectoryRobustly` — recovers from "file where directory should be" corruption
- [`Sources/FluidAudio/DownloadUtils.swift:540`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L540) — `downloadWithProgress` — dedicated session + `DownloadProgressDelegate`
- [`Sources/FluidAudio/DownloadUtils.swift:572`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L572) — `downloadSubdirectory` — opt-in pull of a single subdir (e.g. PocketTTS Mimi encoder)
- [`Sources/FluidAudio/DownloadUtils.swift:693`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L693) — `fetchHuggingFaceFile` — single-file fetch with exponential-backoff retry
- [`Sources/FluidAudio/DownloadUtils.swift:742`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L742) — `DownloadProgressDelegate` — `didWriteData → onProgress` shim

## Edge Cases & Failure Modes
- `HuggingFaceDownloadError.rateLimited` — promoted from any `429`/`503` response in either listing or download phases. Not retried inside `downloadRepo`; the outer "delete-and-retry-once" catches it.
- `HuggingFaceDownloadError.htmlErrorResponse` — HF returns `200 OK` with an HTML rate-limit page on overload; the `<` / `<!doctype html` sniff rejects them before JSON parsing.
- `HuggingFaceDownloadError.modelNotFound` — thrown only at the post-download verification step; means a required model was missing from the listing entirely.
- 0-byte files trigger HTTP 500 on HF — the downloader skips the request and creates the empty file locally (see line 401).
- `loadModels` will silently delete the entire cached repo folder if the first load fails. Concurrent loaders of the same `Repo` would race on this wipe.
- `createDirectoryRobustly` removes any file blocking a needed directory path — destructive on corrupted caches but necessary for the retry path to succeed.
- Timeout defaults to 30 minutes (`DownloadConfig.timeout = 1800`) — large models on slow links may still hit it.
- `coremldata.bin` is the in-bundle sentinel for a real compiled `.mlmodelc`. A directory missing it throws `fileReadCorruptFile`.
- Auth tokens are picked from any of three env vars in fall-through order; if a stale token is set in any of them the request will 401 with no fallback.
- `clearAllModelCaches` uses a hardcoded `~/.cache/fluidaudio/Models` path on macOS but `~/Library/Caches/fluidaudio/Models` on iOS — split paths could leak storage if the user moves data between platforms.

## Performance / Concurrency Notes
- `sharedSession` is a `static let` initialised lazily via `ModelRegistry.configuredSession()` — one `URLSession` for the entire process.
- Per-byte progress sessions are dedicated (`URLSession` per download) to avoid cross-talk between `URLSessionDownloadDelegate` callbacks. They are explicitly finished via `finishTasksAndInvalidate()` in a `defer`.
- Downloads are sequential, not concurrent — large repos download one file at a time. No throttling across concurrent `loadModels` calls.
- Progress weighting: known sizes get byte-weighted progress; files with size `-1` are weighted as 0 (under-report) and progress falls back to file-count weighting when `totalBytes == 0`.
- `validateJSONResponse` decodes the entire body as UTF-8 to sniff the first character — for very large JSON listings this is wasteful but bounded by HF directory size.

## Test Coverage
No dedicated `DownloadUtilsTests.swift`; behaviour is exercised end-to-end via `Tests/FluidAudioTests/CI/BasicInitializationTests.swift` and module-specific integration tests (ASR, Diarizer) that load real models.

## Changelogs
