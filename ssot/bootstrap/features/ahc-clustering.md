---
id: ahc-clustering
name: AHC Clustering
repo: FluidAudio
status: active
linked_features: [offline-diarizer-manager, vbx-clustering, fastcluster-bridge]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# AHC Clustering

## TL;DR
`AHCClustering` performs agglomerative hierarchical clustering with centroid linkage on L2-normalized 256-dim embeddings, using the C++ `fastcluster_compute_centroid_linkage` FFI bridge. Returns per-embedding cluster IDs by cutting the dendrogram at a user-supplied distance threshold.

## Motivation — why it exists

## Context

## What It Does
`cluster(embeddingFeatures:threshold:)` first L2-normalizes the row-major flattened feature matrix via per-row `vDSP_dotprD` + `vDSP_vsmulD`. Then it crosses the FFI boundary: `fastcluster_compute_centroid_linkage(normalizedPointer, count, dimension, dendrogramPointer, dendrogramLength)` populates an `(N-1) × 4` SciPy-format dendrogram (left node, right node, distance, sample count). The `convertThresholdToDistance` helper translates the user's cosine-similarity threshold to the equivalent Euclidean distance on unit-norm vectors. `assignmentsFromDendrogram` cuts the dendrogram at `distanceThreshold` and produces per-leaf cluster IDs, then `remapClusterIds` renumbers them densely from 0.

Uses `OSSignposter` for performance instrumentation ("Agglomerative Hierarchical Clustering" interval). Conditional `#if canImport(FastClusterWrapper)` import accommodates SwiftPM target naming via `FluidAudio_FastClusterWrapper`.

## Key Code
- [`AHCClustering.swift:20`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Clustering/AHCClustering.swift#L20) — `cluster(embeddingFeatures:threshold:)` — the entry point.
- [`AHCClustering.swift:40`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Clustering/AHCClustering.swift#L40) — FastCluster FFI call site.
- [`AHCClustering.swift:70`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Clustering/AHCClustering.swift#L70) — `normalizeFeatures` row-by-row L2 normalization.

## Edge Cases & Failure Modes
- Returns `[]` for zero embeddings, `[0]` for a single embedding (singleton cluster).
- Returns `Array(0..<count)` (every embedding in its own cluster) if `fastcluster_compute_centroid_linkage` returns non-success — defensive fallback that logs the status code.
- Empty feature dimension (`dimension == 0`) returns `Array(repeating: 0, count: count)` (single cluster).
- Zero-magnitude rows in the normalization step have `scale = 0`, producing all-zero normalized vectors — these become indistinguishable to fastcluster and will cluster arbitrarily.
- The dendrogram buffer is `(count - 1) * 4` `Double`s; for very large `count` this could be memory-intensive (e.g., 10k embeddings → 320 KB).

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/Offline/AHCClusteringTests.swift` — covers FFI roundtrip, threshold cuts, and the fallback path.

## Changelogs
