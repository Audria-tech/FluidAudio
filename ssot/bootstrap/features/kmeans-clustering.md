---
id: kmeans-clustering
name: K-Means Clustering
repo: FluidAudio
status: active
linked_features: [offline-diarizer-manager, vbx-clustering, speaker-count-constraints]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# K-Means Clustering

## TL;DR
`KMeansClustering` provides classical k-means with k-means++-style centroid initialization, used as the speaker-count-forcing fallback when VBx's cluster count doesn't meet the user's `numSpeakers`/min/max constraints. Returns both assignments and centroids.

## Motivation — why it exists

## Context

## What It Does
`cluster(embeddings:numClusters:maxIterations:seed:)` is the assignment-only convenience; `clusterWithCentroids` returns both assignments and the final centroids. Both L2-normalize the input embeddings, initialize centroids deterministically with a seeded `SeededRNG`, then loop assign→update until convergence (`newAssignments == assignments`) or `maxIterations=300` is reached. Assignment uses cosine similarity (dot product on unit vectors). `OSSignposter` instruments the "KMeans Clustering" interval.

The early-return paths: empty input → `([], [])`; empty dimension → all-zero assignments; `count <= k` → identity assignment with embeddings as centroids.

## Key Code
- [`KMeansClustering.swift:8`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Clustering/KMeansClustering.swift#L8) — `@available(macOS 14.0, iOS 17.0, *) struct KMeansClustering`.
- [`KMeansClustering.swift:39`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Clustering/KMeansClustering.swift#L39) — `clusterWithCentroids` — main loop entry.
- [`KMeansClustering.swift:69`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Clustering/KMeansClustering.swift#L69) — assign-update loop with `newAssignments == assignments` early-exit.

## Edge Cases & Failure Modes
- `seed: nil` uses `UInt64.random(in: 0...UInt64.max)` — non-reproducible by default. Tests pass an explicit seed for determinism.
- `k = min(numClusters, count)` ensures the algorithm never requests more clusters than data points.
- Zero-magnitude embeddings have ill-defined cosine similarity; the L2 normalization step uses an implicit epsilon (whatever `VDSPOperations.l2Normalize` provides — 1e-12).
- The single max-iterations bound is 300; for non-converging inputs (degenerate centroids) the assignments will be whatever was current at iteration 300 — no warning.

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/Clustering/KMeansClusteringTests.swift` — covers reproducibility, convergence, and the early-return paths.

## Changelogs
