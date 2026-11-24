> **Archived revision.** This is the design as it stood before the Stage 8
> scope cuts, kept because it contains the density-packing and memory-limit
> designs for a follow-on write-up. It is not maintained; its internal links
> and file names reflect the layout at the time. The current design is
> [`../cache-experiment.md`](../cache-experiment.md).

# The Big Cache: Measuring What Rust Changes in Memory Management Compared With Modern Go

*   **Author:** Radesh Govind
*   **Status:** Proposed
*   **Date:** 2026-11-23 (revised 2026-11-24 after design review)

--------------------------------------------------------------------------------

## Overview

This project models the workload of a documented production migration from
Go to Rust and measures, under controlled and repeatable conditions, what
changes. The workload is an in-memory LRU cache that holds millions of entries,
modelled on the Discord "Read States" service whose Go version showed periodic
latency spikes that the team attributed to garbage collection **(needs
verification)**. As understood before verification, Read States records, for
each user and channel, which messages the user has read: a very large set of
small records, read far more often than written, with a few hot keys. The
workload below (millions of small entries, Zipf-distributed keys, mostly
reads) is modelled on that; the mapping is confirmed or corrected at
checkpoint C1.

The cache is built in two memory layouts. In the **pointer layout**, each
entry is its own heap allocation linked to its neighbours by pointers. In the
**slab layout**, entry records live in a pre-allocated array and values in
size-class byte slabs, linked by integer indexes; records and slabs contain no
pointers, so the garbage collector does not scan them. The same cache contract
is implemented five times:

1.  **Go, default settings**: pointer layout, no tuning.
2.  **Go, tuned**: the same code with `GOGC` and `GOMEMLIMIT` chosen by a
    documented sweep.
3.  **Go, slab layout**: with `GOGC` and `GOMEMLIMIT` chosen by its own
    documented sweep.
4.  **Rust, pointer layout**: no garbage collector; node memory is freed
    explicitly when an entry is evicted.
5.  **Rust, slab layout**.

Each layout exists in both languages (decision D5), which keeps "the layout"
and "the language" from being confused:

| Rung | Variant name | Language | `CACHE_LAYOUT` | Layout |
| --- | --- | --- | --- | --- |
| 1 | `go-default` | Go | `pointer` | pointer layout |
| 2 | `go-tuned` | Go | `pointer` | pointer layout |
| 3 | `go-pointerfree` | Go | `slab` | slab layout |
| 4 | `rust-pointer` | Rust | `pointer` | pointer layout |
| 5 | `rust-index` | Rust | `slab` | slab layout |

Each version runs on Kubernetes in the cloud, under load from k6 on a separate
node pool, with results written to files committed to git. Datadog is used to
diagnose and present what happened, not as the source of the published numbers.
A second campaign reuses the same images to test pod density and behaviour at a
memory limit.

The output is a methodology page and one blog post. The author uses them as a
portfolio piece when talking with companies that run or plan Rust migrations.

## The Problem

"Rust has no garbage collector, so it has no GC pauses" is incomplete in three
ways:

*   **The mechanism is usually misnamed.** In current Go, stop-the-world pauses
    are typically under 1 ms even on large heaps (to be confirmed at C1). The cost on
    a large, pointer-heavy heap is mostly the concurrent mark phase: CPU spent
    scanning millions of pointers, and "mark assists" that slow request
    goroutines while that happens. A post that only measures pause time can show
    Go looking fine and explain the wrong thing.
*   **Go has improved and has its own fixes.** Since the Discord post, Go added a
    redesigned GC pacer, `GOMEMLIMIT`, and container-aware `GOMAXPROCS`
    **(needs verification for exact versions)**. The standard Go answer to a
    large cache is a pointer-free layout; a comparison against untuned,
    pointer-heavy Go would omit that standard fix. Go also forces
    a garbage collection if none has run for about 2 minutes **(needs
    verification for the pinned version)**; on a large heap that rarely
    changes, this forced cycle can be the main source of periodic spikes,
    which is why every measurement window spans several of these intervals.
*   **Benchmarks of this kind are easy to bias by accident.** Different libraries,
    hit ratios, CPU throttling, a load generator that hides slow responses
    (coordinated omission), or a profiler that costs more in one language than
    the other can each produce a gap that has nothing to do with the language.

The proposed solution is to treat the Go versions as a ladder rather than a
single opponent. Each rung shows how much of the gap Go can recover, and at
what cost in code complexity (measured as hot-path line count in
`HOTPATH_NOTES.md`). Whatever gap remains after the last Go rung is attributed
to the runtime and its memory management, together with the HTTP stack,
mutex and map differences listed in `FAIRNESS.md`; lock wait is measured
separately so it is not mistaken for a memory-management effect. The
remaining gap is published whatever its size.

## Goals

1.  **Reproduce the mechanism.** Show, with GC traces and runtime metrics, that
    default Go on a large pointer-heavy cache reports the CPU share spent on GC
    marking and whether p99.9 latency rises during GC cycles. The alignment
    measure is pre-registered and defined in `METRICS.md`: on the server-side
    per-second series, a "spike" second has p99.9 above 3 × the run's median
    per-second p99.9 (seconds with fewer than 1,000 requests are excluded);
    the statistic is the share of spike seconds that overlap a GC cycle, minus
    the mean of the same share after 100 circular random shifts of the GC
    timeline. Done when that measure is
    reported for every `go-default` run, whether or not it shows alignment, or
    the failure to reproduce is documented with its cause.
2.  **Measure the full ladder.** Report all five versions.
    *   **Conditions:** cloud dedicated-CPU nodes; all variants of a
        comparison in the same session on the same node; at least 5 blocks
        per main-sweep configuration (one "block" = all five variants once,
        in a freshly shuffled order); pods of 2 vCPUs. Load levels and outcome definitions
        are in the Run protocol.
    *   **Headline metric:** client-side p99.9 latency. Server-side p99.9 is
        reported with equal weight, and p99 alongside both. Pass/fail uses
        one fixed latency target; each variant's slowdown factor (p99 at a
        level divided by its p99 at 25%) is reported next to it.
    *   **Other metrics:** memory-management CPU share, measured the same way
        for both languages by an external CPU sampler in every measurement
        run (GC plus allocator for Go; allocator plus deallocation for Rust);
        working-set memory per pod and bytes per cache entry; CPU seconds per
        1,000 requests, where the cache-aside PUT after a miss counts as a
        request.
    *   **If credit allows (see Budget):** the **reference configuration** (5M
        entries at the 70% and 110% load levels) repeated on two more nodes,
        checked by the test in decision D9, and repeated with 4-vCPU pods, to show how core
        count changes the gap.
3.  **Sweep the cause, not one point.** Run the cache at 1M, 2M, 5M and 10M
    entries, so the shape of the curve explains the mechanism. The difference
    between pointer-heavy and pointer-free storage is covered by the layout
    pairs (pointer layout against slab layout), not by a separate axis.
4.  **Measure density and memory-limit behaviour** on the same images:
    *   pods per node, calculated from each variant's own measured memory and
        CPU, then checked by packing a real node after the main sweeps
    *   behaviour while the working set ramps toward and past the pod memory
        limit, with the application's bound deliberately set above it: the
        working-set fraction at the first sign of distress, the time and entry
        count reached before the OOM kill, and latency during the approach
    *   image size per language
5.  **Make every published number reproducible.** Each number in the post comes
    from a committed run folder with a manifest (git SHA, image digests,
    toolchain versions, node ID and type, kernel, cgroup version, transparent
    huge page mode, rules version). A reader can rerun the same configuration
    from the repository.
6.  **Make the comparison defensible.** Predictions are frozen before any
    implementation code; the latency target is frozen at C6 from local runs;
    the rule documents are frozen before any cloud comparison exists (see
    Rules versioning under Non-Functional Requirements).
7.  **Make the author able to explain the hot path.** Each language's hot path
    (entry layout, eviction, request handling; roughly 100–200 lines per
    layout) is annotated by the author, and a log records mistakes found in
    AI-generated code.
8.  **Show working use of Datadog.** Dashboards and profiles in Datadog are
    used to diagnose and illustrate the results in one dedicated profiling
    campaign, with Datadog profiler overhead measured once per language and
    published.

## Non-Goals

*   **The proxy experiment.** It begins only after the cache post is published,
    and its framing must be fixed first (see the "Proxy framing" Open
    Question).
*   **Library-based cache variants before checkpoint C7.** They follow the
    five hand-written core variants (decision D2).
*   **A third codebase for density.** Density and memory-limit work reuses the
    cache images.
*   **Any database.** Off the hot path it adds nothing; on the hot path it hides
    runtime differences.
*   **A Python or CPU-heavy parser experiment.**
*   **Publishing numbers from a laptop or local Kubernetes** (kind, k3d, Docker).
    Local runs check correctness and the pipeline only.
*   **Datadog as the system of record** for any published number, and the
    Datadog node agent on the nodes running the system under test (SUT nodes)
    during measurement runs.
*   **A live A/B switch at the load balancer as evidence.** It may appear as a
    labelled illustration only.
*   **A general verdict on Go versus Rust.** Claims cover this workload, these
    pinned toolchain versions, and this hardware.
*   **Compile-time safety claims.** A load test cannot measure them.
*   **A time schedule.** Work is ordered by checkpoints; the author sets the pace.

## Checkpoints

Work is ordered by checkpoints, not dates. Each checkpoint ends when its exit
criterion is met.

| # | Name | Exit criterion | Produces |
| --- | --- | --- | --- |
| C1 | Verify facts | Every C1 item (each "needs verification" marker and each Open Question marked "Answered at C1") confirmed or refuted with a source in `claims.yaml`; Datadog, `perf` and provider decisions written | `claims.yaml` |
| C2 | Write the rules | Rules v0 committed; `PREDICTIONS.md` frozen | `FAIRNESS.md`, `METRICS.md`, `RUN_PROTOCOL.md`, `PREDICTIONS.md` |
| C3 | Specify the cache | Conformance suite and hit-ratio simulation run against a stub | `SPEC.md`, `conformance/`, `sim/` |
| C4 | Build the five variants | All five pass conformance and allocation tests; hot paths annotated | Go and Rust code, `HOTPATH_NOTES.md`, `AI_LOG.md` |
| C5 | Build the harness locally | One local end-to-end run produces a valid run folder and chart | `harness/`, `campaigns/` |
| C6 | Reproduce the mechanism locally | Alignment measure computed and reported for local `go-default` runs; if there is no alignment, the cause is documented and a go/no-go decision recorded; latency target frozen |  local run folders, alignment chart |
| C7 | Core cloud campaigns | Starts with the smoke test (one variant per language, no comparison; rules v1 freeze). Ends when the minimum publishable campaign is committed and the remaining campaigns have run in Budget order until credit runs out; the post lists which ran | `TUNING.md`, run folders |
| C8 | Density and memory limit | Density packing and memory-limit ramp committed | run folders |
| C9 | Datadog profiling | Profiling campaign done; screenshots and overhead recorded | `posts/img/` |
| C10 | Methodology post | Published | `posts/methodology.md` |
| C11 | Cache post | Published with every external claim sourced | `posts/big-cache.md` |

Gated follow-ons: library-based variants start after C7; the proxy starts
after C11.

## Open Questions

*   **Datadog access**: Does the GitHub Student Pack Datadog offer include the
    Continuous Profiler and APM, and how many hosts? Does it produce symbolized
    CPU profiles for the Rust binary? Answered at checkpoint C1. No published number depends
    on the answer.
*   **CPU sampler on cloud nodes**: Does the chosen provider allow a privileged
    pod to run `perf` (or an eBPF profiler) on the SUT node, with frame
    pointers or DWARF unwinding working for both binaries, so that it produces
    symbolized CPU profiles for both? Answered at C1.
    Fallback order if not: change provider; then use an eBPF profiler (for
    example Parca or Pyroscope's eBPF agent) under the same frame rule; only
    as a last resort report the metric for Go only, in which case Goal 2's
    CPU-share bullet and the Rust rows of METRICS.md are reworded before any
    results and the post says so.
*   **Discord facts**: Date, Go version, entry count, spike interval, and whether
    tuning was tried. Answered at C1 and recorded in `claims.yaml`.
*   **Forced GC in current Go**: Does the roughly 2-minute forced GC still exist
    in the pinned Go version, and when does it fire? Answered at C1.
*   **Which Go collector**: Does the pinned Go version use the newer "Green
    Tea" collector (which marks memory span by span) by default? Answered at
    C1. `PREDICTIONS.md` states which collector the predictions assume; if
    Green Tea is the default, the post says so and the mechanism story is
    re-checked at C6.
*   **Map growth behaviour**: How do the pinned Go and Rust versions grow a
    map (all at once or incrementally)? The design pre-sizes every map, so
    this affects only the explanation **(needs verification)**.
*   **Node fit**: Do the chosen node types fit `go-default` at 10M entries?
    Estimate: about 3 GB live heap, up to about 7 GB peak under default
    settings, a pod limit near 9 GB, which fits a 16 GB node minus system
    reserve. Answered at C1; if it does not fit, the entry-count sweep stops
    at 5M and the post says so.
*   **Token scope**: What is the narrowest token the provider offers for
    tagging one cluster, and can storage keys be limited to one bucket?
    Answered at C1 **(needs verification)**.
*   **Proxy framing**: Did Cloudflare's Pingora and Linkerd's Rust proxy
    replace Go services? Pingora reportedly replaced NGINX (written in C), and
    linkerd2-proxy a Scala/JVM proxy **(needs verification)**. Answered before
    any proxy work (not a C1 item); until then the
    proxy is not framed as a Go-to-Rust migration.
*   **Cloud provider**: Does DigitalOcean Kubernetes offer dedicated-CPU nodes and
    a static CPU manager, which gives a Guaranteed pod with a whole-number CPU
    request exclusive vCPUs (sibling hyperthreads may still be shared; see the
    Sizing table notes), and how much Student Pack credit is available? If
    not, which provider? Answered at C1.
*   **Repository home**: Where the real git repository will live; the current
    folder is a temporary app workspace. Decided by the author before C5 (not
    a C1 item).

## Critical User Journeys (CUJs)

### CUJ 1: Run one measurement session end to end

*   **Given**: Both images are built, all five variants pass the conformance
    suite, a cloud account is configured, and rules v1 is frozen. (This
    describes every session after the first; the first session is the cloud
    smoke test, which freezes rules v1.)
*   **When**: The author starts one campaign naming the experiment, entry
    count(s), load levels and number of repetitions.
*   **Then**: A cluster is created with a time-to-live label; a campaign Job on
    the support pool calibrates the weakest variant's capacity, then runs every
    variant in shuffled round-robin blocks on the same SUT node, each run with a
    fresh pre-filled pod, warm-up and measurement window; each run lands in its
    own folder with a manifest and validity verdict and is uploaded to the
    permanent results bucket as soon as it is validated; the cluster is
    destroyed at the end, or by the out-of-cluster reaper if the session is
    abandoned, without losing any finished run.

### CUJ 2: A sceptical reader checks a number

*   **Given**: The cache post is published with a chart of p99.9 latency.
*   **When**: A reader follows the chart's link to the repository.
*   **Then**: They find the run folders behind the chart, the manifest with
    toolchain versions, node ID and rules version, and the command to rerun
    it; large raw files are public-read in the results bucket.

### CUJ 3: The author explains a result in an interview

*   **Given**: The post and annotated hot paths exist.
*   **When**: An interviewer asks why `go-pointerfree` closes most of the gap,
    or why it does not.
*   **Then**: The author can point to the GC events, the CPU-sampler profiles,
    the load-level and entry-count curves and the annotated code, and compare
    the outcome with their pre-registered prediction.

## Technical Plan (Architecture)

### Components

```text
 ┌──────────────── Developer laptop ────────────────┐
 │ repo: harness/ experiments/cache/ results/ docs/  │
 │ make local-*    → Docker with --memory/--cpus     │
 │                   (correctness + pipeline only)   │
 │ make session-up → Terraform; starts campaign Job  │
 │ make collect    → pulls from bucket; session-down │
 └───────────────┬───────────────────────────────────┘
                 │                       GitHub Actions "reaper" (cron):
                 │                       destroys clusters past their TTL
                 ▼                       tag or with a stale heartbeat tag
 ┌──────────── Cloud Kubernetes cluster (one per session, TTL tag) ─────────┐
 │                                                                           │
 │  load pool (dedicated CPU)        SUT pool (one dedicated-CPU node)       │
 │  ┌──────────────┐   HTTP/1.1      ┌───────────────────────────────────┐  │
 │  │ k6 pod(s)    │ ──────────────▶ │ cache pod: ONE variant at a time  │  │
 │  │ open model   │  keep-alive     │ go-default | go-tuned |           │  │
 │  └──────────────┘                 │ go-pointerfree | rust-pointer |   │  │
 │                                   │ rust-index                        │  │
 │                                   │ /metrics, /admin/histograms       │  │
 │                                   ├───────────────────────────────────┤  │
 │                                   │ CPU sampler DaemonSet (perf 99 Hz)│  │
 │                                   └─────────────────┬─────────────────┘  │
 │  support pool                                       │ metrics, profiles   │
 │  ┌──────────────────────────────────────────────────▼─────────────────┐  │
 │  │ campaign Job (runs the session, heartbeat tag, uploads each run)    │  │
 │  │ Prometheus (scrapes /metrics, kubelet/cAdvisor, k6) — D1             │  │
 │  │ Datadog agents: installed only for the profiling campaign           │  │
 │  └──────────────────────────────────┬──────────────────────────────────┘  │
 └─────────────────────────────────────┼────────────────────────────────────┘
                                       │ each validated run uploaded at once
                                       ▼
          permanent bucket (long-lived Terraform stack, never destroyed by a session)
                                       │ make collect
                                       ▼
          results/cache/<env>/<timestamp>-<sha>/  manifest.yaml, histograms/,
          profiles/, metrics/, derived.json (committed); large raw files stay
          in the bucket, with checksums in the manifest
                                       │ analysis (Python)
                                       ▼
                 derived.json → charts and tables → posts/
   Datadog (SaaS) ── screenshots, profiles ──▶ posts/   (explains, never the source)
```

1.  **Cache service (five variants, two images).** One HTTP service contract.
    Go builds one binary and Rust builds one binary; each binary contains both
    of its language's layouts, chosen at startup by `CACHE_LAYOUT`. The five
    variants are these two images run with different environment variables.
    Go builds a static binary onto `gcr.io/distroless/static`; Rust builds for
    glibc (`x86_64-unknown-linux-gnu`) onto `gcr.io/distroless/cc`, so Rust
    uses glibc's `memcpy` family rather than musl's, which is generally slower
    **(needs verification)**. Image size is reported per language.
2.  **Conformance suite.** A black-box test program that runs against any
    variant's HTTP API and checks behaviour: get, put, eviction order, byte
    bound, keys that land in the same shard, and hit count on a fixed replayed
    key sequence. All five variants must give identical hit counts on the same
    sequence, otherwise latency comparisons are confounded.
3.  **Load generator.** k6 with the `constant-arrival-rate` executor (open
    model), so slow responses do not reduce the offered load and hide latency
    (coordinated omission). k6 remote-writes summary statistics to Prometheus
    with its push interval set explicitly to 1 second (the default interval
    and per-second behaviour **need verification**); it does not write
    per-request output to git. The number of k6 virtual users is fixed by
    formula, the same for every variant at a level: `preAllocatedVUs =
    ceil(rate × latency target × 4)` and `maxVUs = 2 ×` that, recorded in the
    manifest, because dropped iterations depend on it. At levels up to 85%, a run is invalid if k6 drops iterations or if
    the k6 node's CPU, averaged over 10-second windows, rises above 70% at any
    point in the measurement window. The load pool is sized from the expected
    capacity so k6 stays within that limit, with memory sized for k6's
    end-of-run percentile storage; density packing adds k6 pods in
    proportion to the replicas. At the 110% overload level, dropped
    iterations and their count are the recorded outcome, not a reason to
    discard the run.
4.  **Campaign Job and orchestrator.** `make session-up` runs Terraform, then
    starts a Kubernetes Job on the support pool that executes a campaign file
    run by run, so the campaign does not depend on the laptop staying awake.
    The orchestrator is a Python program (`campaign.py`), because it shuffles,
    calibrates and resumes. The Job uploads each run folder to the permanent
    bucket as soon as the run is validated, and reads resume progress from
    the bucket. Every minute it updates a `heartbeat` tag on the cluster
    through the provider API, using the narrowest token the provider offers
    (its actual scope is a C1 check); the full-account token never enters the
    cluster. `make collect` pulls run folders from the bucket; `make
    session-down` destroys the cluster and reconciles Terraform state if the
    reaper already deleted it.
5.  **Infrastructure as code.** Terraform with the DigitalOcean provider (or
    the replacement provider chosen at C1), in two stacks:
    *   **Persistent stack**, created once and never destroyed by a session:
        the results bucket and the container registry.
    *   **Session stack**, one per session: the cluster with three node pools
        separated by taints and node selectors: `sut` (exactly one
        dedicated-CPU node for the main campaigns, 4 vCPU / 16 GB, or 8 vCPU /
        32 GB for the 4-vCPU check, sizes and prices to verify at C1, so every variant in a
        comparison runs on the same physical node; Guaranteed QoS with requests
        equal to limits), `load` (k6), `support` (campaign Job, Prometheus).
        The cluster carries provider-side `ttl-hours` and `heartbeat` tags.
6.  **Metrics collection (decision D1: in-cluster Prometheus).** Prometheus on
    the support pool scrapes the cache pod's `/metrics`, kubelet/cAdvisor
    (working set, CPU, CFS throttling) and k6 every 2 seconds. At the end of
    each run the campaign Job exports range queries for the run window into the
    run folder. A run is invalid if Prometheus reports missed scrapes for the
    SUT pod. For latency, Prometheus carries only k6's per-second series, used
    for live viewing and GC alignment; it is not the source of any latency
    percentile or GC timing (component 7).
7.  **Latency and GC timing (wall clock).** Two latency views are reported
    side by side:
    *   **Client side (k6):** run-level p99 and p99.9 come from k6's
        end-of-run summary, configured to report those percentiles and
        restricted by tag to the measurement scenario (warm-up excluded). They
        include queueing time, such as waiting while GC workers hold the CPU.
        k6's per-second series is used only to line latency up with GC events; run-level percentiles are never built by
        combining per-second ones.
    *   **Server side:** every variant records each request into an HDR
        histogram that rotates every second and is stamped with wall-clock
        time (`GET /admin/histograms`). Both languages stamp at the I/O
        layer: Go through a wrapped `net.Conn`, Rust through an `AsyncRead`
        shim around the `TcpStream`. The stamp is the moment a read returns
        the first bytes after the previous response was written (in Go,
        `net/http`'s background one-byte read counts as that read), so every
        request on a keep-alive connection gets its own start time. A
        conformance case with a pipelined, delayed keep-alive client checks
        it. The rule is written in FAIRNESS.

    A gap between client and server percentiles measures time outside the
    server's stamped span: kernel and runtime queueing, network and client
    overhead. GC
    cycle times come from `GODEBUG=gctrace=1` (each line carries its offset
    from process start) plus the wall-clock process start time the variant
    logs at startup; no extra GC poller is needed. Client and server series
    are aligned using the NTP offsets of the SUT and load nodes, recorded in
    the manifest. Request-path instrumentation is identical within each
    language and follows the same stamping rule in both (FAIRNESS).
8.  **CPU sampler (decision D6).** A privileged DaemonSet on the `sut` pool runs
    `perf record` at 99 Hz on the cache process for the whole measurement
    window of every measurement run, and folds the stacks with `inferno` (a
    Rust port of the flame-graph stack-folding scripts) into files stored in
    the run folder. CPU is split into categories by a reviewed, frozen frame
    list in METRICS.md, with a one-line reason per
    frame and the same counting rule for both languages:
    *   **Memory management, Go:** marking (`runtime.gcBgMarkWorker`,
        `runtime.gcAssist*`), sweeping (`runtime.bgsweep`, `runtime.sweepone`,
        `mspan.sweep`), write barriers (`runtime.gcWriteBarrier*`,
        `wbBufFlush`), scavenging (`bgscavenge`), allocation
        (`runtime.mallocgc`), and their children.
    *   **Memory management, Rust:** mimalloc (`mi_*`) and the deallocation
        part of `drop_in_place` frames (user drop logic excluded).
    *   **Lock wait, both:** Go `sync.(*Mutex).lockSlow`, `semacquire`,
        `runtime.lock2`; Rust futex and parking frames. Reported separately,
        because mutex implementations differ between the languages.

    When a sample matches more than one category, precedence is lock wait,
    then memory management, then other. The Go total is cross-checked against
    `runtime/metrics` GC CPU classes and the difference published. Unwinding:
    Rust is built with `RUSTFLAGS=-C force-frame-pointers=yes` and mimalloc with
    `-fno-omit-frame-pointer` (through `CFLAGS`); because the precompiled Rust
    standard library has no frame pointers, Rust is sampled with `perf
    --call-graph dwarf`. Smoke checks in each language confirm that a
    synthetic allocation loop, a contended-mutex loop and a drop-heavy loop
    each attribute more than 90% of samples to the expected category. Each run
    reports its share of truncated stacks, and page-fault counts (kernel-side
    memory work outside both categories). Sampler
    overhead (sampler on versus off) is part of the minimum campaign for one
    variant per language, and for every variant if credit allows; it is
    published.
9.  **Results store.** Each run writes an immutable folder
    `results/cache/<env>/<timestamp>-<sha>/`, where `<env>` is `local` or the
    cloud provider's short name (for example `do`), containing `manifest.yaml`,
    `histograms/`, `profiles/` (folded stacks), `metrics/` (Prometheus
    exports), `logs/` and `derived.json`. The campaign Job uploads each folder
    to the permanent bucket as soon as the run is validated; `make collect`
    pulls them and they are committed to git. Large raw files (perf data)
    stay in the bucket, with paths and SHA-256 checksums in the manifest.
    Charts are built only from run folders and refuse to mix environments.
10. **Analysis.** A Python script reads run folders and computes per-run
    metrics. Comparisons use paired differences within each block (variant
    minus baseline variant, measured minutes apart on the same node). Each
    comparison reports the mean paired difference with a Student's t 95%
    confidence interval (n − 1 degrees of freedom), which is the decision
    rule, and a descriptive sign count ("k of n blocks agree in sign"; with 5
    blocks an exact sign test cannot go below p = 0.0625, so it is not used as
    a test); an interval is published only with at least 5 blocks, and
    3-block cross-node results are reported as direction and size only. The
    comparisons that get confidence intervals are fixed:
    *   `go-tuned` minus `go-default`, and `go-pointerfree` minus
        `go-default`: how much Go recovers by tuning and by layout.
    *   `rust-pointer` minus `go-tuned` (pointer layout): the gap left after
        Go's best tuning.
    *   `rust-index` minus `go-pointerfree` (slab layout): the gap left after
        Go's best layout. **This is the headline comparison.**
    *   `rust-index` minus `rust-pointer`: how much layout matters without a
        garbage collector.

    Main-sweep results are stated as holding for one node; the cross-node
    check is judged by the test in decision D9.
11. **Datadog.** Not installed in measurement sessions. A dedicated profiling
    campaign installs the cluster agent and the node agent (profiler on) and
    runs each variant once to produce dashboards and profiles that explain and
    illustrate results. Datadog profiler overhead is measured once per
    language and published.
12. **Reaper.** A scheduled GitHub Actions workflow lists clusters through the
    provider API and destroys any cluster older than its `ttl-hours` tag or
    whose `heartbeat` tag is older than 30 minutes (see Cost control).
    `make extend-ttl` raises the tag.

### Cache service contract (same for all five variants)

| Endpoint | Purpose |
| --- | --- |
| `GET /v1/entries/{key}` | Return the value or 404. Counts as a read; updates recency. |
| `PUT /v1/entries/{key}` | Insert or replace. May evict least-recently-used entries until the byte bound holds. |
| `POST /admin/prefill` | Fill the cache inside the process with N deterministic entries (seeded), so pre-fill does not depend on network load. |
| `GET /admin/stats` | Entry count, logical bytes charged, hits, misses, evictions, per-shard lock acquisitions, and map growth since prefill (must be zero). Used by the conformance suite and validity checks. |
| `GET /admin/histograms` | Per-second HDR latency histograms with wall-clock timestamps since the last reset. |
| `POST /admin/reset` | Clear stats and histograms at the start of the measurement window. |
| `GET /metrics` | Prometheus text format: process and runtime metrics (see below). |
| `GET /healthz`, `GET /readyz` | Liveness, and readiness only after pre-fill finishes. |

*   **Keys and values.** Keys are exactly 16 bytes and stored inline in every
    layout (`[16]byte` in Go, `[u8; 16]` in Rust). Lookups always compare the
    full key; no layout is keyed by hash alone. Values are 100–300 bytes.
*   **Pointer layout** (Go rungs 1–2, Rust rung 4). Per entry, one heap node
    holding the inline key, timestamp and `prev`/`next` pointers, plus one
    separate heap allocation for the value: two allocations per entry in both
    languages. The shard map goes from key to node pointer (Go
    `map[[16]byte]*node`; Rust `HashMap<[u8; 16], NonNull<Node>>`).
*   **Slab layout** (Go rung 3, Rust rung 5). Entry metadata lives in a
    pre-allocated array of fixed-size records (inline key, timestamp, value
    size class and slot number, `prev`/`next` as `uint32` indexes), with no
    pointers. Values live in size-class slabs: four classes of 128, 192, 256
    and 320 bytes, each a large byte array split into fixed slots with its own
    free list. There is no compaction: an evicted entry's slot goes back on its
    class's free list. The shard map goes from key to record index (Go
    `map[[16]byte]uint32`, which contains no pointers, so the GC does not scan
    it; Rust `HashMap<[u8; 16], u32>`). Slabs grow in fixed chunks, so steady
    state has zero allocations per entry in both languages.
*   **Pre-sized maps.** Every shard map and record array, in every layout and
    both languages, is created before prefill with capacity = shard bound ÷
    the minimum per-entry charge (180 bytes), so no map resizes during
    warm-up or measurement (growth behaviour differs between the languages;
    see Open Questions).
*   **Bounded by a logical byte charge, the same in every layout.** Each
    entry is charged 16 (key) + its value length + 64 (a fixed metadata
    constant), whatever its physical layout. Each of the 64 shards has its own
    bound of `CACHE_MAX_BYTES / 64` and evicts from its own LRU list when the
    sum of its charges would exceed that bound; there is no global counter.
    Because the charge is identical, all five variants evict the same keys for
    the same request sequence (checked by the conformance replay); under load,
    hit ratios agree within the run-protocol tolerance. Physical cost (slot slack, allocator overhead, GC
    headroom) is only an output: bytes per entry is computed after each run as
    working set divided by entry count, and slot slack is reported separately.
    Go's `GOMEMLIMIT` has no Rust equivalent, so the bound lives in the
    application in both languages.
*   **Concurrency.** The FNV-1a 64-bit hash of the key selects one of
    `S = 64` shards (well above the core count, to keep lock contention low),
    each protected by the language's standard mutex (`sync.Mutex`,
    `std::sync::Mutex`). Every GET takes the shard lock because it updates
    recency. The lock pattern is identical in both languages; its cost is
    measured separately as lock wait (component 8).
*   **Map hashing.** Go uses its built-in map hash and Rust uses `HashMap` with
    the `ahash` hasher instead of the default SipHash. Both are expected to use
    AES instructions on amd64 **(needs verification)**; the Rust build enables
    `target-feature=+aes`, and each binary logs its hash path at startup,
    checked at C4.
*   **Workload.** Zipf-distributed keys (exponent fixed in `SPEC.md`, starting
    value 1.1) over a key space 1.25 times
    the number of entries the cache can hold, so some keys are always absent.
    90% of requests are GETs and 10% PUTs; in addition, every GET that misses
    is followed by a PUT of that key (cache-aside), which drives eviction. The
    expected hit ratio for these parameters is computed by a simulation that
    uses the same FNV-1a sharding and per-shard bounds, at C3, and written into
    SPEC.md. Entry counts 1M / 2M / 5M / 10M.
*   **Single source.** `experiments/cache/SPEC.md` is the single source for
    these values; the Technical Plan states the intent, and SPEC.md wins
    where they differ.

### Sizing table (how memory settings relate)

| Campaign | `CACHE_MAX_BYTES` | Pod CPU (= `GOMAXPROCS` = tokio workers) | Pod memory limit | `GOMEMLIMIT` (go-tuned and go-pointerfree) |
| --- | --- | --- | --- | --- |
| Main sweeps | entry count × the logical charge per entry (16 + mean value length + 64) | 2 vCPUs on a 4-vCPU / 16 GB dedicated node, same for all variants | per variant: its peak working set in the sizing pre-run × 1.3; go-tuned uses go-default's limit | each tuned variant's chosen fraction of its pod limit |
| 4-vCPU check | same as main sweeps | 4 vCPUs on an 8-vCPU / 32 GB dedicated node | as main sweeps, from a 4-vCPU sizing pre-run | chosen fraction of the pod limit |
| Density (pack) | 1M entries × logical charge | per variant, from its measured CPU, in whole vCPUs (see below) | per variant: its peak working set at 1M entries in the sizing pre-run × 1.15; go-tuned uses go-default's | chosen fraction of that limit |
| Memory-limit ramp | 1.5 × pod limit, as a logical charge (deliberately above it) | 2 vCPUs | fixed: go-default's main-sweep limit at 5M entries, the same for all variants | chosen fraction of the pod limit |

All four values are recorded in every manifest. The **sizing pre-run** is a
short run per variant (prefill, then 85% of the calibrated rate, the highest
non-overload level, for 5 minutes, at least two forced-GC intervals) after
calibration; `go-pointerfree`'s pre-run uses default GC settings, and its
tuning sweep then runs on the resulting limit; it measures peak working set and CPU per 1,000
requests and is not itself a measurement run. The vCPUs left on each node (2
of 4, or 4 of 8) run the kubelet, the CPU sampler and system pods, so they do
not share the cache pod's vCPUs. Cloud vCPUs are usually hyperthreads, so the
cache pod may still share physical cores, memory bandwidth and the last-level
cache with them; every manifest records `lscpu` topology and the pod's cpuset,
and this is stated as a limitation. `go-tuned` runs the same code as `go-default`,
so it always uses `go-default`'s pod limit; this removes any circular
dependency between its limit and its `GOMEMLIMIT`. In the ramp, maps and slab
arrays are pre-sized for the entry count expected at the pod limit (from the
main sweeps' bytes per entry), not from `CACHE_MAX_BYTES`, and each variant's
working set right after startup is recorded in the manifest. The pod limit
is the only hard stop; `GOMEMLIMIT` makes Go's collector work harder as memory
approaches it but does not stop growth.

### Runtime metrics exposed by each variant

*   **Go:** `runtime/metrics` values exported on `/metrics`: GC CPU seconds,
    GC pause histogram, GC cycle count, heap live bytes, heap goal, mark-assist
    CPU if available in the pinned version **(needs verification)**; gctrace
    lines and the wall-clock process start time (component 7).
*   **Rust:** process RSS, mimalloc process info (current and peak committed
    memory) **(needs verification of what the crate exposes)**, live bytes
    tracked by the cache, request counts.
*   **Both:** per-second HDR latency histograms; CPU-sampler profiles with
    memory-management and lock-wait categories; CPU
    seconds and memory working set from the cgroup (read by kubelet /
    cAdvisor), CFS throttled periods, `memory.events` `max`, `oom` and
    `oom_kill` counters, memory pressure (PSI); hits, misses and evictions.

### Run protocol

**Session.** One cluster, one SUT node. For each entry count in the campaign:

1.  **Calibrate.** Against `go-default` (expected to be the weakest variant),
    with a generous provisional pod limit (1.5 × the C6 local estimate). The
    ramp starts at 500 requests per second (adjusted at C6 if that is already
    near capacity); the client-side p99 of that first 60-second step is the
    **low-load value**. A coarse pass raises the rate by 25% every 60 seconds
    to find the region; a fine pass then raises it by 5% per step, each step
    held 150 seconds (longer than one forced-GC interval) and judged on its
    worst 10-second p99. The ramp stops at the first fine step whose worst
    10-second p99 exceeds 4 times the low-load value or where k6 drops
    iterations; the last step before it is 100% capacity for this session and
    entry count. Each other variant then runs a 150-second check at that rate;
    if any of them drops iterations or exceeds 4 times its own low-load p99,
    the ramp is repeated against that variant, the weakest one sets 100%, and
    the reason is recorded.
2.  **Size.** Run the sizing pre-run for each variant (see the sizing table
    notes; for the entry-count sweep, at that sweep's fixed rate), then set
    the per-variant pod limits.
3.  **Measure.** For each repetition, run all five variants in a freshly
    shuffled order (one "block"), each as:
    1.  Deploy the variant with fixed env vars (recorded in the manifest).
    2.  Call `/admin/prefill`; wait for `/readyz`.
    3.  Warm up at the target level for a fixed period; discard.
    4.  `POST /admin/reset`, then measure for a window long enough to span
        several forced-GC intervals (at least 10 minutes) and collect at least
        10^5 samples for p99.9.
    5.  Collect histograms, GC events, logs, profiles, Prometheus exports,
        `/admin/stats`, `kubectl describe` output.
    6.  Validity checks: no dropped iterations (except at the overload level),
        k6 node CPU at or under 70% (10-second averages), no missed scrapes,
        no restarts unless the run is a memory-limit run, CFS throttling recorded, and the measured
        hit ratio within 2 percentage points of the SPEC.md target and of the
        other variants in the block.
    7.  Delete the pod.

Every main-sweep and 4-vCPU configuration gets at least 5 blocks. The
cross-node check uses 3 blocks per node (direction and size only); tuning
sweeps use 3 shuffled blocks of all 12 settings; the memory-limit ramp uses 3
runs per variant. An OOM kill at the 110% level is recorded as the outcome
"killed". An OOM kill at 25–85% invalidates the run and triggers one resize
(pod limit × 1.2) and rerun, recorded in the manifest.

**Load levels.** 25%, 50%, 70% and 85% of capacity, plus 110% (overload).

**Latency target (decision D10).** One fixed client-side p99 threshold, the
same for every variant, decides every pass/fail in this design. The starting
value is 5 ms. At C6 it is checked against local runs (every variant's p99
at the 25% level must be at most half the target), then frozen as a dated v0
amendment. It is not changed after any cloud run; smoke-test amendments are
limited to operational items (timeouts, scrape configuration, sizing
factors).
Next to every pass/fail, each variant's **slowdown factor** (its p99 at that
level divided by its own p99 at 25%) is reported; it adds detail and decides
nothing. At the overload level the outcome per variant is:

*   **held:** client-side p99 at or under the target, and no dropped
    iterations
*   **degraded:** p99 over the target, but fewer than 1% of scheduled
    iterations dropped
*   **overloaded:** 1% or more of scheduled iterations dropped

**Campaign shape.** To keep run counts in check, two sweeps are run instead of
the full grid:

*   **Tuning sweeps:** 12 GOGC × GOMEMLIMIT settings for `go-tuned` and,
    separately, for `go-pointerfree`, at 5M entries and the 70% level, in 3
    shuffled blocks each (decision D12).
*   **Load-level sweep:** all five levels at 5M entries.
*   **Entry-count sweep:** 1M, 2M, 5M and 10M entries, all four at one fixed
    absolute request rate: 70% of go-default's capacity at 10M entries. Its 5M
    runs are separate from the load-level sweep's, because the rates
    differ. Changing only the entry count keeps heap size as the single
    variable. Each entry count's own calibrated capacity is reported as a
    separate result.
*   **Cross-node check (decision D9):** the reference configuration (5M
    entries, the 70% and 110% levels) repeated in two more sessions, each on a
    different SUT node, with 3 blocks each.
*   **4-vCPU check (decision D11):** the reference configuration with 4-vCPU
    pods, 5 blocks, in its own session.

### Budget

Estimated at about 17 minutes per measurement run (prefill, warm-up,
10-minute window, collection, teardown) and about 25 minutes per ramp run
(prefill plus up to 20 minutes). Figures are cluster-hours (one SUT, one load
and one support node). Calibration and sizing take about 75 minutes per
session and entry count (coarse pass about 20, fine pass about 12, other
variants' checks about 18, sizing pre-runs about 25). Larger nodes cost
more per hour; dollar figures per node type are added at C1 from the
provider's price list.

| Campaign | Runs | Cluster-hours (approx.) |
| --- | --- | --- |
| Tuning sweep, go-tuned (12 settings × 3 blocks, 5M, 70%) | 36 | 10 |
| Tuning sweep, go-pointerfree (12 settings × 3 blocks, 5M, 70%) | 36 | 10 |
| Load-level sweep (5 levels × 5 variants × 5 blocks) | 125 | 35 |
| Sampler overhead, minimum (1 variant per language × on/off × 3) | 12 | 3 |
| Entry-count sweep (4 counts × 5 variants × 5 blocks) | 100 | 28 |
| Cross-node check (2 levels × 5 variants × 3 blocks × 2 nodes) | 60 | 17 |
| 4-vCPU check (2 levels × 5 variants × 5 blocks, larger node) | 50 | 14 |
| Sampler overhead, remaining variants (3 × on/off × 3) | 18 | 5 |
| Memory-limit ramp (5 variants × 3 runs) | 15 | 6 |
| Density packing (5 variants, larger SUT pool) | 5 | 3 |
| Datadog profiling (5 variants, plus overhead once per language) | 9 | 3 |
| Calibration and sizing (about 12 session × entry-count combinations) | — | 15 |
| **Total** | **about 466** | **about 149** |

**Minimum publishable campaign:** both tuning sweeps, the load-level sweep at
5M entries and the minimum sampler-overhead runs, plus their calibration
(about 60 cluster-hours). The entry-count sweep, cross-node check, 4-vCPU
check, remaining sampler overhead, memory-limit ramp, density packing and
Datadog profiling (C9) follow in that order while credit remains. Each
campaign file repeats its own estimate, and
`session-up` refuses to start a campaign whose estimate exceeds the
configured remaining credit.

### Density and memory-limit campaign (same images)

*   **Pods per node (measure, then pack).** Runs after the main sweeps. Each
    variant's pod is sized from its own sizing pre-run at 1M entries: memory
    request equal to its peak working set × 1.15, CPU request equal to its CPU
    per 1,000 requests × the per-replica rate × 1.2, rounded up to the next
    whole core, so `GOMAXPROCS` and tokio workers are set to that whole
    number. The per-replica rate is fixed at 50% of go-default's
    capacity at 1M entries. The calculated density is node allocatable (after
    system-reserved memory and DaemonSets, reported separately) divided by
    that size. A packing run then starts replicas of one variant on an empty
    SUT node until the scheduler refuses, drives every replica at the same
    per-replica rate, and records whether all replicas keep client-side
    p99 under the latency target. Reported: calculated pods per node, packed
    pods per node, and whether the latency target held.
*   **Memory-limit ramp.** Using the ramp row of the sizing table, prefill
    brings the working set to about 50% of the pod limit; a scripted write
    phase then adds new entries at a fixed byte rate (set in
    `RUN_PROTOCOL.md`) until the pod is killed, while reads continue at 50% of
    capacity. A run that reaches 20 minutes without a kill stops and is recorded
    as "no kill". No core variant can shed memory, so every run is expected
    to end in an OOM kill; the results are how each runtime approaches it:
    *   the working-set fraction at the first sign of distress: client-side
        p99 above the latency target, memory-management CPU share above 25%
        of the pod's CPU over a 10-second window, a nonzero `max` counter in `memory.events`, or
        memory pressure (PSI `some avg10`) above 10%
    *   the time and entry count reached before the kill
    *   p99 and memory-management CPU share during the approach

    A memory-pressure-aware rung that could degrade gracefully stays in Future
    Improvements.
*   **Image size.** Compressed image size from the registry, reported per
    language (start-up time appears once, as a footnote).

### Architecture decisions

*   **D1. Metrics collection path: decided, in-cluster Prometheus** (author).
    Used for resource and runtime metrics; latency and GC timing come from
    each variant's own wall-clock records.
*   **D2. Cache implementation source: decided, hand-written sharded LRU from
    one shared spec** (author). The five core
    variants use a hand-written sharded LRU built from one shared spec, so
    layout and algorithm are held constant; the remaining differences between
    languages are listed in `FAIRNESS.md`. Library-based variants (Go
    `hashicorp/golang-lru`, Go `freecache` or `bigcache`, Rust `lru` or `moka`)
    are added as extra ladder rungs **only after checkpoint C7** (core cloud
    results committed). They form a "what a team would ship" section of the
    post and reuse the same contract, conformance suite and run protocol.
    As part of that post-C7 library work, `freecache` serves as a reference
    for Go rung 3 (labelled as not a strict LRU); if the hand-written slab is
    much worse, the post explains why. This reference is optional for the
    first publication.
*   **D3. HTTP stack: decided, base level** (author). Go uses `net/http` from
    the standard library. Rust uses `hyper` directly on the `tokio` multi-thread
    runtime, with no web framework. Both serve HTTP/1.1 with keep-alive and no
    TLS. Routing is a small hand-written match on method and path in both. The
    tokio worker-thread count and Go's `GOMAXPROCS` are both fixed equal to the
    pod's CPU limit, are not part of any tuning sweep, and are recorded in the
    manifest.
*   **D4. Rust memory allocator: decided, mimalloc** (author). Set as the
    global allocator through the `mimalloc` crate. Chosen because it is
    expected to return freed memory to the OS sooner than the system allocator
    and jemalloc, so the working set tracks live bytes during the memory-limit
    ramp, and because it is actively maintained (both **need
    verification**). Accepted cost: fewer allocator-level
    counters than jemalloc; Rust's memory-management CPU share comes from the
    CPU sampler (D6), and fragmentation is estimated as RSS minus live bytes
    tracked by the cache. All `MIMALLOC_*` environment settings are recorded in
    the manifest.
*   **D5. Rust LRU layout: decided, both layouts as core rungs** (author).
    Rust pointer layout (twin of Go rungs 1–2) and Rust slab layout (twin of
    Go rung 3), with the same per-entry allocation counts as their Go twins.
    The pointer layout uses `unsafe` raw pointers, allowed by FAIRNESS in that
    one file only (`lru_pointer.rs`), with every `unsafe` block commented with
    its safety argument and covered by tests run under Miri. Accepted cost: a
    fifth variant and one more hot path to explain.
*   **D6. Memory-management CPU share: decided, one external sampler for both
    languages** (author). `perf` at 99 Hz on the SUT node in every measurement
    run (overhead measured as in component 8), with one frame-counting rule for both languages. Accepted cost: small
    sampling overhead in every run (measured and published), node privileges,
    and frame attribution the author must be able to explain. Depends on the
    C1 check that the provider allows it; the fallback order is in Open
    Questions.
*   **D7. Density: decided, measure then pack** (author). Pods are sized per
    variant from their own measurements, and the calculated density is
    checked by packing a real node. Accepted cost: an extra campaign, and a
    headroom factor (1.15) that is fixed in FAIRNESS before results.
*   **D8. Load levels: decided, four levels plus overload** (author). 25%, 50%,
    70%, 85% and 110% of the weakest variant's measured capacity, with the
    overload level recorded as an outcome. Accepted cost: more runs; contained
    by crossing the load-level sweep (at 5M entries) with the entry-count sweep
    (at one fixed rate) instead of running the full grid.
*   **D9. Statistical scope: decided, main sweeps on one node, headline
    checked on three** (author). Full sweeps run on one SUT node per session,
    analysed as paired differences within blocks, and stated as holding for
    one node. If credit allows, the reference configuration is repeated on two
    more nodes to check whether the headline differences keep their sign and
    rough size on other nodes. The test: the same sign on every node, and each
    node's mean paired difference within a factor of 2 of the main-sweep
    value. "Other nodes" means other VMs; any host identifier the provider
    exposes is recorded. Accepted cost: about 17 extra cluster-hours;
    generality is checked for the reference configuration only.
*   **D10. Latency target: decided, a fixed number plus a reported slowdown
    factor** (author). One fixed client-side p99 threshold (starting at 5 ms,
    checked at C6) decides every pass/fail; the slowdown factor is reported
    alongside and decides nothing. Accepted cost: the threshold is a
    judgement call, mitigated by freezing it at C6 from local runs only.
*   **D11. CPU size: decided, 2 vCPUs for the main sweeps, 4 vCPUs for the
    reference-configuration check** (author). Fewer vCPUs are expected to make
    the collector's share of CPU more visible; the 4-vCPU check, if credit allows, shows how
    the gap changes with more. Accepted cost: about 14 extra cluster-hours.
*   **D12. Tuning rule for go-tuned and go-pointerfree: decided, best p99.9
    with ties broken by CPU** (author). Each tuned variant gets its own sweep
    (go-tuned on go-default's pod limit; go-pointerfree on its own). The rule
    is mechanical: among settings with no OOM kill, take the one with the
    lowest mean client-side p99.9; every setting whose mean p99.9 is within 5%
    of that best counts as tied, and among ties the lowest memory-management
    CPU share wins. The rule is in rules v0 before the sweeps run. Limitation,
    stated in the post: each variant is tuned once (5M entries, 70% level, 2
    vCPUs) and those settings are reused everywhere. Giving go-pointerfree its
    own sweep means the headline compares Rust with Go's best layout and best
    settings; accepted cost about 10 cluster-hours.

## Alternatives Considered

### Alternative A: Three independent experiments (cache, proxy, packed node), each with its own Go and Rust pair

*   **Pros**: Covers memory, CPU and per-connection cost; more breadth for a
    portfolio.
*   **Cons**: Six services to verify and explain; the packed-node experiment
    has no workload of its own; depth suffers in every experiment.
*   **Reason for Rejection**: Rejected by author: too wide for the depth the
    goal needs. The cache comes first; density reuses the cache images; the
    proxy starts only after the cache post is published.

### Alternative B: Cache and proxy both planned from the start, plus a density campaign over both

*   **Pros**: Three posts; adds the CPU and connection-handling story.
*   **Cons**: Twice the fairness and explanation work up front; a slip leaves
    two half-finished experiments.
*   **Reason for Rejection**: Rejected by author: spreads the work too thin.
    The proxy remains a gated follow-on.

### Alternative C: Live A/B switch at the load balancer from Go pods to Rust pods as the main evidence

*   **Pros**: Visual, easy to demo on a Datadog dashboard.
*   **Cons**: Mixes warm-up, autoscaling and leftover state into the numbers;
    keep-alive connections stay pinned to old pods, so the switch is not clean.
*   **Reason for Rejection**: Rejected by author: a demo, not a measurement.
    Isolated, repeated runs produce the evidence. A labelled memory-limit demo
    may appear in the post as an illustration.

### Alternative D: Two-way comparison (tuned Go against Rust)

*   **Pros**: Half the variants; simpler charts.
*   **Cons**: A Go practitioner will point out that the standard fix for a
    large cache is a pointer-free layout; skipping it makes the result easy to
    dismiss.
*   **Reason for Rejection**: Rejected by author in favour of the five-variant
    ladder (Go default, Go tuned, Go slab layout, Rust pointer layout, Rust
    slab layout).

### Alternative E: Datadog as the single source for measurements and profiling

*   **Pros**: One stack; strongest Datadog story.
*   **Cons**: Student-plan features unconfirmed; rolled-up metrics at 10-second
    or coarser granularity; data expires so post links die; agent and profiler
    overhead may differ by language.
*   **Reason for Rejection**: Rejected by author: published numbers come from
    committed raw files. Datadog diagnoses and presents.

### Alternative F: Publishing numbers from local Kubernetes (kind or k3d)

*   **Pros**: Free; fast iteration.
*   **Cons**: All "nodes" share one machine (inside a VM on macOS), so the load
    generator and the service compete for CPU and memory bandwidth.
*   **Reason for Rejection**: Rejected by author: local runs check correctness
    and the pipeline only.

### Alternative G: A CPU-heavy parser experiment with a Python baseline

*   **Pros**: Matches well-known Python-to-Rust stories (Pydantic v2, ruff,
    uv).
*   **Cons**: A third language and a different mechanism; widens scope.
*   **Reason for Rejection**: Rejected by author: scope. Possible separate post
    later.

### Alternative H: A database behind the service

*   **Pros**: Looks like a realistic service.
*   **Cons**: Network and database time dominate and hide runtime differences.
*   **Reason for Rejection**: Rejected by author: removed entirely.

### Alternative I: Metrics collection by direct file polling, or by Datadog API export (D1)

*   **Pros**: Direct polling has fewer moving parts and exact timestamps;
    Datadog export keeps one stack.
*   **Cons**: Direct polling is custom code to own and explain, with no live
    view; Datadog export depends on the unconfirmed plan and rolled-up data.
*   **Reason for Rejection**: Rejected by author in favour of in-cluster
    Prometheus, a standard, portable component.

### Alternative J: Library-only cache implementations (D2)

*   **Pros**: Matches what teams ship; less code.
*   **Cons**: Compares library designs (locking, metadata, eviction policy) as
    much as languages; `moka` is not a plain LRU.
*   **Reason for Rejection**: Rejected by author as the core comparison.
    Library variants return as extra rungs after checkpoint C7.

### Alternative K: Rust web framework (axum) or fastest servers (fasthttp, actix-web) (D3)

*   **Pros**: axum is closer to what Rust teams ship; fasthttp and actix-web
    give each side its fastest server.
*   **Cons**: axum adds a layer only on the Rust side; fasthttp changes Go's
    allocation behaviour, which is the thing being measured.
*   **Reason for Rejection**: Rejected by author in favour of base-level
    stacks (`net/http` and `hyper`).

### Alternative L: System allocator or jemalloc for Rust (D4)

*   **Pros**: System allocator is the out-of-the-box default; jemalloc gives
    detailed allocator counters for fragmentation analysis.
*   **Cons**: musl's malloc is slow under multi-threaded load and glibc can hold
    memory **(needs verification)**; jemalloc returns memory gradually (more to explain at the memory
    limit), and its upstream maintenance status needs verification.
*   **Reason for Rejection**: Rejected by author in favour of mimalloc.

### Alternative M: One Rust layout only (D5)

*   **Pros**: One fewer variant and one fewer hot path to explain; no `unsafe`
    if only the slab layout is kept.
*   **Cons**: Without a Rust twin for each Go layout, a gap between Go and Rust
    could come from the layout rather than the language.
*   **Reason for Rejection**: Rejected by author: keeping both layouts in both
    languages is what separates layout effects from language effects.

### Alternative N: Go-only GC metric, or separate profiling runs for CPU share (D6)

*   **Pros**: Go-only is simplest and avoids node privileges; separate
    profiling runs keep measurement runs free of sampling overhead.
*   **Cons**: Go-only loses the "where does the CPU go" comparison; separate
    runs measure the share under different conditions from the latency
    numbers.
*   **Reason for Rejection**: Rejected by author in favour of one external
    sampler in every measurement run.

### Alternative O: Calculate density only, and pack a node later (D7)

*   **Pros**: No extra campaign; uses numbers the main sweeps already produce.
*   **Cons**: A calculation, not a demonstration on a real node.
*   **Reason for Rejection**: Rejected by author in favour of measuring, then
    packing a real node.

### Alternative P: One fixed load level, or three levels without overload (D8)

*   **Pros**: Fewer runs.
*   **Cons**: One level hides how the gap changes with load, and whoever picks
    the level largely picks the result; without an overload level, behaviour
    beyond capacity is never measured.
*   **Reason for Rejection**: Rejected by author in favour of four levels plus
    an overload level.

### Alternative Q: Resample within one node only, or run every comparison on three nodes (D9)

*   **Pros**: One node is cheapest; three nodes for everything gives the
    strongest evidence across machines.
*   **Cons**: One node leaves "would another machine differ?" unanswered;
    three nodes for everything costs about three times the main sweeps.
*   **Reason for Rejection**: Rejected by author in favour of main sweeps on
    one node with the reference configuration checked on three.

### Alternative R: Latency target relative to Go's baseline, or each variant against itself (D10)

*   **Pros**: A baseline-relative target adapts to the hardware; a
    per-variant target measures degradation directly.
*   **Cons**: Harder to explain; a per-variant target gives faster variants a
    stricter bar.
*   **Reason for Rejection**: Rejected by author in favour of one fixed number
    for pass/fail, with the slowdown factor reported alongside.

### Alternative S: One CPU size only (2 or 4 vCPUs) (D11)

*   **Pros**: Fewer runs.
*   **Cons**: A single size leaves "would more cores close the gap?"
    unanswered.
*   **Reason for Rejection**: Rejected by author in favour of 2 vCPUs for the
    main sweeps and a 4-vCPU reference-configuration check.

### Alternative T: Pick tuned settings by tail latency alone, or by GC CPU alone (D12)

*   **Pros**: Simpler single-criterion rules.
*   **Cons**: Latency alone has no rule for near-ties; GC CPU alone may not
    give Go its best latency, the headline metric.
*   **Reason for Rejection**: Rejected by author in favour of best p99.9 with
    ties broken by memory-management CPU.

## Risks and Mitigations

| Risk | Type | Mitigation | Where enforced |
| --- | --- | --- | --- |
| Credit runs out partway through the campaigns | cost | Minimum publishable campaign first; later campaigns in Budget order; `session-up` refuses a campaign whose estimate exceeds remaining credit; the post lists which campaigns ran | Budget, C7 |
| A cluster is left running (laptop sleeps, script fails) | cost | Reaper destroys clusters past TTL or with a stale heartbeat; billing alert; validated runs already in the bucket | NFR: Cost control, component 12 |
| Provider does not allow `perf` on the SUT node | validity | Fallback order: change provider, then an eBPF profiler under the same frame rule, then Go-only with Goal 2 reworded before results | Open Questions, D6, C1 |
| Privileged sampler pod on the SUT node | security | Runs only on the `sut` pool of a short-lived cluster with no other tenants; the full-account token never enters the cluster | component 8, NFR: Secrets |
| Cloud token or storage keys broader than claimed | security | Narrowest available scope checked at C1; storage keys separate from the cloud token | Open Questions, NFR: Secrets |
| Variation between "identical" cloud hosts, shared physical cores | validity | All variants of a comparison on one node in shuffled blocks; cross-node check; `lscpu` and cpuset in every manifest; stated as a limitation | D9, Sizing table notes |
| The GC effect does not appear on 2-vCPU nodes | validity | Pre-registered alignment measure reported either way; 4-vCPU check; a negative result is published | Goal 1, D11 |
| The frame list misattributes CPU | validity | Frozen, reviewed frame list; Go total cross-checked against `runtime/metrics`; synthetic allocation-loop smoke check | component 8, METRICS |
| Bugs in AI-generated code, or `unsafe` Rust corrupting memory under load | failure | Conformance suite in CI; allocation-count tests; Miri for `lru_pointer.rs`; Go race detector; AI mistakes logged | C4, D5 |
| `go-default` does not fit the node at 10M entries | failure | C1 fit check; sweep capped at 5M if it does not fit, and the post says so | Open Questions |
| Calibration overstates a variant's capacity | validity | Fine steps longer than a forced-GC interval, judged on worst 10 seconds; cross-check of every variant | Run protocol |
| Public raw files incur egress cost | cost | Only large raw files are public; small results are in git; billing alert covers egress | Results store, NFR: Cost control |

## Non-Functional Requirements

*   **Measurement validity.** Every published number must come from a run that
    passed the validity checks in `docs/RUN_PROTOCOL.md`: no dropped k6
    iterations (except at the overload level, where they are the outcome),
    load-node CPU at or under 70% (10-second averages), no missed Prometheus
    scrapes of the SUT pod, no restarts unless the run is a memory-limit run,
    CFS throttling recorded, hit ratio within tolerance.
*   **Reproducibility.** Images are referenced by digest, toolchains are pinned
    (`go.mod` toolchain line, `rust-toolchain.toml`, `Cargo.lock`), and every run
    folder carries a manifest. Results are dated: "as of <date>, Go x.y /
    Rust x.y".
*   **Cost control.**
    *   One cluster per session, created and destroyed by script; `make
        session-down` is idempotent and safe to rerun.
    *   The cluster carries provider-side time-to-live and heartbeat tags; the
        out-of-cluster reaper destroys it if either expires, so a sleeping
        laptop cannot leave it running. Validated runs are already in the
        permanent bucket, so reaping loses at most the run in progress.
    *   The Budget section gives the total estimate and the minimum
        publishable campaign; each campaign file states its own estimate,
        checked against the remaining credit before the session starts.
    *   A billing alert is set on the cloud account; the orchestrator refuses
        to start if a cluster from a previous session still exists, or if the
        reaper has not run in the last 24 hours (scheduled jobs can be delayed
        or disabled). The reaper runs every 15 minutes and also deletes
        leftover volumes and load balancers.
*   **Repository size.** Only small files (histograms, folded profiles,
    Prometheus exports, logs, manifests) are committed. Large raw files stay in
    the permanent bucket with checksums in the manifest.
*   **Secrets.** The Datadog API key and cloud token are read from environment
    variables or a git-ignored `.env`, passed to the cluster as Kubernetes
    secrets, and never written into manifests or results. The full-account
    cloud token never enters the cluster: the campaign Job holds the narrowest
    token the provider offers for tagging (scope checked at C1) and separate
    storage keys for the results bucket. The reaper uses a GitHub Actions
    secret.
*   **Rules versioning.** `PREDICTIONS.md` freezes at C2 and is never
    edited. `FAIRNESS.md`, `METRICS.md` and `RUN_PROTOCOL.md` share one rules
    version: v0 at C2, amendable until the cloud smoke test at the start of
    C7, each change with a dated reason in a published changelog, frozen as v1
    there. After v1, any change bumps the rules version; run folders record
    it; the analysis refuses to mix versions. If a change alters code or
    settings used by only some variants, the author reruns those variants and
    records the reason in the changelog.

## Additional Resources

*   [`concepts_brief.md`](../../concepts_brief.md): the brainstorm brief this
    design responds to.
*   Design review findings applied in this revision:
    [`zoom_out_report.md`](../../zoom_out_report.md),
    [`zoom_out_report_rerun.md`](../../zoom_out_report_rerun.md),
    [`comprehension_report.md`](../../comprehension_report.md),
    [`critic_report.md`](../../critic_report.md).
*   [`system_design.excalidraw`](../../system_design.excalidraw): system design
    diagram.
*   Discord engineering blog post on moving Read States from Go to Rust
    **(URL and details to be recorded in `claims.yaml` at C1)**.

## Future Improvements

*   **Maximum throughput at a latency target** for every variant, beyond the
    single overload level.
*   **A uniform-key control run** at the same hit ratio, to bound how much
    hot-shard lock contention contributes.
*   **Library-based variants** (decision D2), after checkpoint C7.
*   **A memory-pressure-aware rung**: a cache that reads cgroup `memory.current`
    or pressure-stall information and lowers its own bound, in both languages.
*   **The proxy experiment**, after the cache post is published, once its
    framing is settled (see Open Questions).
*   **An older Go version** run on the same workload, to show how much Go itself
    improved since the Discord post.
*   **The labelled memory-limit demo** for the post: shrink the limit live and
    show each variant's behaviour.

--------------------------------------------------------------------------------

## Detailed Implementation Plan

The repository does not exist yet; every file below is created. Paths are
relative to the repository root. Files are grouped by the checkpoint that
creates them; the order of groups is the order of work. No production code is
written in this design.

### Target Files to Create/Modify

#### Repository root (C2)

1.  **`README.md`** (Create). *Changes:* what the project is, the five
    variants, how to run locally and in the cloud (`make` targets), where
    results and posts live, link to the methodology. *Why:* entry point for
    readers who follow a link from the post.
2.  **`Makefile`** (Create). *Changes:* targets `build` (two images), `test`,
    `conformance VARIANT=…`, `local-run VARIANT=… ENTRIES=… LEVEL=…`,
    `persistent-up` (bucket and registry, once), `session-up ENV=… CAMPAIGN=…`
    (creates the cluster and starts the campaign Job), `status`, `collect`
    (from the bucket), `extend-ttl HOURS=…`, `session-down`, `analyze`,
    `charts`. Each target calls a script under `harness/scripts/`. *Why:* one
    command per user journey (CUJ 1).
3.  **Repository hygiene files** (Create): `.gitignore` (`.env`, Terraform
    state, build outputs, caches) and `.env.example` (names of
    `DIGITALOCEAN_TOKEN`, `DD_API_KEY`, `DD_SITE`, `REGISTRY`,
    `OBJECT_STORE_BUCKET`, no values). *Why:* keep secrets and local state out
    of git.
4.  **`.github/workflows/ci.yml`** (Create). *Changes:* on push, build both
    images, run Go and Rust unit tests (including Miri for the pointer layout)
    and linters, start each of the five variants in Docker and run the
    conformance suite against it, run analysis tests, `terraform fmt -check`
    and `terraform validate`. *Why:* a variant that breaks the contract must
    fail before it reaches a cloud run.
5.  **`.github/workflows/reaper.yml`** (Create). *Changes:* scheduled every 15
    minutes; lists clusters through the provider API and destroys any cluster
    past its `ttl-hours` tag or whose `heartbeat` tag is older than 30
    minutes; also deletes leftover volumes and load balancers; never touches
    the persistent stack; records its last run time (checked by
    `session-up`); logs what it destroyed. *Why:* cost safety that does not depend on the laptop.

#### Rules and definitions (C1–C2, written before any service code)

6.  **`docs/METHODOLOGY.md`** (Create). *Changes:* the question, the ladder and
    layout pairs, environments, links to the rule files, how to read the
    charts. *Why:* the single page every post links to.
7.  **`docs/FAIRNESS.md`** (Create; v0 at C2, v1 at the C7 smoke test, a single
    version number after). *Changes:*
    *   toolchain pins and build flags (Go version and whether PGO is used;
        Rust release profile, LTO, `codegen-units`; frame pointers or debug
        info kept in both binaries, and `-fno-omit-frame-pointer` in `CFLAGS`
        for the mimalloc C build, so the CPU sampler can unwind stacks)
    *   allocator (mimalloc, D4); HTTP stacks (D3); hand-written LRU (D2)
    *   layouts: inline 16-byte keys, per-entry allocation counts for each
        layout, size classes 128/192/256/320 bytes, no compaction (D5)
    *   logical byte charge per entry, identical in every layout
    *   concurrency: `S = 64`, standard mutexes; map hashing (Go built-in,
        Rust `ahash`); every map pre-sized to final capacity before prefill
    *   instrumentation: identical request-path instrumentation (per-second
        HDR histograms, the server clock start point for each stack, admin
        endpoints) in every variant
    *   `unsafe` policy: allowed only in `lru_pointer.rs`, each block with a
        safety comment, Miri-tested
    *   threads: `GOMAXPROCS` and tokio workers fixed at the pod CPU (2 vCPUs
        in the main sweeps, 4 in the 4-vCPU check)
    *   transparent huge page mode pinned on the SUT node
    *   sizing table values, including the 1.3 and 1.15 headroom factors and
        go-tuned using go-default's pod limit
    *   the latency target (D10), set at C6
    *   the AI process: same prompt template per language, a cap on fix
        iterations, prompts committed under `docs/prompts/`
    *   tuning budget: `go-tuned` and `go-pointerfree` each sweep GOGC over
        50/100/200/400 and GOMEMLIMIT at 70/80/90% of their pod limit, at 5M
        entries and the 70% level only, in 3 shuffled blocks, with the chosen
        values reused for every entry count; winners chosen by D12; Rust uses `MIMALLOC_*` defaults with no
        tuning sweep, an accepted asymmetry stated in the post: Go gets a
        tuning rung because its runtime exposes GC settings, and mimalloc's
        defaults are what a Rust service ships with
    *   Rust target `x86_64-unknown-linux-gnu`, `RUSTFLAGS=-C
        force-frame-pointers=yes`, `target-feature=+aes`; k6 VU formula
    *   the shared rules version and a changelog of amendments with dates and
        reasons
    *Why:* a sceptical reader must see the rules were fixed before the results.
8.  **`docs/METRICS.md`** (Create). *Changes:* numbered definitions with formula
    and source:
    *   p99 and p99.9, client side from k6's end-of-run summary (measurement
        scenario only) and server side from the variants' per-second HDR
        histograms merged over the window, both reported
    *   memory-management CPU share and lock-wait share from CPU-sampler
        profiles, using the frozen frame list (one reason per frame) for each
        language; Go `runtime/metrics` GC CPU as a cross-check, with the
        difference published
    *   memory-limit ramp results: distress fraction, time and entry count
        before the kill, latency during the approach
    *   working set from cAdvisor; bytes per entry (including slot slack);
        CPU seconds per 1,000 requests; calculated and packed pods per node;
        image size per language
    *   the latency target and slowdown factor (D10), and the overload
        categories held, degraded and overloaded with their thresholds
    *   the memory-limit distress thresholds (latency target, 25%
        memory-management CPU share, nonzero `memory.events` `max`, PSI `some
        avg10` above 10%)
    *   the pre-registered GC alignment measure (Goal 1: series, spike
        threshold, minimum requests per second, 100 circular shifts, statistic)
    *   the CPU-category precedence rule and the truncated-stack share
    *   the applicability matrix (Primary = a headline result in the post;
        Secondary = supporting or explanatory; N/A with reason; GC rows are N/A
        for Rust, not zero)
    *   the fixed list of comparisons from component 10, with the headline
        comparison marked
    *   the rule for a real difference: the Student's t 95% confidence
        interval of the mean paired within-block difference (variant minus
        baseline variant, at least 5 blocks) excludes zero, with a descriptive
        sign count reported alongside; cross-node results must keep the same sign
    *Why:* every published number has one written definition.
9.  **`docs/RUN_PROTOCOL.md`** (Create). *Changes:* the session procedure
    (sizing pre-run, calibration, shuffled blocks, fresh pod per run), load
    levels and the overload rule, warm-up length, measurement window (at least
    10 minutes), minimum samples, at least 5 blocks, the sizing table,
    validity checks including the hit-ratio tolerance, the campaign shape
    (two crossing sweeps at a fixed rate for the entry-count sweep, plus the
    cross-node check), the budget and the minimum publishable campaign. *Why:* runs are comparable only if they follow one procedure.
10. **`docs/PREDICTIONS.md`** (Create, frozen at C2). *Changes:* for each layout
    pair, load level and entry count, the expected direction, rough size and
    mechanism. Never edited after results exist; outcomes are added in a
    separate column after C7. *Why:* interview material independent of who
    typed the code.
11. **`docs/AI_LOG.md`** and **`docs/prompts/`** (Create). *Changes:* dated
    entries of mistakes found in AI-generated code and how they were caught;
    the prompt templates and prompts used. *Why:* evidence of judgement and a
    FAIRNESS audit trail.
12. **`docs/designs/big_cache.md`** and **`docs/designs/SITEMAP.md`** (Create).
    *Changes:* this design and the design index. *Why:* the project's
    design-document folder.
13. **`experiments/cache/claims.yaml`** (Create at C1). *Changes:* one entry per
    external claim: source URL, quoted claim, conditions (Go version, entry
    count, date), verification status, our reproduction (filled after C7).
    Also holds the C1 checklist results: Datadog plan, Go forced-GC behaviour,
    provider capabilities, and whether `perf` works on the SUT node. *Why:* no
    figures quoted from memory.

#### Cache specification and conformance (C3)

14. **`experiments/cache/SPEC.md`** (Create). *Changes:*
    *   the HTTP contract, including `/admin/histograms` and `/admin/reset`
    *   LRU semantics: GET updates recency; PUT may evict until
        `CACHE_MAX_BYTES` holds; how bytes are counted for each layout
    *   both layouts exactly as in the Technical Plan, with per-entry
        allocation counts
    *   sharding (FNV-1a 64-bit hash modulo 64); map hashing per language
    *   the prefill algorithm (a seeded generator producing key `i` and a value
        whose length and bytes are derived from `i` and the seed, identical in
        both languages)
    *   the workload: Zipf parameter, key space 1.25 × capacity, 90/10 mix,
        cache-aside PUT on miss, and the target hit ratio computed by a
        simulation script (`experiments/cache/sim/hit_ratio.py`, committed)
    *   map and record-array pre-size (shard bound ÷ 180-byte minimum charge)
    *   `/admin/stats` and `/admin/histograms` JSON schemas; required
        `/metrics` names; environment variables (`CACHE_MAX_BYTES`,
        `CACHE_SHARDS`, `CACHE_LAYOUT`, `PORT`, `PREFILL_SEED`)
    *Why:* one spec, five implementations.
15. **`experiments/cache/sim/hit_ratio.py`** (Create). *Changes:* replays the
    seeded workload against a reference LRU with the same FNV-1a sharding,
    per-shard bounds and logical byte charge, and prints the expected hit
    ratio and eviction rate. *Why:* the target the validity check compares against.
16. **`experiments/cache/conformance/`** (Create, Go module: `go.mod`, `main.go`,
    `cases.go`, `README.md`). *Changes:* a black-box CLI that takes a base URL
    and checks: get/put round trip; 404 on miss; eviction order on a small byte
    bound; byte bound never exceeded; many keys forced into the same shard
    return their own values; prefill entry count and checksum of sampled
    values; a fixed replayed key sequence yields an exact expected hit count,
    identical for every variant; histograms reset and fill; a pipelined,
    delayed keep-alive client gets one latency sample per request; readiness
    only after prefill. Non-zero exit code on any failure. *Why:* behavioural
    equivalence across five implementations.

#### Go implementation (C4)

One Go module builds one binary; `CACHE_LAYOUT=pointer` or `slab` selects the
layout at startup. Variants `go-default` and `go-tuned` use `pointer`;
`go-pointerfree` uses `slab`.

17. **`experiments/cache/go/go.mod`** (Create). *Changes:* module path, pinned
    `toolchain` line; dependencies limited to the Prometheus client library and
    an HDR histogram library. *Why:* reproducible builds.
18. **`experiments/cache/go/cmd/cache/main.go`** (Create). *Changes:* read env
    vars, pick layout, pre-size maps, start the `net/http` server with a
    listener that wraps each `net.Conn` to stamp when request bytes first
    arrive, log configuration and process start time (wall clock) at
    startup. *Why:*
    entry point.
19. **`experiments/cache/go/internal/lru/pointer.go`** (Create, hot path).
    *Changes:* 64 shards, each a `sync.Mutex`, a `map[[16]byte]*node` and a
    doubly linked list; each node holds the inline key, timestamp, `prev`/`next`
    pointers and a `[]byte` value allocated separately. *Why:* rungs 1 and 2;
    the pointer-heavy layout the GC must scan.
20. **`experiments/cache/go/internal/lru/slab.go`** (Create, hot path).
    *Changes:* 64 shards, each with a `map[[16]byte]uint32`, a pre-allocated
    array of entry records (inline key, timestamp, size class, slot number,
    `prev`/`next` as `uint32`), and four size-class slabs (128/192/256/320
    bytes) with free lists, grown in fixed chunks, never compacted. *Why:*
    rung 3; almost nothing for the GC to scan.
21. **`experiments/cache/go/internal/lru/lru_test.go`** (Create). *Changes:*
    unit tests for both layouts: eviction order, byte bound, same-shard keys,
    slot reuse after eviction, concurrent get/put under the race detector, and
    an allocation test confirming zero allocations per PUT in steady state for
    the slab layout and two for the pointer layout. *Why:* catch bugs and
    prove the allocation counts the design claims.
22. **`experiments/cache/go/internal/server/server.go`** (Create, hot path).
    *Changes:* handlers for the contract; manual routing on method and path;
    request bodies read fully and closed; each request's time from the
    connection stamp to response written is recorded into the current
    per-second histogram. *Why:* contract and latency measurement.
23. **`experiments/cache/go/internal/prefill/prefill.go`** (Create). *Changes:*
    the seeded generator from SPEC.md. *Why:* identical data across languages.
24. **`experiments/cache/go/internal/metrics/`** (Create: `metrics.go`,
    `histograms.go`). *Changes:* export selected `runtime/metrics` values and
    cache stats (including per-shard lock acquisitions) in Prometheus format;
    rotate HDR histograms every second with wall-clock stamps and serve them
    on `/admin/histograms`. *Why:* METRICS.md sources and GC-to-latency
    alignment.
25. **`experiments/cache/go/Dockerfile`** and **`.golangci.yml`** (Create).
    *Changes:* multi-stage build with the pinned Go image, `CGO_ENABLED=0`,
    `-trimpath`, onto `gcr.io/distroless/static`; enabled linters. *Why:* same
    image pattern as Rust; CI.

#### Rust implementation (C4)

One Rust binary; `CACHE_LAYOUT=pointer` or `slab` selects the layout at
startup. Variant `rust-pointer` uses `pointer`; `rust-index` uses `slab`.

26. **`experiments/cache/rust/Cargo.toml`**, **`Cargo.lock`** and
    **`rust-toolchain.toml`** (Create). *Changes:* dependencies `hyper`,
    `hyper-util`, `http-body-util`, `tokio` (multi-thread runtime), `mimalloc`,
    `ahash`, `hdrhistogram`, `prometheus-client`; release profile per FAIRNESS
    (LTO, `codegen-units`, frame pointers kept); `CFLAGS` with
    `-fno-omit-frame-pointer` for the mimalloc build; pinned stable
    toolchain.
    *Why:* D3, D4, pinned builds.
27. **`experiments/cache/rust/src/main.rs`** (Create). *Changes:* set mimalloc as
    global allocator, read env vars, build the tokio runtime with worker
    threads equal to the CPU limit, start the hyper server, log configuration
    and process start time. *Why:* entry point.
28. **`experiments/cache/rust/src/lru.rs`** (Create). *Changes:* a small trait
    both layouts implement, selected at startup. *Why:* one binary, two
    layouts, same server code.
29. **`experiments/cache/rust/src/lru_pointer.rs`** (Create, hot path).
    *Changes:* 64 shards, each a `std::sync::Mutex` around a
    `HashMap<[u8; 16], NonNull<Node>, ahash::RandomState>` and a doubly linked
    list of separately allocated nodes (inline key, timestamp, raw `prev`/`next`
    pointers) with each value in its own `Box<[u8]>`. `unsafe` is confined to
    this file; each block states why it is sound. *Why:* rung 4, twin of Go
    rungs 1–2 (D5).
30. **`experiments/cache/rust/src/lru_slab.rs`** (Create, hot path). *Changes:*
    64 shards, each with a `HashMap<[u8; 16], u32, ahash::RandomState>`, a
    `Vec` of entry records (same fields as Go's), and four size-class slabs
    with free lists, grown in fixed chunks, never compacted; safe Rust only.
    *Why:* rung 5, twin of Go rung 3 (D5).
31. **`experiments/cache/rust/src/server.rs`** (Create, hot path). *Changes:*
    hyper service with manual routing over a `TcpStream` wrapped in an
    `AsyncRead` shim that stamps the first read after each response; full
    body read; time recorded into the
    per-second histogram. *Why:* contract and latency measurement.
32. **`experiments/cache/rust/src/prefill.rs`** (Create). *Changes:* the seeded
    generator from SPEC.md, byte-identical to Go's. *Why:* identical data.
33. **`experiments/cache/rust/src/metrics.rs`** (Create). *Changes:* process RSS,
    mimalloc process info, live bytes tracked by the cache, counters, in
    Prometheus format; per-second HDR histograms with wall-clock stamps on
    `/admin/histograms`. *Why:* METRICS.md sources.
34. **`experiments/cache/rust/tests/lru.rs`** (Create). *Changes:* the same unit
    cases as the Go tests, run against both layouts, including an allocation
    count check with a counting allocator in tests; `cargo +nightly miri test`
    runs them for `lru_pointer.rs`. *Why:* parity, proof of allocation counts,
    and evidence the `unsafe` code is sound.
35. **`experiments/cache/rust/Dockerfile`** (Create). *Changes:* multi-stage
    build for `x86_64-unknown-linux-gnu` with frame pointers and AES enabled,
    onto `gcr.io/distroless/cc`. *Why:* glibc avoids musl's slower `memcpy`
    family on the hot path.

#### Variants, load and campaigns (C4–C5)

36. **`experiments/cache/variants/{go-default,go-tuned,go-pointerfree,rust-pointer,rust-index}.yaml`**
    (Create). *Changes:* image (one per language), `CACHE_LAYOUT`, runtime env
    vars (for `go-tuned` and `go-pointerfree`, the values chosen by their
    tuning sweeps), and how
    requests and limits are derived from the sizing table. *Why:* a variant is
    data, not code; the manifest copies this file.
37. **`experiments/cache/TUNING.md`** (Create after the sweeps). *Changes:*
    both sweeps' results at 5M entries and the 70% level, the chosen values,
    links to run folders. *Why:* shows `go-tuned` and `go-pointerfree` were
    tuned by procedure, not by hand.
38. **`experiments/cache/load/calibrate.js`** (Create). *Changes:* the
    coarse-then-fine ramp from the Run protocol: first 60-second step sets the
    low-load value; coarse +25% steps of 60 seconds; fine +5% steps of 150
    seconds judged on their worst 10-second p99; stops at 4 times the low-load
    value or dropped iterations; reports the last passing rate as capacity. *Why:* defines 100% for the load levels.
39. **`experiments/cache/load/steady.js`** (Create). *Changes:*
    `constant-arrival-rate` at a rate passed in; seeded Zipf key choice over
    1.25 × capacity; 90/10 GET/PUT mix with a PUT after every GET miss;
    separate tagged warm-up and measurement scenarios; VUs set by the
    FAIRNESS formula; push interval 1 second; writes the end-of-run
    summary (p99 and p99.9 for the measurement scenario, the source of
    client-side percentiles) and remote-writes per-second statistics. *Why:* main
    measurement runs at every load level.
40. **`experiments/cache/load/memlimit-ramp.js`** (Create). *Changes:* after
    prefill to about 50% of the pod limit, writes new entries at the fixed
    byte rate from RUN_PROTOCOL until the pod is killed or 20 minutes pass
    ("no kill"), while reads continue at 50% of capacity. *Why:* memory-limit campaign.
41. **`experiments/cache/load/density.js`** (Create). *Changes:* drives every
    replica at the same per-replica rate through per-pod addresses and reports
    p99 per replica. *Why:* packing run.
42. **`experiments/cache/campaigns/*.yaml`** (Create). *Changes:* one file per
    campaign (two tuning sweeps, load-level sweep, entry-count sweep, cross-node
    check, 4-vCPU check, sampler overhead, memory limit, density, Datadog
    profiling), each
    listing configurations, the fixed rate rule, blocks, and a header with run
    count, estimated cluster-hours and cost, matching the Budget table. *Why:*
    input to the campaign Job; the plan of record for each session.

#### Harness (C5)

43. **`harness/local/compose.yaml`** (Create). *Changes:* run one variant with
    `--cpus` and `--memory`, a k6 container and Prometheus on Docker. *Why:*
    local correctness and pipeline checks.
44. **`harness/infra/terraform/do/persistent/`** and
    **`harness/infra/terraform/do/session/`** (Create, each with `main.tf`,
    `variables.tf`, `outputs.tf`). *Changes:* the persistent stack holds the
    results bucket and container registry and is never destroyed by a
    session; the session stack holds the DOKS (DigitalOcean Kubernetes) cluster with three node pools
    (`sut` with one dedicated-CPU node by default and a variable for the
    density campaign, `load`, `support`), taints and labels, provider-side
    `ttl-hours` and `heartbeat` tags, and a kubeconfig output. *Why:* cloud environment;
    replaced by another provider folder if C1 rules out DigitalOcean.
45. **`harness/k8s/sut.yaml.tmpl`** (Create). *Changes:* Deployment for one
    variant with node selector and toleration for `sut`, Guaranteed QoS sized
    from the sizing table, image by digest, env from the variant file,
    readiness probe on `/readyz`, `gctrace` to stderr for Go. *Why:* the system
    under test.
46. **`harness/k8s/sampler-daemonset.yaml`** (Create). *Changes:* privileged
    DaemonSet on the `sut` pool with `perf` installed; started and stopped by
    the campaign Job for each measurement window; writes perf data and folded
    stacks to a host path collected after the run. *Why:* D6.
47. **`harness/k8s/k6-job.yaml.tmpl`** (Create). *Changes:* Job on the `load` pool
    running the chosen script with the rate for the current level. *Why:* load
    generation.
48. **`harness/k8s/campaign-job.yaml.tmpl`** (Create). *Changes:* Job on the
    `support` pool with a service account allowed to manage the SUT
    Deployment and k6 Jobs, and a scoped cloud token (tag own cluster, write
    to the bucket); runs `campaign.py` for the given campaign file; updates
    the `heartbeat` tag every minute; uploads each validated run folder and
    its large files to the bucket. *Why:* campaigns survive the laptop going to sleep.
49. **`harness/k8s/prometheus/values.yaml`** (Create). *Changes:* Helm values for
    Prometheus on the `support` pool; scrape SUT `/metrics` and
    kubelet/cAdvisor every 2 seconds; remote-write receiver for k6. *Why:* D1.
50. **`harness/k8s/datadog/values.yaml`** (Create). *Changes:* Helm values
    used only by the Datadog profiling campaign: cluster agent on `support`,
    node agent on `sut` with the profiler on. *Why:* Datadog is not installed
    in measurement sessions.
51. **`harness/scripts/session-up.sh`**, **`session-down.sh`**, **`extend-ttl.sh`**
    (Create). *Changes:* Terraform apply/destroy for the session stack only;
    Helm installs; refuse to start if a cluster already exists or the
    campaign's estimated cost exceeds the configured remaining credit;
    reconcile Terraform state if the reaper deleted a cluster; verify node
    types; set the TTL tag; start the campaign Job. *Why:* cost control.
52. **`harness/campaign/campaign.py`** (Create, Python, runs inside the
    campaign Job). *Changes:* sizing pre-runs and calibration per entry count,
    then shuffled blocks run by run; resume from the runs already in the
    bucket without repeating them; update the heartbeat tag. *Why:* the
    session procedure from RUN_PROTOCOL.md.
53. **`harness/scripts/run.sh`** (Create). *Changes:* one run: render templates,
    deploy, prefill, wait ready, warm-up, reset, start sampler, measure, stop
    sampler, collect, write manifest, validate, delete pod. *Why:* the
    per-run steps.
54. **`harness/scripts/collect.sh`** (Create). *Changes:* inside the cluster,
    after each run: fetch `/admin/histograms` and `/admin/stats`; export
    Prometheus range queries and k6 per-second statistics; capture `kubectl
    logs --timestamps`; fetch sampler output and fold it with `inferno`; save
    `kubectl describe`, node ID and NTP offsets; record checksums of large
    files. On the laptop, `make collect` pulls run folders from the bucket. *Why:* everything a run needs lands in its folder.
55. **`harness/scripts/validate.sh`** (Create). *Changes:* apply the validity
    checks; write `valid: true/false`, the overload outcome when relevant, and
    reasons into the manifest. *Why:* invalid runs are kept but never charted.
56. **`harness/schema/manifest.schema.json`** (Create). *Changes:* required
    fields: git SHA, rules version, image digests, toolchains,
    variant file, env vars, sizing values, load level and rate, entry count,
    block and position, node ID and type, kernel, cgroup version, transparent
    huge page mode, `lscpu` topology, pod cpuset, startup working set, NTP
    offsets,
    k6 script hash, Datadog agent mode, sampler on/off, hit ratio, object
    storage checksums, validity. *Why:* every run is self-describing.

#### Analysis (C5, used from C6)

57. **`harness/analysis/pyproject.toml`** (Create). *Changes:* pinned
    dependencies (pandas or polars, matplotlib, hdrhistogram, jsonschema,
    pytest). *Why:* reproducible analysis.
58. **`harness/analysis/analyze.py`** (Create). *Changes:* read run folders,
    validate manifests, merge per-second histograms into run percentiles,
    compute `derived.json`, compute paired within-block differences with
    Student's t intervals and descriptive sign counts (`scipy.stats`), compute the
    GC alignment measure, compare cross-node results, refuse to mix
    environments or rules versions. *Why:* one shared script for every number.
59. **`harness/analysis/cpu_share.py`** (Create). *Changes:* read folded stacks,
    apply the frame list from METRICS.md, output memory-management and
    lock-wait CPU shares per run with the sample count. *Why:* D6.
60. **`harness/analysis/gc_align.py`** (Create). *Changes:* convert gctrace
    offsets to wall-clock time using the logged process start time, and
    overlay GC cycles on per-second client-side and server-side latency. *Why:* Goal 1.
61. **`harness/analysis/charts.py`** (Create). *Changes:* the post's charts
    from `derived.json` only: latency by load level, latency by entry count,
    CPU share, bytes per entry, density, memory-limit and overload outcomes.
    *Why:* charts are reproducible.
62. **`harness/analysis/tests/`** (Create). *Changes:* tests on small fixture
    run folders, including a mixed-environment case and a mixed-version case
    that must fail, and a frame-attribution case for each language. *Why:*
    analysis bugs would corrupt every result.

#### Results and posts (C6–C11)

63. **`results/README.md`** (Create). *Changes:* folder layout, where large
    files live, how to rerun a run from its manifest. *Why:* CUJ 2.
64. **`results/cache/<env>/<timestamp>-<sha>/`** (Created by runs). *Changes:*
    `manifest.yaml`, `histograms/`, `profiles/`, `metrics/`, `logs/`,
    `derived.json`. *Why:* source of truth.
65. **`posts/methodology.md`** (Create, C10), **`posts/big-cache.md`** (Create,
    C11) and **`posts/img/`**. *Changes:* the posts, Datadog screenshots as
    static images, a predictions-versus-outcomes table, and a "where the gap
    is smaller than the story claimed" section. *Why:* the deliverable.
66. **`experiments/cache/HOTPATH_NOTES.md`** (Create, C4). *Changes:* the
    author's line-by-line notes on `pointer.go`, `slab.go`, `server.go`,
    `lru_pointer.rs`, `lru_slab.rs` and `server.rs`, including the safety
    argument for each `unsafe` block. *Why:* interview readiness.

### Verification Plan

*   **Build:** `go build ./...` and `go vet ./...`; `cargo build --release`;
    `make build` produces two images, one per language; the five variants are
    these images with different env vars.
*   **Unit tests:** `go test -race ./...`; `cargo test`; `cargo +nightly miri
    test` (pointer layout); `pytest` in `harness/analysis`; allocation-count
    tests pass for every layout.
*   **Static analysis:** `golangci-lint run`; `cargo clippy -- -D warnings`;
    `cargo fmt --check`; `terraform fmt -check` and `terraform validate`.
*   **Conformance:** `make conformance VARIANT=<each>` passes for all five
    variants with identical hit counts on the replayed sequence. Required in
    CI.
*   **Pipeline (local, C5):** `make local-run VARIANT=go-default ENTRIES=1000000
    LEVEL=50` (locally, `LEVEL` is a percentage of a capacity calibrated
    against `go-default` on the same machine with the same rule as in the
    cloud) produces a run folder whose manifest validates against the
    schema, with histograms and gctrace, and `make charts` renders from it.
*   **Mechanism (C6):** a local `go-default` run shows GC cycles lining up with
    latency in the alignment chart; folded-stack profiles exist for every
    variant; `cpu_share.py` attributes frames in both languages; the synthetic
    allocation loop puts more than 90% of samples in allocator frames.
*   **Cloud smoke test (start of C7):** `perf` works on the SUT node; one short
    run per variant passes all validity checks and appears in the bucket; the
    reaper destroys a test cluster whose heartbeat tag is stopped, and the
    bucket survives. Rules v1 freezes here.
*   **Overhead checks:** sampler on versus off for each variant (C7), and
    Datadog profiler on versus off once per language (C9), with the
    differences recorded and published.

--------------------------------------------------------------------------------

<!-- Provenance: Indicates initial structural drafting via EGM. -->

*Drafted with EGM*
