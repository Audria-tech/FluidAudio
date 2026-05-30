---
id: fastcluster-bridge
name: FastCluster Bridge
repo: FluidAudio
status: active
linked_features: [ahc-clustering, offline-diarizer-manager]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# FastCluster Bridge

## TL;DR
`FastClusterWrapper` is a C wrapper around the BSD-licensed `fastcluster` C++ library (Daniel Müllner / Google) that exposes centroid-linkage hierarchical clustering through a Swift-callable C API. Used by `AHCClustering` to reproduce pyannote/community-1's clustering step.

## Motivation — why it exists

## Context

## What It Does
The directory contains:
- `FastClusterWrapper.cpp` — C++ implementation wrapping the upstream fastcluster algorithms.
- `fastcluster_internal.hpp` — vendored fastcluster headers.
- `include/FastClusterWrapper.h` — C API header exposing `fastcluster_compute_centroid_linkage(...)` and the `fastcluster_wrapper_status` enum (`FASTCLUSTER_WRAPPER_SUCCESS` etc.).
- `include/module.modulemap` — Swift module bridge so the C header can be imported as a module from Swift code.

The wrapper exposes a single function: `fastcluster_compute_centroid_linkage(data, pointCount, dimension, dendrogramOut, dendrogramLength)`. Inputs are row-major double-precision feature vectors. Output is a SciPy-format dendrogram with `(pointCount - 1) * 4` doubles: `(leftNodeIdx, rightNodeIdx, distance, sampleCount)` per merge step. Algorithm: centroid linkage, which merges the pair of clusters whose centroids are closest at each step.

`AHCClustering.swift` imports this module via a conditional `#if canImport(FastClusterWrapper)` / `#elseif canImport(FluidAudio_FastClusterWrapper)` block to accommodate SwiftPM's target-naming convention.

## Key Code
- [`FastClusterWrapper/include/FastClusterWrapper.h`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FastClusterWrapper/include/FastClusterWrapper.h) — C API header.
- [`FastClusterWrapper/FastClusterWrapper.cpp`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FastClusterWrapper/FastClusterWrapper.cpp) — wrapper implementation.
- [`FastClusterWrapper/include/module.modulemap`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FastClusterWrapper/include/module.modulemap) — Swift module map.
- [`FastClusterWrapper/README.md`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FastClusterWrapper/README.md) — purpose and licensing notes.

## Edge Cases & Failure Modes
- `fastcluster_compute_centroid_linkage` returns a non-success status on bad input shapes; `AHCClustering` catches this and falls back to singleton clusters (each embedding its own cluster).
- The dendrogram output buffer must be exactly `(pointCount - 1) * 4` doubles; the wrapper takes its length as a parameter for bounds checking.
- The C++ side allocates internally; pointCount = 0 or 1 is degenerate (handled by `AHCClustering` before the FFI call).
- License is BSD-2-Clause; `ThirdPartyLicenses/fastcluster-LICENSE.md` carries the upstream copyright.
- The wrapper does not support distance metrics other than centroid linkage — pyannote uses centroid linkage specifically.

## Test Coverage
- Exercised through `Tests/FluidAudioTests/Diarizer/Offline/AHCClusteringTests.swift`.

## Changelogs
