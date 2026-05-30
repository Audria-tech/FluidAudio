---
id: model-registry
name: Model Registry
repo: FluidAudio
status: active
linked_features: []
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# Model Registry

## TL;DR
Configures the base HTTP endpoint and (macOS-only) HTTP/HTTPS proxy settings used by every FluidAudio downloader. Resolves HuggingFace-style URLs for models, datasets, and tree-listing API calls, and produces a shared `URLSession` that all download paths reuse.

## Motivation — why it exists

## Context

## What It Does
`ModelRegistry` is a no-instance namespace (`public enum`) that centralises three concerns: (1) what host serves model artifacts, (2) how to build the HuggingFace-style URL shapes for that host, and (3) how the underlying `URLSession` is wired with proxy settings. The base URL falls through a fixed priority chain — programmatic override via `baseURL` setter → `REGISTRY_URL` env → `MODEL_REGISTRY_URL` env → hard-coded `https://huggingface.co` — so deployments behind a private mirror can swap the host without touching call sites.

URL construction is offered as four typed builders (`apiModels`, `resolveModel`, `apiDatasets`, `resolveDataset`) plus a string-returning `resolveDatasetBase`. All four throw `ModelRegistry.Error.invalidURL` if the assembled string fails `URL(string:)` parsing — the caller cannot accidentally end up with a stringly-typed nil URL. The HuggingFace API shape (`/api/models/<repo>/tree/main/<sub>` for listings, `/<repo>/resolve/main/<file>` for content) is hard-coded here; any mirror must match it.

Proxy handling reads `https_proxy` / `http_proxy` from the environment (macOS only — the iOS path returns `nil`) and packs them into a `connectionProxyDictionary` keyed by the `kCFNetworkProxies…` constants. `configuredSession()` returns a fresh `URLSession` with that configuration applied, which `DownloadUtils.sharedSession` initialises once at process startup.

## Key Code
- [`Sources/FluidAudio/ModelRegistry.swift:27`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelRegistry.swift#L27) — `nonisolated(unsafe)` storage backing the programmatic override
- [`Sources/FluidAudio/ModelRegistry.swift:32`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelRegistry.swift#L32) — `baseURL` precedence chain (override → REGISTRY_URL → MODEL_REGISTRY_URL → huggingface.co)
- [`Sources/FluidAudio/ModelRegistry.swift:47`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelRegistry.swift#L47) — `apiModels(_:_:)` — `/api/models/<repo>/<apiPath>` listings
- [`Sources/FluidAudio/ModelRegistry.swift:56`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelRegistry.swift#L56) — `resolveModel(_:_:)` — `/<repo>/resolve/main/<file>` content URL
- [`Sources/FluidAudio/ModelRegistry.swift:65`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelRegistry.swift#L65) — `apiDatasets` / `resolveDataset` / `resolveDatasetBase` dataset variants
- [`Sources/FluidAudio/ModelRegistry.swift:90`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelRegistry.swift#L90) — `configuredSession()` builds the shared `URLSession`
- [`Sources/FluidAudio/ModelRegistry.swift:103`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelRegistry.swift#L103) — `configureProxySettings()` reads env, returns `[String: Any]` keyed by `kCFNetworkProxies*` constants
- [`Sources/FluidAudio/ModelRegistry.swift:128`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/ModelRegistry.swift#L128) — `parseProxyURL` extracts host/port, branches on `HTTP`/`HTTPS`

## Edge Cases & Failure Modes
- `Error.invalidURL` thrown if any URL string fails to parse — typically when `baseURL` is overridden with a malformed value (e.g. missing scheme).
- Trailing slash in `baseURL` is **not** normalised — overriding with `"https://mirror/"` produces `"https://mirror//api/models/..."` (double slash). Callers must pass an unsuffixed host.
- Proxy parsing is best-effort: if `https_proxy` is set but missing host or port, a warning is logged and the proxy is dropped silently.
- `_customBaseURL` is `nonisolated(unsafe)` — concurrent reads during a `baseURL` setter call are undefined. The doc comment relies on the "set once at startup" convention, not a runtime guard. [REVIEW: `nonisolated(unsafe) static` — set-once invariant not enforced at runtime]
- Proxy support is **macOS only**; iOS returns `nil` from `configureProxySettings`.
- The HuggingFace path shape (`/resolve/main/`, `/api/models/`, `/api/datasets/`) is baked into builders — a mirror without HuggingFace's URL layout cannot be used.

## Performance / Concurrency Notes
URL builders allocate a `String` and a `URL` per call but do no I/O. `configuredSession()` allocates a new `URLSession` per invocation — `DownloadUtils` caches it in a `static let`, so the registry is hit exactly once per process.

## Test Coverage
`Tests/FluidAudioTests/Shared/ModelRegistryTests.swift` covers the env-var precedence and URL-construction throw paths.

## Changelogs
