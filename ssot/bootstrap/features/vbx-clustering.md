---
id: vbx-clustering
name: VBx Clustering
repo: FluidAudio
status: active
linked_features: [offline-diarizer-manager, ahc-clustering, kmeans-clustering, speaker-count-constraints]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# VBx Clustering

## TL;DR
`VBxClustering` ports the BUT/FIT Variational Bayes algorithm for speaker diarization: it refines AHC's hard cluster IDs into soft `gamma` posteriors + mixture weights `pi` via EM iterations with ELBO tracking. `refineWithConstraints` wraps this with K-Means re-clustering when the detected speaker count falls outside user constraints.

## Motivation — why it exists

## Context

## What It Does
`refine(rhoFeatures:initialClusters:)` flattens the per-frame 128-dim `rho` features into a single `[Double]` buffer (memcpy per row), builds an initial `gamma` matrix from the hard AHC assignments (one-hot), and runs `runVBx(...)` with `Fa`/`Fb` warm-start parameters (default 0.07/0.8) and an `initSmoothing=7.0` temperature. The EM iterations use BLAS for the heavy linear algebra, tracking ELBO and stopping at `config.vbx.convergenceTolerance` (default 1e-4) or `config.vbx.maxIterations` (default 20). On BLAS failure, falls back to the initial gamma + uniform pi.

`refineWithConstraints(rhoFeatures:trainingEmbeddings:initialClusters:constraints:)` runs `refine` first, then checks `constraints.needsAdjustment(detectedCount:)`. If the count is outside `[minSpeakers, maxSpeakers]`, it calls `KMeansClustering.clusterWithCentroids` on the original embeddings (not rho) with `targetCount` and returns a `VBxOutput` with `wasAdjusted=true` + `originalClusterCount` set + the K-Means centroids. The K-Means `maxIterations` is hard-coded to 100 here (vs the default 300).

Outputs are returned in a `VBxOutput` (gamma, pi, hardClusters, centroids, numClusters, elbos, wasAdjusted, originalClusterCount).

## Key Code
- [`VBxClustering.swift:41`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Clustering/VBxClustering.swift#L41) — `refine(rhoFeatures:initialClusters:)` — VBx entry.
- [`VBxClustering.swift:119`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Clustering/VBxClustering.swift#L119) — `runVBx` call site + BLAS-failure fallback to uniform pi + initial gamma.
- [`VBxClustering.swift:685`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Diarizer/Offline/Clustering/VBxClustering.swift#L685) — `refineWithConstraints` — K-Means re-clustering fallback.

## Edge Cases & Failure Modes
- Empty `rhoFeatures` → returns an empty `VBxOutput` with `numClusters: 0`.
- Empty first feature row (`dimension == 0`) → logs error and returns the same empty output.
- PLDA `psi` dimension mismatch → falls back to an identity `phi = [1.0, ..., 1.0]`.
- Empty `initialClusters` → initial gamma is uniform (every frame equally assigned to every speaker).
- Cluster indices in `initialClusters` are clamped via `max(0, min(cluster, speakerCount - 1))` to keep gamma well-formed.
- BLAS failure (any error in `runVBx`) falls back to the initial gamma + uniform pi; `elbos = []`. The downstream pipeline still gets a valid output but with degraded quality.
- `refineWithConstraints` returns the K-Means result with the *VBx-derived* gamma + pi (these may not correspond to the new K-Means clusters). [REVIEW: gamma/pi are likely stale w.r.t. the K-Means clusters; downstream centroid math in `OfflineDiarizerManager.computeCentroids` checks `wasAdjusted` and switches to using the K-Means centroids directly, sidestepping this.]
- The K-Means re-cluster uses 100 iterations (hard-coded) — half the default 300 from `KMeansClustering`.

## Test Coverage
- `Tests/FluidAudioTests/Diarizer/Offline/VBxConstraintTests.swift` — `refineWithConstraints` adjustment behavior.

## Changelogs
