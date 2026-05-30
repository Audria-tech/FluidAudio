---
id: app-logger
name: App Logger
repo: FluidAudio
status: active
linked_features: []
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# App Logger

## TL;DR
`AppLogger` is the module-wide logging wrapper. Delegates to `OSLog.Logger` for Unified Logging and optionally mirrors to stderr (always in `DEBUG`; warnings / errors / faults also in release). Subsystem defaults to `"com.fluidinference"`, category varies per component.

## Motivation — why it exists

## Context

## What It Does
Constructed with a `category` (and optionally a subsystem). Each level method (`debug`, `info`, `notice`, `warning`, `error`, `fault`) routes through a single private `log(_:_:)`. In `DEBUG` builds every level mirrors to console; in release, debug/info/notice go only to `os_log`, warnings/errors/faults additionally mirror to stderr. Warnings/errors/faults in release mirror **synchronously** (`logToConsoleSynchronously`) so a fault message lands before any subsequent `exit()`; everything else hops onto a detached `.utility`-priority Task that writes through the `LogConsole` actor.

## Key Code
- [`Sources/FluidAudio/Shared/AppLogger.swift:10`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AppLogger.swift#L10) — `nonisolated(unsafe) static var defaultSubsystem` — set-once at startup convention
- [`Sources/FluidAudio/Shared/AppLogger.swift:12`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AppLogger.swift#L12) — `Level` enum (`debug` … `fault`) with `rawValue` ordering
- [`Sources/FluidAudio/Shared/AppLogger.swift:64`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AppLogger.swift#L64) — `log(_:_:)` — `#if DEBUG` mirrors everything; release uses level switch
- [`Sources/FluidAudio/Shared/AppLogger.swift:96`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AppLogger.swift#L96) — release-mode warning/error/fault → synchronous stderr write
- [`Sources/FluidAudio/Shared/AppLogger.swift:131`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/AppLogger.swift#L131) — `actor LogConsole` — formats and writes async log lines to `FileHandle.standardError`

## Edge Cases & Failure Modes
- `defaultSubsystem` is `nonisolated(unsafe)` — concurrent mutation while loggers are being constructed is undefined; the doc comment says "set before creating any logger instances". [REVIEW: `AppLogger.defaultSubsystem` is `nonisolated(unsafe)` — set-once invariant not enforced]
- Writes to `FileHandle.standardError` can throw on closed pipes; both the sync and async paths catch and `print` a fallback line — recoverable but spams stdout.
- `DateFormatter` is created inside the synchronous path on every call (not cached) — minor allocation cost per warning/error in release. The async `LogConsole` actor caches its formatter.
- `Task.detached(priority: .utility)` for debug/info logs means messages may appear out of order relative to wall time when many calls happen near-simultaneously.
- No log-level filtering — every level routes to `os_log` always; consumers must use `log` profile / predicates to filter.

## Test Coverage
No dedicated logger tests; behaviour is exercised indirectly throughout the test suite (logs surface in test output).

## Changelogs
