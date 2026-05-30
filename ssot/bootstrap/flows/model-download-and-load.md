---
id: model-download-and-load
name: Model Download and Load
trigger_type: user-action
trigger: Any manager's `initialize()` / `loadModels()` / `downloadAndLoad()` invocation that needs CoreML weights on disk.
end_state: Every required `.mlmodelc` for the requested `Repo` is compiled on disk under `~/Library/Application Support/FluidAudio/Models/<folderName>/` (or the TTS `~/.cache/fluidaudio/Models/` root), an `[name: MLModel]` map is returned to the caller, and byte-weighted progress was reported through the `onProgress` closure.
involves_features: [download-utils, model-registry, model-names, asset-downloader, mlmodel-configuration-utils]
linked_flows: []
sync_async: async
typical_latency_ms: null
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Model Download and Load

## TL;DR
Resolves a `Repo` value to a cache directory of compiled `.mlmodelc` bundles, walks the HuggingFace tree-listing API to fetch any missing files, compiles each bundle, and returns the loaded `MLModel` instances. Every cross-cutting load path in FluidAudio (ASR, VAD, Diarizer, TTS, G2P) funnels through this flow exactly once per process per repo.

## Trigger & Preconditions
Triggered by `DownloadUtils.loadModels(_:variant:configuration:progressHandler:)` from every manager's initializer (e.g. `AsrModels.downloadAndLoad`, `VadManager.loadIfNeeded`, `DiarizerModels.download`, `TtsModels.download`, `SortformerModels.loadFromHuggingFace`, `LSEENDModel.loadFromHuggingFace`, `MultilingualG2PModel.loadIfNeeded`). Preconditions: network reachable on cold cache (or models already present); optional `HF_TOKEN` env var if the repo is gated; writable cache directory.

## Stages
1. **Resolve required file set** — `ModelNames.getRequiredModelNames(for:variant:)` translates the `Repo` (and optional string variant such as `"int8"`, `"offline"`, `"g2p-only"`, or a specific bundle name) into a `Set<String>` of expected files: [`Sources/FluidAudio/ModelNames.swift:1006`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelNames.swift#L1006).
2. **Resolve cache directory** — `MLModelConfigurationUtils.defaultModelsDirectory(for:)` returns `~/Library/Application Support/FluidAudio/Models/<Repo.folderName>/`: [`Sources/FluidAudio/Shared/MLModelConfigurationUtils.swift:25`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/MLModelConfigurationUtils.swift#L25).
3. **Configure shared session** — `ModelRegistry.configuredSession()` returns the once-per-process `URLSession`, wired with `https_proxy` / `http_proxy` env vars on macOS; `DownloadUtils.sharedSession` caches it: [`Sources/FluidAudio/ModelRegistry.swift:90`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelRegistry.swift#L90).
4. **Resolve base URL + auth** — `ModelRegistry.baseURL` picks the override → `REGISTRY_URL` → `MODEL_REGISTRY_URL` → `huggingface.co`; `DownloadUtils.huggingFaceToken` reads `HF_TOKEN` / `HUGGING_FACE_HUB_TOKEN` / `HUGGINGFACEHUB_API_TOKEN` for a Bearer header: [`Sources/FluidAudio/ModelRegistry.swift:32`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelRegistry.swift#L32), [`Sources/FluidAudio/DownloadUtils.swift:17`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L17).
5. **Try-load once** — `DownloadUtils.loadModels` calls `loadModelsOnce(...)`; on failure it deletes the cached repo directory and retries exactly once (single-shot corruption recovery): [`Sources/FluidAudio/DownloadUtils.swift:115`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L115).
6. **Cache-check** — For each required file, presence + a `coremldata.bin` sentinel inside the `.mlmodelc` bundle confirm a valid compiled package: [`Sources/FluidAudio/DownloadUtils.swift:247`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L247).
7. **Walk HF tree** — On any miss, `listDirectory` walks the `<base>/api/models/<repo>/tree/main/<sub>` JSON listing recursively, filtered against the required-model patterns and a hard-coded suffix allowlist (`.json`, `.txt`, `.bin`, `.model`): [`Sources/FluidAudio/DownloadUtils.swift:307`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L307).
8. **Per-file download** — `downloadWithProgress` spins up a dedicated `URLSession` per file with a `DownloadProgressDelegate` so byte-level progress callbacks don't cross-talk between concurrent transfers: [`Sources/FluidAudio/DownloadUtils.swift:540`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L540).
9. **Validate response** — `validateJSONResponse` rejects HuggingFace's HTML rate-limit pages (`200 OK` with `<!doctype html`) as `htmlErrorResponse`; `429` / `503` are promoted to `rateLimited`; zero-byte files (HF returns 500) get a local touch instead of a request: [`Sources/FluidAudio/DownloadUtils.swift:43`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L43), [`Sources/FluidAudio/DownloadUtils.swift:401`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L401).
10. **Compile + load** — `loadModelsOnce` constructs an `MLModelConfiguration` via `MLModelConfigurationUtils.defaultConfiguration(computeUnits:)` (`.cpuAndNeuralEngine`, `allowLowPrecisionAccumulationOnGPU = true`) and instantiates each `.mlmodelc` into an `MLModel`: [`Sources/FluidAudio/DownloadUtils.swift:191`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/DownloadUtils.swift#L191), [`Sources/FluidAudio/Shared/MLModelConfigurationUtils.swift:11`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/MLModelConfigurationUtils.swift#L11).
11. **Optional sidecar fetches** — TTS / G2P paths call `AssetDownloader.ensure(_:)` for voice packs, vocab `.json`, and runtime safetensors blobs (skip-if-exists + atomic write or move): [`Sources/FluidAudio/Shared/AssetDownloader.swift:60`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AssetDownloader.swift#L60).
12. **Return + report 100%** — `[String: MLModel]` is returned; progress reaches `1.0` through the byte-weighted (downloads 0→0.5, compilation 0.5→1.0) `DownloadPhase` ticker.

## Outputs
- `[String: MLModel]` keyed by the file basename (e.g. `"Preprocessor.mlmodelc"`).
- Side effect: cache directory populated with compiled `.mlmodelc` bundles plus auxiliary `.json` / `.bin` / `.model` files.
- Progress callbacks (`onProgress(Double)` 0.0–1.0) byte-weighted across listing, downloads (0→0.5), and compile (0.5→1.0).

## Error Modes
- `HuggingFaceDownloadError.rateLimited` — `429` / `503`; caught by the once-only outer retry that purges the cache first.
- `HuggingFaceDownloadError.htmlErrorResponse` — HF served a `200 OK` HTML overload page.
- `HuggingFaceDownloadError.modelNotFound` — a required file was missing from the listing entirely.
- `fileReadCorruptFile` — directory exists but `coremldata.bin` sentinel missing → cache wipe + retry.
- Timeout after 30 min (`DownloadConfig.timeout = 1800`) on slow links.
- Auth failure (401) if any of the three HF token env vars is stale.
- `ModelRegistry.Error.invalidURL` from a malformed `baseURL` override.

## Prompts and Models Used

## Usage Metrics

## Changelogs
