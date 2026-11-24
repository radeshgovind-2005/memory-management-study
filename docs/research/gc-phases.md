# Go GC phases: what each stage costs, how to observe it, and what each ladder rung changes

Research note for the cache experiment (`docs/designs/cache-experiment.md`). It
describes the Go garbage collection cycle stage by stage, lists the signals
that expose each stage, maps each ladder rung to the stages it changes, sets out
the comparable observables on the Rust side, and ends with a small proposal for
the design. Its proposal was adopted in the design as decision D13 and Goal 9.

## How this was checked

Statements about runtime internals were checked against the Go source. The
Go 1.26.5 tree installed on the author's machine was read directly, and the
Go 1.27.0 tag was spot-checked on GitHub for the parts most likely to change
(experiment defaults, the gctrace format, the full `runtime/metrics` name list,
the forced-GC period and the allocator entry points). Go 1.27.0 was released on
2026-08-19 and is the current major release ([release history](https://go.dev/doc/devel/release)).
The design does not yet name its pinned Go version; this note assumes 1.26 or
1.27. Source links use the `go1.27.0` tag; line positions may differ slightly
from 1.26.

Anything not confirmed from go.dev, the Go source, the Go blog or a Go design
document is marked **(unverified)**.

## Summary

* In Go 1.26 and 1.27 a GC cycle has five observable stages: a short
  stop-the-world (STW) sweep termination, a concurrent mark phase (background
  workers, idle workers and mutator assists, with the write barrier on), a
  short STW mark termination, concurrent sweep, and the background scavenger,
  which returns memory to the OS. The pacer decides when the next cycle starts.
* The Green Tea collector is the default since Go 1.26. It changes how marking
  scans small objects (by span instead of object by object), but not the phase
  structure, the pacer, the STW points or the assist mechanism.
* At `GOMAXPROCS=2` the runtime starts **no dedicated mark worker**. The 25%
  background target (0.5 CPU) is met by fractional workers instead, plus idle
  workers when a P has nothing else to run. Assists therefore carry relatively
  more of the work than on larger machines.
* `GODEBUG=gctrace=1`, which the design already collects, gives per-cycle wall
  time for the three main stages and a per-cycle CPU split into assist,
  background and idle. Most of the per-phase analysis can be done on data the
  design already produces.
* Two details affect the existing plan. First, the `/cpu/classes/...` metrics
  are snapshots that update only at a GC stop-the-world, so they step once per
  cycle and cannot be read per second. Second, the forced periodic GC is not
  labelled as forced in gctrace or in `/gc/cycles/forced`, so it has to be
  inferred from timing.
* Go 1.27 makes the compiler call size-specialised allocation functions
  directly (`runtime.mallocgcSmallScanNoHeaderSC…` and similar). A perf frame
  rule that matches only the exact name `runtime.mallocgc` would undercount
  allocation. The frame list should match the `runtime.mallocgc` prefix. The
  design's `mspan.sweep` frame is now `runtime.(*sweepLocked).sweep`.

---

## 1. The GC cycle in current Go

### 1.1 Stages

The authoritative description is the header comment of
[`runtime/mgc.go`](https://github.com/golang/go/blob/go1.27.0/src/runtime/mgc.go).
The Go GC is a concurrent, non-generational, non-compacting mark-sweep
collector with a write barrier. A cycle proceeds as follows.

| # | Stage | STW? | What happens | Main cost driver |
| --- | --- | --- | --- | --- |
| 1 | **Sweep termination** | Yes | Stop the world so every P reaches a safe point; finish any spans left unswept (only if this cycle started early). | Time to stop running goroutines; usually well under 1 ms. |
| 2a | **Mark setup** | Yes (same STW) | Set `gcphase = _GCmark`, enable the write barrier, enable assists, queue root-marking jobs. Nothing is scanned until every P has the write barrier on. | Same STW. |
| 2b | **Concurrent mark** | No | Start the world. Mark workers scan roots (goroutine stacks, globals, runtime structures) and then drain the grey work queue. Goroutines that allocate perform *mark assists*. New objects are allocated already marked ("black"). | Scan work, which is roughly proportional to live pointer-containing heap plus stacks and globals. |
| 3 | **Mark termination** | Yes | A distributed termination check (`gcMarkDone`) finds no work left; stop the world, disable workers and assists, flush per-P caches. | Time to stop goroutines plus housekeeping; usually sub-millisecond. |
| 4 | **Concurrent sweep** | No | Set `gcphase = _GCoff`, turn the write barrier off, start the world. Spans are swept lazily by allocating goroutines (before a span is reused) and by the background `bgsweep` goroutine. | Number of spans; tends to be small next to mark. |
| 5 | **Scavenger** | No | The background scavenger returns free pages to the OS (`madvise`), paced towards a memory target. Allocation can also scavenge eagerly ("assist") when it would otherwise exceed the memory limit. | Free memory to release. |

Sources: [`mgc.go`](https://github.com/golang/go/blob/go1.27.0/src/runtime/mgc.go),
[`mgcsweep.go`](https://github.com/golang/go/blob/go1.27.0/src/runtime/mgcsweep.go),
[`mgcscavenge.go`](https://github.com/golang/go/blob/go1.27.0/src/runtime/mgcscavenge.go).
The [GC guide](https://go.dev/doc/gc-guide) lists the latency sources as:
brief STW pauses at the phase transitions; scheduling delay because the GC
takes 25% of CPU during mark; user goroutines assisting the GC under a high
allocation rate; extra work on pointer writes during mark (the write barrier);
and goroutines briefly suspended while their stacks are scanned. The guide adds
that pauses "are more strongly proportional to GOMAXPROCS algorithmically, but
most commonly are dominated by the time it takes to stop running goroutines".

### 1.2 Mark workers at 2 vCPUs

The pacer aims for background marking to use 25% of `GOMAXPROCS`
(`gcBackgroundUtilization = 0.25` in
[`mgcpacer.go`](https://github.com/golang/go/blob/go1.27.0/src/runtime/mgcpacer.go)).
In `startCycle` the runtime rounds `GOMAXPROCS × 0.25` to a whole number of
*dedicated* workers. If the rounding error is above 30%, it reduces that number
and makes up the remainder with a *fractional* worker. At `GOMAXPROCS = 2` the
target is 0.5 CPU: rounding gives 1 dedicated worker (100% error), which is
reduced to **0 dedicated workers and a fractional utilisation goal of 0.25 per
P**. The source comment calls out this case ("GOMAXPROCS<=3 or GOMAXPROCS=6").

*Idle* workers run whenever a P would otherwise be idle. In a normal cycle up
to `GOMAXPROCS − dedicated` of them may run, which is 2 here. During a
**periodic (time-triggered) cycle** the runtime limits idle workers to 1 when
there are no dedicated workers (same function). Idle-worker CPU uses spare
capacity: the `runtime/metrics` documentation says it "should be subtracted
from the total GC CPU time to obtain a measure of compulsory GC CPU time"
([runtime/metrics](https://pkg.go.dev/runtime/metrics)). At the 25% load level
there is a lot of idle CPU, so total GC CPU will look larger than the cost the
GC actually imposes on requests unless idle time is shown separately.

### 1.3 Mark assists and assist credit

While marking runs, every allocation adds to the allocating goroutine's *assist
debt*, at a rate the pacer sets (`assistWorkPerByte`: the scan work still
expected divided by the heap growth still allowed before the goal). When a
goroutine is in debt, `gcAssistAlloc`
([`mgcmark.go`](https://github.com/golang/go/blob/go1.27.0/src/runtime/mgcmark.go))
first tries to *steal background credit* (`bgScanCredit`, built up when
background workers do more work than assists need). If there is not enough
credit, the goroutine scans at least `gcOverAssistWork` (64 KiB of scan work)
itself. If no scan work is available, it parks on the assist queue
(`gcParkAssist`) until background workers flush credit (`gcFlushBgCredit`).
Assist time is therefore spent on the request goroutine: it adds directly to
the latency of the request that allocated. With no dedicated worker at
2 vCPUs, background credit builds up more slowly than on a larger machine, so
assists are expected to do a larger share of the marking (inferred from the
mechanism; to be measured, not assumed).

### 1.4 Write barrier

During mark, every heap pointer write goes through the write barrier, which
shades both the old and the new pointer and buffers them per P
(`runtime.gcWriteBarrier1`…`8` in assembly, flushed by `runtime.wbBufFlush` /
`wbBufFlush1`). Writes of integers, including `uint32` slab indexes, have no
barrier. Outside mark the barrier is a predictable branch. The GC guide notes
that this cost is the one latency source that execution traces do not show
([GC guide](https://go.dev/doc/gc-guide)); only CPU sampling sees it.

### 1.5 The pacer and the GOGC heap goal

The pacer was redesigned in Go 1.18
([design doc 44167](https://github.com/golang/proposal/blob/master/design/44167-gc-pacer-redesign.md)).
Since then the heap goal counts GC roots as well as live heap:

> Target heap memory = Live heap + (Live heap + GC roots) × GOGC / 100
> ([GC guide](https://go.dev/doc/gc-guide))

The minimum total heap is 4 MiB. "Doubling GOGC will double heap memory
overheads and roughly halve GC CPU cost" (same source). A cycle is *triggered*
before the goal is reached, so that marking can finish in time. The trigger
sits between about 0.7 and about 0.95 of the way from the last marked heap to
the goal (`minTriggerRatioNum = 45` and `maxTriggerRatioNum = 61` over 64 in
`mgcpacer.go`), depending on how long marking took in earlier cycles.

For this cache, the live heap is dominated by the cache itself (GBs at 5M and
10M entries), and that size is almost the same with either layout. The
difference between layouts is how much of that heap has to be **scanned**,
not how large it is. So at the same GOGC both layouts have a similar heap goal
and a similar time between cycles for a given allocation rate, but very
different mark work per cycle.

### 1.6 GOMEMLIMIT and the GC CPU limiter

`GOMEMLIMIT` (Go 1.19, [design 48409](https://github.com/golang/proposal/blob/master/design/48409-soft-memory-limit.md))
sets a soft limit on the total memory the Go runtime uses, computed as
`Sys − HeapReleased` ([GC guide](https://go.dev/doc/gc-guide)). The pacer uses
the lower of the GOGC goal and a goal derived from the memory limit
(`heapGoalInternal` / `memoryLimitHeapGoal` in `mgcpacer.go`). When the limit
is the binding one, cycles become more frequent as the heap approaches it.

To prevent a death spiral, a CPU limiter caps GC CPU at "roughly 50%, with a
`2 * GOMAXPROCS` CPU-second window" ([GC guide](https://go.dev/doc/gc-guide)).
It is implemented as a leaky bucket that fills with GC CPU and drains with
mutator CPU, with a capacity of 1 CPU-second per P
([`mgclimit.go`](https://github.com/golang/go/blob/go1.27.0/src/runtime/mgclimit.go)).
When the limiter is engaged, the heap may exceed the limit; the program slows
down "at most by 2x". `/gc/limiter/last-enabled:gc-cycle` records the last
cycle in which the limiter engaged.

The guide recommends leaving 5–10% headroom for memory the runtime does not
know about. It advises against `GOGC=off` with a limit when the memory is shared
with other programs, and against using a limit to avoid out-of-memory errors
when a program is already near its environment's limit. The `go-tuned` sweep
should be read with these points in mind: a setting that only performs well
because the limiter is engaged is trading memory safety for latency.

### 1.7 The forced periodic GC

`forcegcperiod = 2 * 60 * 1e9` (2 minutes) in
[`proc.go`](https://github.com/golang/go/blob/go1.27.0/src/runtime/proc.go)
is unchanged in 1.26 and 1.27. `sysmon` triggers a cycle of kind
`gcTriggerTime` when more than 2 minutes have passed since the last GC ended,
unless GOGC is negative (`GOGC=off`) (`gcTrigger.test` in `mgc.go`). Three
details matter for the experiment:

* It only fires when no other cycle has run for 2 minutes. A rung with frequent
  heap-triggered cycles never sees it; a rung with rare cycles (high GOGC or a
  low allocation rate) may be dominated by it.
* It is **not** marked `(forced)` in gctrace. That marker means a
  `runtime.GC()` call (`work.userForced = trigger.kind == gcTriggerCycle`). It
  is also counted in `/gc/cycles/automatic`, not in `/gc/cycles/forced`
  ("forced by the application"). It has to be inferred, for example as a cycle
  whose start is about 120 s after the previous cycle and whose starting heap is
  below the lowest possible trigger (the previous marked heap plus 45/64 of the
  way to the goal, Section 1.5), which gctrace=1 alone can compute.
* During a periodic cycle the idle mark workers are limited (Section 1.2), so
  its wall-clock mark time can be longer than a heap-triggered cycle with the
  same scan work.

### 1.8 What changed recently, and Green Tea status

| Version | Change relevant here | Source |
| --- | --- | --- |
| 1.18 | Pacer redesign; GC roots (stacks, globals) included in the GOGC goal | [design 44167](https://github.com/golang/proposal/blob/master/design/44167-gc-pacer-redesign.md), [GC guide](https://go.dev/doc/gc-guide) |
| 1.19 | `GOMEMLIMIT` and the GC CPU limiter | [design 48409](https://github.com/golang/proposal/blob/master/design/48409-soft-memory-limit.md), [Go 1.19 notes](https://go.dev/doc/go1.19) |
| 1.20 | `/cpu/classes/...` metrics **(version unverified; present in 1.26/1.27 source)** | [runtime/metrics](https://pkg.go.dev/runtime/metrics) |
| 1.21 | Execution tracing overhead cut from about 10–20% to about 1–2% CPU "for many applications" (the blog gives no exact version for the new figure; 1.21 is from its context) | [Go blog, 2024](https://go.dev/blog/execution-traces-2024) |
| 1.22 | `/sched/pauses/...` histograms; `/gc/pauses:seconds` deprecated **(version unverified; deprecation text confirmed in source)** | [runtime/metrics](https://pkg.go.dev/runtime/metrics) |
| 1.25 | Container-aware `GOMAXPROCS` (cgroup CPU limit, periodic updates; off if `GOMAXPROCS` is set explicitly); Green Tea as `GOEXPERIMENT=greenteagc`; `runtime/trace.FlightRecorder` | [Go 1.25 notes](https://go.dev/doc/go1.25) |
| 1.26 | **Green Tea on by default**; opt-out `GOEXPERIMENT=nogreenteagc`; expected 10–40% lower GC overhead, about 10% more on AVX-512 CPUs (Ice Lake / Zen 4 and newer) | [Go 1.26 notes](https://go.dev/doc/go1.26), [Green Tea blog](https://go.dev/blog/greenteagc) |
| 1.27 | Compiler calls size-specialised malloc routines (opt-out `GOEXPERIMENT=nosizespecializedmalloc`, expected to be removed in 1.28); no GC, pacer or `runtime/metrics` changes (the metric name list is identical to 1.26) | [Go 1.27 notes](https://go.dev/doc/go1.27); source diff |

**Green Tea.** In the `go1.27.0` source, `GreenTeaGC: true` is still in the
experiment baseline
([`buildcfg/exp.go`](https://github.com/golang/go/blob/go1.27.0/src/internal/buildcfg/exp.go)),
and the flag still exists. The 1.26 notes said the opt-out was "expected to be
removed in Go 1.27"; the 1.27 notes do not mention it. Whether
`nogreenteagc` still builds a working non-Green-Tea runtime in 1.27 is
**unverified**. Green Tea queues *spans* rather than objects and marks objects
in a span with inline mark bits. In the source this applies to objects whose
heap bits live in the span (`heapBitsInSpan`, size ≤ 512 bytes on 64-bit) and
that are at least 16 bytes (`gcUsesSpanInlineMarkBits` in
[`mgcmark_greenteagc.go`](https://github.com/golang/go/blob/go1.27.0/src/runtime/mgcmark_greenteagc.go)).
The blog says it helps most on regular heaps with many same-size objects, and
can regress when only one object per page is scanned at a time
([Green Tea blog](https://go.dev/blog/greenteagc)).

Consequences for this cache. In the pointer layout, the per-entry `node` (small,
pointer-containing) and the map groups holding `*node` are scanned through
Green Tea's span path. Values are `[]byte` backing arrays, which are pointer-free
("noscan"): they are marked but not scanned. In the slab layout, the large
record and slab arrays are pointer-free large objects, and a
`map[[16]byte]uint32` stores no pointers in its slots, so marking hardly visits
the cache at all. Green Tea therefore lowers the cost of the rung-1 and rung-2
layout. It does not change the *direction* of any prediction, but it can
change the size of the `go-pointerfree − go-default` gain. `PREDICTIONS.md`
should state which collector it assumes, as the design's Open Questions
already require.

---

## 2. Observable signals per stage

### 2.1 gctrace (already collected)

Format in 1.26 and 1.27 ([`runtime/extern.go`](https://github.com/golang/go/blob/go1.27.0/src/runtime/extern.go); print code in `gcMarkTermination`, `mgc.go`):

```
gc # @#s #%: #+#+# ms clock, #+#/#/#+# ms cpu, #->#-># MB, # MB goal, # MB stacks, #MB globals, # P
```

| Field | Meaning (from the print code) | Stage |
| --- | --- | --- |
| `gc #` | Cycle number | — |
| `@#s` | Seconds from runtime init to the **start of sweep termination** (`tSweepTerm − runtimeInitTime`), millisecond resolution | Cycle start |
| `#%` | Cumulative GC CPU fraction since program start (`gc_cpu_fraction`); not per cycle | — |
| clock `A+B+C ms` | A = sweep-termination STW wall time; B = concurrent mark wall time; C = mark-termination STW wall time | 1, 2b, 3 |
| cpu `A+B/C/D+E ms` | A = sweep-termination CPU (STW wall × Ps); B = **assist** CPU; C = **background** CPU (dedicated + fractional); D = **idle** worker CPU; E = mark-termination CPU (STW wall × Ps) | 1, 2b, 3 |
| `#->#-># MB` | Heap at GC start, heap at GC end, **live (marked) heap** | Mark result |
| `# MB goal` | Heap goal for this cycle | Pacer |
| `# MB stacks`, `#MB globals` | Scannable stack and global bytes (root size) | Roots |
| `# P` | GOMAXPROCS | — |
| `(forced)` | `runtime.GC()` call only; **not** the periodic GC | — |

Not in gctrace: sweep duration, scavenger activity and write-barrier cost. The
MB fields are truncated to whole MiB.

Related knobs:

* `gctrace=2` adds per-size-class scan counts after each line. With Green Tea:
  objects scanned individually ("sparse"), objects scanned through sparse spans,
  objects scanned through dense spans, and span counts (`dumpScanStats`). The
  counters are incremented in the scan loop only when `gctrace > 1`, so the
  overhead is small but non-zero. **Diagnostic runs only.** This is the most
  direct "objects scanned per cycle" signal Go offers.
* `gcpacertrace=1` prints pacer internal state; the format is not documented.
  **Diagnostic only.**
* `scavtrace=1` prints about one line per GC cycle: KiB returned to the OS in
  the background and eagerly, KiB currently released, and heap utilisation
  ([`extern.go`](https://github.com/golang/go/blob/go1.27.0/src/runtime/extern.go)).
  The cost is the same as gctrace (one line per cycle). It is safe in
  measurement runs.

gctrace costs one formatted stderr line per cycle. With the cycle counts
expected here (seconds to minutes apart) this is negligible and safe in
measurement runs. The format is documented as "subject to change", so the
parser should check the Go version in the manifest and fail loudly on an
unexpected format.

### 2.2 runtime/metrics (safe in measurement runs)

`metrics.Read` does not stop the world (there is no `stopTheWorld` in
[`runtime/metrics.go`](https://github.com/golang/go/blob/go1.27.0/src/runtime/metrics.go)).
Current `prometheus/client_golang` builds its `go_memstats_*` values from
`runtime/metrics` rather than calling `ReadMemStats`
([`go_collector_latest.go`](https://github.com/prometheus/client_golang/blob/main/prometheus/go_collector_latest.go)).
Exact names, identical in 1.26 and 1.27:

| Stage | Metric | Notes |
| --- | --- | --- |
| STW 1 and 3 | `/sched/pauses/total/gc:seconds` (histogram) | Per-pause, from the decision to stop to restart. Replaces the deprecated `/gc/pauses:seconds`. |
| STW 1 and 3 | `/sched/pauses/stopping/gc:seconds` (histogram) | The "stopping" part only: time to bring all Ps to a halt. |
| Non-GC STW | `/sched/pauses/total/other:seconds`, `/sched/pauses/stopping/other:seconds` | Catches `ReadMemStats` and `GOMAXPROCS` changes, which should be zero in a clean run. |
| Mark | `/cpu/classes/gc/mark/assist:cpu-seconds` | Assist CPU (mutator). |
| Mark | `/cpu/classes/gc/mark/dedicated:cpu-seconds` | Background mark CPU. At 2 Ps this is fractional-worker time, since there is no dedicated worker (gctrace also adds fractional into its "background" field). **(The docs do not say explicitly that fractional time is counted here; inferred.)** |
| Mark | `/cpu/classes/gc/mark/idle:cpu-seconds` | Idle-worker CPU; subtract it to get compulsory GC CPU. |
| STW | `/cpu/classes/gc/pause:cpu-seconds` | Pause × GOMAXPROCS. |
| All GC | `/cpu/classes/gc/total:cpu-seconds` | Sum of the above. |
| Scavenger | `/cpu/classes/scavenge/{background,assist,total}:cpu-seconds` | |
| Context | `/cpu/classes/user:cpu-seconds`, `/cpu/classes/idle:cpu-seconds`, `/cpu/classes/total:cpu-seconds` | |
| Pacer | `/gc/heap/goal:bytes`, `/gc/heap/live:bytes` (marked by the previous GC), `/gc/gogc:percent`, `/gc/gomemlimit:bytes` | |
| Scan size | `/gc/scan/heap:bytes`, `/gc/scan/stack:bytes`, `/gc/scan/globals:bytes`, `/gc/scan/total:bytes` | `/gc/scan/heap` is "the total amount of heap space that is scannable": the best per-run proxy for pointer-heavy heap. |
| Cycles | `/gc/cycles/automatic:gc-cycles`, `/gc/cycles/forced:gc-cycles`, `/gc/cycles/total:gc-cycles` | The periodic GC is counted in *automatic*. |
| Limiter | `/gc/limiter/last-enabled:gc-cycle` | Non-zero means the 50% limiter engaged at some point. |
| Allocation | `/gc/heap/allocs:bytes`, `/gc/heap/allocs:objects`, `/gc/heap/frees:*` | Allocation rate drives cycle frequency and assists. |
| Memory | `/memory/classes/heap/{objects,free,released,unused}:bytes`, `/memory/classes/total:bytes` | Sawtooth and scavenger effect. |
| Scheduling | `/sched/latencies:seconds` (histogram) | Time runnable before running. Shows CPU taken by mark workers. |

**Important semantic detail.** All `/cpu/classes/...` values come from a
snapshot taken at GC STW points. `cpuStatsAggregate.compute` copies
`work.cpuStats` as of the last STW, and a TODO notes that it is not refreshed
between cycles ([`runtime/metrics.go`](https://github.com/golang/go/blob/go1.27.0/src/runtime/metrics.go)).
These counters therefore step once per cycle. A 2-second Prometheus scrape gives
exact **per-cycle** deltas and correct per-run totals, but it cannot show GC
CPU second by second. For per-second GC CPU, use perf. The docs also call these
metrics "an overestimate, and not directly comparable to system CPU time"; the
design's planned cross-check against perf should expect a systematic gap.

### 2.3 runtime/trace (diagnostic runs only)

The execution tracer, parsed by `go tool trace` or programmatically with
`golang.org/x/exp/trace`, records these ranges and events (names from
[`internal/trace/event.go`](https://github.com/golang/go/blob/go1.27.0/src/internal/trace/event.go)
and `runtime/proc.go`):

* `stop-the-world (GC sweep termination)` and `stop-the-world (GC mark termination)`
  ranges: each STW with exact boundaries. Other reasons such as
  `stop-the-world (read mem stats)` are separate.
* `GC concurrent mark phase`: the global mark range, from which the cycle's
  begin and end are taken.
* `GC mark assist`: a **per-goroutine** range for every assist. This is the
  only signal that shows which goroutine, and so which request, paid for
  marking.
* `GC incremental sweep`: per-goroutine sweeping done during allocation.
* Metric events `/gc/heap/goal:bytes`, `/memory/classes/heap/objects:bytes` and
  `/sched/gomaxprocs:threads`, giving the heap sawtooth at event resolution.
* The `go tool trace` web UI has a minimum mutator utilisation (MMU) view
  (`/mmu` handler in `cmd/trace`).

Overhead: about 1–2% CPU "for many applications" since the tracer redesign
([Go blog](https://go.dev/blog/execution-traces-2024)). A high-RPS server
generates many goroutine events, so the overhead and trace size for *this*
workload should be measured, not assumed **(unverified for this workload)**.
`runtime/trace.FlightRecorder` (Go 1.25) keeps a ring buffer and writes the last
N seconds on demand ([Go 1.25 notes](https://go.dev/doc/go1.25)). That keeps
file sizes bounded, but the tracer overhead is the same while it runs. To
attribute assists to requests, a diagnostic build can wrap each request in
`trace.WithRegion` (or a task); this adds per-request cost and belongs only in
the diagnostic image.

The GC guide says execution traces show every GC latency source except the
write barrier ([GC guide](https://go.dev/doc/gc-guide)).

### 2.4 runtime.ReadMemStats

`ReadMemStats` calls `stopTheWorld(stwReadMemStats)`
([`mstats.go`](https://github.com/golang/go/blob/go1.27.0/src/runtime/mstats.go)).
Every call is an STW pause that shows up in `/sched/pauses/total/other` and in
traces. It should not be called in measurement runs; everything it provides is
available through `runtime/metrics`. The validity check could require
`/sched/pauses/total/other` to have only a handful of samples per run.

### 2.5 perf frames per stage (already collected; names checked in 1.26/1.27 source)

| Stage | Frames (function name or prefix, with children) |
| --- | --- |
| STW 1 / setup | `runtime.gcStart`, `runtime.stopTheWorldWithSema`, `runtime.startTheWorldWithSema`, `runtime.gcResetMarkState` |
| Mark, background | `runtime.gcBgMarkWorker` → `runtime.gcDrainMarkWorkerFractional` / `…Idle` / `…Dedicated` → `runtime.gcDrain` |
| Mark, assists | `runtime.gcAssistAlloc`, `runtime.gcAssistAlloc1`, `runtime.gcDrainN`, `runtime.gcParkAssist` |
| Mark, scanning (inside the above) | `runtime.markroot`, `runtime.scanstack`, `runtime.scanObject` (note the capital O in current source), `runtime.greyobject`; Green Tea: `runtime.scanSpan`, `runtime.tryDeferToSpanScan`, `internal/runtime/gc/scan.ScanSpanPacked*`, `…ExpandAVX512` |
| Write barrier | `runtime.gcWriteBarrier1`…`8`, `runtime.wbBufFlush`, `runtime.wbBufFlush1` |
| Mark termination | `runtime.gcMarkDone`, `runtime.gcMarkTermination` |
| Sweep | `runtime.bgsweep`, `runtime.sweepone`, `runtime.(*sweepLocked).sweep` (the design lists `mspan.sweep`, which no longer exists under that name) |
| Scavenger | `runtime.bgscavenge`, `runtime.(*pageAlloc).scavenge`, `runtime.(*pageAlloc).scavengeOne` |
| Allocation | prefix `runtime.mallocgc` (matches `mallocgc`, `mallocgcSmallScanNoHeader`, and the 1.27 `mallocgcSmallScanNoHeaderSC*`, `mallocgcSmallNoScanSC*`, `mallocgcTinySC*`, `…SlowPath`); `runtime.newobject`, `runtime.growslice`, `runtime.makeslice` |

In Go 1.27 the compiler calls the size-specialised routines directly
([Go 1.27 notes](https://go.dev/doc/go1.27)), so `runtime.mallocgc` may not
appear as a parent frame. The frozen frame list should match by prefix, and the
allocation smoke test should confirm this on the pinned version.

perf at 99 Hz on 2 vCPUs gives about 120k samples in a 10-minute window. That
is plenty per run but only tens to hundreds per mark phase, so per-phase CPU
from perf should be aggregated over all cycles in a run. If perf records with
`-k CLOCK_MONOTONIC`, and the service logs one paired
(`CLOCK_MONOTONIC`, `CLOCK_REALTIME`) reading at startup, each sample can be
tagged as *in mark*, *in sweep window*, or *outside GC* by joining against
gctrace intervals. Go's `nanotime` on Linux uses the monotonic clock
**(unverified that the gctrace `@` origin, `runtimeInitTime`, can be recovered
to better than about 1 ms from a log line written in `main`; the offset is
small but should be measured once)**.

### 2.6 Safety summary

| Signal | Overhead | Measurement runs? |
| --- | --- | --- |
| `gctrace=1`, `scavtrace=1` | One stderr line per cycle | Yes |
| `runtime/metrics` at a 2 s scrape | Small, no STW | Yes |
| perf 99 Hz with monotonic timestamps | Already budgeted and measured by the design | Yes (already) |
| `gctrace=2`, `gcpacertrace=1` | Counters in the scan loop; extra lines | Diagnostic only |
| `runtime/trace` / FlightRecorder | About 1–2% generically; unmeasured here | Diagnostic only |
| Per-request trace regions | Per-request cost | Diagnostic only |
| `ReadMemStats` | STW per call | Never |

---

## 3. Mapping ladder rungs to stages

"Live" means live heap; "scan" means scannable bytes and pointer-containing
objects. Directions are relative to `go-default` at the same entry count and
load unless stated. These are mechanism-based expectations to register in
`PREDICTIONS.md`, not results.

| Change | Stage(s) affected | Mechanism | Metric that should move | Expected direction |
| --- | --- | --- | --- | --- |
| **GOGC raised** (`go-tuned`) | Trigger / pacer; how often mark runs | Goal = live + (live + roots) × GOGC/100; more headroom between cycles at the same allocation rate | Cycles per minute (gctrace count, `/gc/cycles/automatic`); `/gc/heap/goal`; heap sawtooth amplitude; working set | Cycles ↓ roughly ∝ 1/GOGC; goal and working set ↑ |
| GOGC raised | Mark (per cycle) | Scan work depends on live scannable heap, not on GOGC | Mark CPU per cycle (gctrace cpu B+C+D); mark wall time per cycle | ≈ unchanged |
| GOGC raised | Mark (per second); assists | Fewer cycles, same work each, so less GC CPU over time; fewer seconds spent in mark | `/cpu/classes/gc/total` per run; share of seconds overlapping mark; spike-second share | ↓ |
| GOGC raised | Periodic GC | If heap-triggered cycles become rarer than every 2 min, the forced periodic GC takes over as the cycle source | Share of cycles inferred as periodic; gaps of about 120 s | ↑ (can reach 100%) |
| **GOMEMLIMIT** set below the GOGC goal | Pacer | Memory-limit goal binds; cycles get more frequent as heap nears the limit | `/gc/heap/goal` flat near the limit; cycles per minute; heap at GC start | Goal capped; cycles ↑ near the limit |
| GOMEMLIMIT too tight | Limiter | GC CPU exceeds about 50% over the 2×GOMAXPROCS-second window; limiter engages and heap goes over the limit | `/gc/limiter/last-enabled` ≠ 0; assist CPU; heap > limit; OOM risk | Should stay 0 for a valid tuned setting |
| GOMEMLIMIT | Scavenger | Runtime returns memory more aggressively to stay under the limit | `/cpu/classes/scavenge/*`; `/memory/classes/heap/released`; scavtrace eager KiB | ↑ when close to the limit |
| **Pointer → slab layout** (`go-pointerfree`) | Mark (per cycle) | Records, slabs and the `map[[16]byte]uint32` groups hold no pointers, so the cache is marked as a few large noscan objects instead of millions of scanned nodes and pointer-holding map groups | `/gc/scan/heap:bytes`; mark CPU per cycle; mark wall time; gctrace=2 objects scanned | ↓ strongly (expected orders of magnitude for the cache part; what remains is stacks, globals and per-request garbage) |
| Pointer → slab | Assists | Less scan work per byte of allocation (`assistWorkPerByte` falls) and shorter marks | Assist CPU per cycle and per run; trace `GC mark assist` ranges | ↓ |
| Pointer → slab | Write barrier | LRU moves and inserts write `uint32` indexes, not pointers; no barrier | perf `gcWriteBarrier*` / `wbBufFlush*` share during mark | ↓ to near zero for cache operations |
| Pointer → slab | Cycle frequency | Live heap size is similar (data is the same), so the goal is similar; per-entry allocations are gone, so allocation rate falls to per-request garbage only | `/gc/heap/allocs:bytes` rate; cycles per minute | Allocation rate ↓; cycles ↓ (possibly to periodic-only) |
| Pointer → slab | STW pauses | Pauses depend on stopping goroutines, not heap size | `/sched/pauses/total/gc` | ≈ unchanged (both sub-ms expected) |
| Pointer → slab | Sweep | Fewer small spans with garbage | perf sweep frames | ↓ (small to begin with) |
| **Cache-aside fills** (PUT after miss) | Allocation; trigger; assists | Pointer layout: 2 allocations per PUT (node + value). Slab layout: 0 per entry. Both: HTTP-layer allocations | `/gc/heap/allocs:objects` per request; cycles per minute; assist CPU | Higher miss rate → more cycles and assists in the pointer layout only |
| **Eviction churn** | Garbage volume; sweep; trigger | Each eviction in the pointer layout turns a node and a value into garbage; the slab layout reuses the slot | `/gc/heap/frees:bytes` rate; heap0 − heap2 per cycle | Pointer layout ↑ with churn; slab ≈ flat |
| **Load level** 25% → 70% → 110% | Assists; idle workers; scheduling | Higher allocation rate per second; less idle CPU, so idle workers do less and assists and fractional workers do more | Assist share of mark CPU; idle share; `/sched/latencies` p99 during mark | Assist share ↑ with load; idle share ↓ |
| **Entry count** 1M → 5M → 10M | Mark (per cycle); goal | Scan work ∝ pointer-heavy live heap; goal ∝ live | Mark CPU per cycle vs `/gc/scan/heap`; cycle spacing at fixed rate | Pointer layout: mark ∝ entries, cycles ↓ (larger goal); slab: mark ≈ flat |
| **Green Tea vs old marker** (not a rung; context) | Mark scanning | Span-based scanning of small objects | Mark CPU per cycle | ↓ for the pointer layout; slab mostly unaffected |

Two interactions are worth registering explicitly.

1. *Raising GOGC trades fewer marks for larger heaps, but each mark is just as
   long.* If latency spikes come from being inside a mark phase,
   `go-tuned` should reduce how often spikes happen, but not how large each one
   is. A per-cycle chart can show this; the run-level p99.9 alone cannot.
2. *The slab layout removes the work inside each mark, not the mark itself.*
   Cycles still happen (per-request garbage, plus the periodic GC), but they
   should be short and assist-light. If `go-pointerfree` still shows spike
   seconds aligned with GC cycles, the remaining cause is outside the cache's
   data structure (HTTP-layer garbage, STW stopping time, or the fractional
   worker taking a P).

---

## 4. The Rust side

Rust has no collection cycle. Its memory-management costs are spread over
individual operations and paid synchronously:

| Go stage | Nearest Rust counterpart | How to observe |
| --- | --- | --- |
| Allocation (`mallocgc`) | `mi_malloc` and friends via `__rust_alloc` | perf frames `mi_*`, `__rust_alloc` (already in the design's category) |
| Mark + sweep (reclaiming garbage) | `drop_in_place` → `__rust_dealloc` → `mi_free`, executed at the moment an evicted entry or replaced value is dropped, often inside the shard's mutex | perf frames; the design already separates deallocation from user drop logic |
| Assists (allocation-coupled GC work on the request path) | None in the Go sense, but each free costs time on the request that evicted it | Same perf category; per-request cost is folded into the request's own latency |
| Scavenger | mimalloc *purge*: unused OS pages decommitted after a delay (`MIMALLOC_PURGE_DELAY`, documented default 1000 ms in v3; `MADV_DONTNEED` by default) ([mimalloc README](https://github.com/microsoft/mimalloc)) | cgroup `memory.current` and working set; `madvise` syscall counts (perf tracepoint, diagnostic only); `mi_process_info` (current and peak RSS and commit) **(unverified which values the pinned Rust crate exposes; the v2 default purge delay is also unverified)** |
| Heap goal headroom | Allocator fragmentation and retained pages | Working set ÷ logical bytes; bytes per entry (already in the design) |
| STW | None | — |

A fair way to frame the comparison:

* **Compare totals, not phases.** The comparable quantities are
  memory-management CPU per 1,000 requests (same perf rule for both languages),
  bytes per entry, and latency percentiles. Phase charts are a Go-only
  explanation of *why* Go's totals and tail look the way they do; they are not a
  head-to-head chart.
* **Name the timing difference.** Go defers reclamation and batches it into
  cycles that overlap requests (cost appears as bursts of background CPU and
  assists). Rust reclaims at the point of release (cost appears as a small,
  steady addition to the request that evicts, possibly under a shard lock).
  Neither is free. The Rust cost can lengthen lock hold times, which the
  design's lock-wait category will pick up.
* **Give Rust the same spike statistic, without alignment.** Report the
  spike-second rate (seconds with p99.9 above 3× the run median) for every
  variant. Only Go gets the GC-alignment statistic, because only Go has a
  timeline to align with. If Rust has a similar spike rate, the spikes are not
  GC-specific.
* **Memory behaviour.** Go's working set follows a sawtooth up to the heap goal
  (or GOMEMLIMIT) and is lowered by the scavenger. mimalloc's follows live data
  plus retained and fragmented pages, lowered by purge. Plotting the working set
  over time for one run of each, on the same axes, shows this without
  suggesting any phase equivalence.

---

## 5. Proposal for the design (small)

The aim is to explain the ladder by stage while adding as little cluster time
and as few rules as possible. Most of the value comes from analysing data the
design already collects.

### 5.1 Additions to measurement runs (no new runs)

1. **Extend the exported `runtime/metrics` list** in `METRICS.md` and
   `internal/metrics` to the names in Section 2.2 (CPU classes split by
   assist, dedicated, idle and pause; scavenge CPU; `/gc/scan/*`;
   `/gc/heap/{goal,live}`; `/gc/cycles/*`; `/gc/limiter/last-enabled`;
   `/sched/pauses/{total,stopping}/{gc,other}`; `/sched/latencies`;
   `/gc/heap/allocs:bytes`; `/memory/classes/heap/*`). This resolves the
   design's "mark-assist CPU if available" verification marker: it is
   available, as `/cpu/classes/gc/mark/assist:cpu-seconds`. Document in
   METRICS.md that CPU classes step once per cycle.
2. **Add `scavtrace=1`** next to `gctrace=1` (one line per cycle).
3. **Fix the perf frame list**: `runtime.(*sweepLocked).sweep` instead of
   `mspan.sweep`; prefix match on `runtime.mallocgc`; add the Green Tea scan
   frames and `runtime.(*pageAlloc).scavenge*`. Split the Go memory-management
   category into sub-categories (mark-background, mark-assist, write barrier,
   sweep, scavenge, allocation, STW) for reporting. The frozen *total* rule is
   unchanged.
4. **Record perf with monotonic timestamps** and log one paired
   monotonic/real-time reading at startup, so perf samples can be joined to
   gctrace intervals.
5. **Validity check**: `/gc/limiter/last-enabled` must not change during the
   window for `go-tuned` and `go-pointerfree` settings chosen by the sweep (or
   the sweep records it as a disqualifier), and `/sched/pauses/total/other`
   must stay near zero.

### 5.2 Analysis code and charts (from existing run folders)

One module, `analysis/gc_phases.py`, with unit tests on fixed gctrace lines
from the pinned version. It parses gctrace into a per-cycle table (start,
three clock durations, assist, background and idle CPU, heap0/heap1/live,
goal, stacks, globals, inferred trigger kind) and joins it with per-second
histograms, Prometheus exports and timestamped perf samples. Charts:

| Chart | Shows | Data |
| --- | --- | --- |
| **Per-cycle phase timeline** (one representative window per rung) | STW, mark and sweep-window bars on a time axis, per-second server p99.9 above, spike seconds marked | gctrace + histograms |
| **Cycle anatomy** (stacked bar per rung, median over cycles) | Mark wall time; assist, background and idle CPU; STW | gctrace |
| **Cycles per minute per rung**, split heap-triggered vs inferred periodic | Effect of GOGC and layout on frequency | gctrace |
| **Mark CPU per cycle vs scannable heap** (scatter, entry-count sweep, coloured by layout) | Scan work ∝ pointer-heavy heap; slab flat | gctrace + `/gc/scan/heap` |
| **Assist utilisation during mark** (assist CPU ÷ mark wall time per cycle) vs per-second p99.9 inside and outside mark | Assist pressure and its latency link | gctrace + histograms |
| **Heap-goal sawtooth** | Heap at start/end, live, goal, GOMEMLIMIT, pod limit, working set over time | gctrace + metrics + cAdvisor |
| **Latency inside vs outside mark** (per-second p99.9 distributions) | Conditional effect of being in mark, per rung | gctrace + histograms |
| **perf CPU by GC sub-category, in mark vs out of mark** | Where GC CPU goes, including write barrier (invisible to traces) | perf + gctrace |
| **Rust companion**: working set over time; memory-management CPU per 1,000 requests | Steady per-operation cost vs batched cost | perf + cAdvisor |

All of these are descriptive. They explain the confidence-interval comparisons
already fixed in the design and add no new decision rules. The directional
predictions in Section 3 should be frozen in `PREDICTIONS.md` at C2, so the
explanatory charts are checked against stated expectations rather than read
after the fact.

### 5.3 A separate diagnostic campaign (optional, small)

Run it once, alongside the Datadog profiling session (C9), so it shares the
cluster setup. It uses separate images built with a `diag` tag; none of its
numbers enter the headline comparisons.

* **Go, 3 rungs × 5M entries × 70% × 2 blocks (6 runs):** `gctrace=2`,
  `gcpacertrace=1`, and `runtime/trace` captured with FlightRecorder snapshots
  (or full traces over windows chosen to include at least one heap-triggered
  and one periodic cycle), with per-request `trace.WithRegion` in the diag
  build. Output: assist time per request (the CDF of latency for requests with
  and without an assist), objects scanned per cycle (the direct form of "mark
  duration vs live pointer count"), and MMU curves per rung.
* **Overhead pair (3 runs):** `go-default` with the diag tracing off versus the
  six runs above, to publish the tracer's cost for this workload.
* **Rust, 2 layouts × 2 blocks (4 runs):** perf with an `madvise` tracepoint
  count and mimalloc process info, to show purge activity next to the working
  set.

**Cost:** about 13 runs × about 17 minutes ≈ 3.7 hours, plus one sizing
pre-run and setup, **≈ 5 cluster-hours** (the design's total goes from about
71 to about 76). Section 5.1 and 5.2 add no cluster time. Analysis code is one
module, its tests, and the chart functions.

**Ordering, by checkpoint:** 5.1 items 1–4 at C4/C5 (they affect images and
METRICS.md, which freeze at C2/C7); Section 3 predictions at C2; 5.2 charts
from C6 local runs onward; 5.3 only after C7, and only if credit remains after
the minimum publishable campaign.

### 5.4 Risks

| Risk | Effect | Mitigation |
| --- | --- | --- |
| **Go-version dependence** | gctrace format is "subject to change"; Green Tea is new (default since 1.26, opt-out status in 1.27 unverified); 1.27 size-specialised malloc changes frame names; worker split differs at other GOMAXPROCS values | Pin the version in the manifest; the parser fails on an unknown format; prefix-based frame rules plus the smoke test; state that phase results hold for one Go version at 2 Ps |
| **Measurement overhead** | Tracing and `gctrace=2` change timing; per-request regions add cost | Keep them out of measurement runs; publish the measured overhead pair |
| **Misreading CPU metrics** | `/cpu/classes` steps per cycle and overestimates; idle-worker GC CPU is not a cost to requests | Use them only per cycle or per run; always show idle separately; use perf for time series |
| **Periodic GC inference** | Not labelled anywhere; heuristic misclassification | Publish the rule (gap ≈ 120 s and heap below trigger) and the count per run; trace in the diagnostic runs can confirm a sample |
| **Clock alignment** | gctrace `@` has ms resolution and an unexposed origin | Paired clock log at startup; measure the offset once locally; treat sub-10 ms alignment as unreliable |
| **Small per-cycle samples** | perf at 99 Hz gives few samples per mark phase | Aggregate across cycles; never chart one cycle's perf breakdown alone |
| **Framing asymmetry** | Phase charts exist only for Go and could read as "Go has more costs" | Present them as an explanation of Go's totals; compare languages only on shared totals and the spike rate |
| **Scope creep** | Phase analysis invites per-phase tuning, more GOMAXPROCS values or more Go versions | Fix the chart list above; per-phase tuning, more cores and older Go versions stay in Future Improvements; the diagnostic campaign runs only when credit remains |
