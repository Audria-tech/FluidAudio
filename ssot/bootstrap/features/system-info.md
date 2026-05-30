---
id: system-info
name: System Info
repo: FluidAudio
status: active
linked_features: [app-logger, performance-metrics]
linked_flows: []
last_human_review: null
last_auto_update: 2026-05-29
source_commit_sha:
  FluidAudio: 94e54782a613d6ce2ba627e17e930f3107652530
---

# System Info

## TL;DR
Host/architecture introspection helpers — OS name + version, CPU brand, core counts, physical memory, Rosetta status — plus current and peak resident memory queries via Mach `task_info`. Includes a one-shot logger (`logOnce`) so every cold-start logs the environment exactly once per process. C shim `MachTaskSelfWrapper` is required because Swift 6 cannot import the `mach_task_self_` macro directly.

## Motivation — why it exists

## Context

## What It Does
`isAppleSilicon` / `isIntelMac` are compile-time arch slices (`arch(arm64)` / `arch(x86_64)`). `summary()` builds a human-readable line — `"<OS> <version>, arch=<arm64|x86_64>, chip=<machdep.cpu.brand_string>, cores=<active>/<total>, mem=<formatted>, rosetta=<bool>"` — by querying `ProcessInfo`, `sysctlbyname`, and `sysctl.proc_translated`. `logOnce(using:)` delegates to the `SystemInfoReporter` actor that guards a `didLog` bool; on first call it emits the summary at `info` level via the supplied `AppLogger`.

`currentResidentMemoryBytes()` and `peakResidentMemoryBytes()` call Mach's `task_info(TASK_VM_INFO)` against `get_current_task_port()` — that C function lives in `Sources/MachTaskSelfWrapper/MachTaskSelf.c` and exists solely because the `mach_task_self_` C macro isn't representable in Swift's modulemap import. The Mach call's `count` parameter is computed from `task_vm_info_data_t` / `natural_t` sizes; result is `info.resident_size` / `info.resident_size_peak`.

## Key Code
- [`Sources/FluidAudio/Shared/SystemInfo.swift:15`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/SystemInfo.swift#L15) — `isAppleSilicon` / `isIntelMac` — compile-time arch checks
- [`Sources/FluidAudio/Shared/SystemInfo.swift:31`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/SystemInfo.swift#L31) — `summary()` — OS + arch + chip + cores + memory + Rosetta one-liner
- [`Sources/FluidAudio/Shared/SystemInfo.swift:85`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/SystemInfo.swift#L85) — `logOnce(using:)` — `SystemInfoReporter` actor guards once-per-process emission
- [`Sources/FluidAudio/Shared/SystemInfo.swift:91`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/SystemInfo.swift#L91) — `currentResidentMemoryBytes` / `peakResidentMemoryBytes` — Mach `task_info(TASK_VM_INFO)`
- [`Sources/FluidAudio/Shared/SystemInfo.swift:135`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/SystemInfo.swift#L135) — `sysctlString` — two-pass sysctl read
- [`Sources/FluidAudio/Shared/SystemInfo.swift:150`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/SystemInfo.swift#L150) — `isRunningTranslated` — `sysctl.proc_translated` for Rosetta detection (macOS 11+)
- [`Sources/FluidAudio/Shared/SystemInfo.swift:166`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/FluidAudio/Shared/SystemInfo.swift#L166) — `actor SystemInfoReporter` — `didLog` flag
- [`Sources/MachTaskSelfWrapper/MachTaskSelf.c:5`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/MachTaskSelfWrapper/MachTaskSelf.c#L5) — `get_current_task_port()` — returns `mach_task_self_` (macro → function)
- [`Sources/MachTaskSelfWrapper/include/MachTaskSelf.h:8`](https://github.com/Audria-tech/FluidAudio/blob/94e54782a613d6ce2ba627e17e930f3107652530/Sources/MachTaskSelfWrapper/include/MachTaskSelf.h#L8) — header

## Edge Cases & Failure Modes
- `currentResidentMemoryBytes` / `peakResidentMemoryBytes` return `nil` on `task_info` failure (rare). Callers must tolerate nil.
- `sysctlString` returns `nil` if the key doesn't exist or sysctl reports zero size — `summary` falls back to `"UnknownChip"`.
- `isRunningTranslated` returns `false` on non-Darwin platforms or under macOS < 11.
- Mach memory queries are macOS/iOS only; the file gates them with `#if canImport(Darwin)`. Cross-platform consumers must check before calling.
- `SystemInfoReporter` is an actor — `logOnce` is `async` and the suspension may delay the very first log entry by a tick.
- `summary()` constructs a `ByteCountFormatter` and a sysctl probe per call — fine for once-per-process but noticeable if called in a tight loop.

## Performance / Concurrency Notes
- `logOnce` guards against duplicate emission across concurrent callers via actor isolation; cheap on repeat calls (just an `await` on a bool check).
- Memory queries are unsynchronised — multiple concurrent calls each issue an independent `task_info` syscall; safe but redundant.

## Test Coverage
No dedicated tests; `summary()` output is visible in CI test logs.

## Changelogs
