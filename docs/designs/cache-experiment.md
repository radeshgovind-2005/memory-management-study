# The Cache Experiment: Measuring What Rust Changes in Memory Management Compared With Modern Go

*   **Author:** Radesh Govind
*   **Status:** In Review (EGM Goldfish review complete for decisions D1–D12;
    Goal 9 and decision D13, added afterwards, not yet reviewed; awaiting
    human review)
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
Pod density is calculated from the same measurements; packing a real node
and a memory-limit test are left for a follow-on post.

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
    *   **Conditions:** cloud dedicated-CPU nodes; all five variants of a
        block on the same node (one "block" = all five variants once, in a
        freshly shuffled order; the block is the pairing unit, and a long
        sweep may span several sessions); at least 5 blocks per main-sweep
        configuration; pods of 2 vCPUs. Load levels and outcome definitions
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
        entries at the 70% load level) repeated on two more nodes, checked by
        the test in decision D9.
3.  **Sweep the cause, not one point.** Run the cache at 1M, 5M and 10M
    entries, so the shape of the curve explains the mechanism. The difference
    between pointer-heavy and pointer-free storage is covered by the layout
    pairs (pointer layout against slab layout), not by a separate axis.
4.  **Report calculated density and image size**: pods per node calculated
    from each variant's own measured memory and CPU, and image size per
    language.
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
9.  **Explain the Go side stage by stage.** For each Go rung, show which stage
    of the garbage collection cycle it changes (how often cycles run, how much
    marking each cycle does, how much of that marking falls on requests as
    mark assists) and how that explains its latency, CPU and memory results.
    The expected direction for each rung and stage is frozen in
    `PREDICTIONS.md` at C2, and the charts are described in "GC stage
    analysis" below. Rust has no collection cycle, so the language comparison
    itself uses only shared totals; the stage charts explain Go's totals.

## Non-Goals

*   **The proxy experiment.** It begins only after the cache post is published,
    and its framing must be fixed first (see the "Proxy framing" Open
    Question).
*   **Library-based cache variants before checkpoint C7.** They follow the
    five hand-written core variants (decision D2).
*   **Density packing on a real node and the memory-limit test.** Both move to
    a follow-on post (decision D7); this post reports calculated density only.
*   **Core-count comparison.** All results are at 2 vCPUs per pod; more cores
    is a Future Improvement and a stated limitation.
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
| C6 | Reproduce the mechanism locally | Alignment measure computed and reported for local `go-default` runs; if there is no alignment, the cause is documented and a go/no-go decision recorded; latency target frozen | local run folders, alignment chart |
| C7 | Core cloud campaigns | Starts with the smoke test (one variant per language, no comparison; rules v1 freeze). Ends when the minimum publishable campaign is committed and the remaining campaigns have run in Budget order until credit runs out; the post lists which ran | `TUNING.md`, run folders |
| C8 | Calculated density | Density table computed from main-sweep measurements | `derived.json`, table |
| C9 | Datadog profiling and GC diagnostics | Profiling campaign and GC diagnostic campaign done; screenshots, traces and tracer overhead recorded | `posts/img/`, diagnostic run folders |
| C10 | Methodology post | Published | `posts/methodology.md` |
| C11 | Cache post | Published with every external claim sourced | `posts/big-cache.md` |

Gated follow-ons: library-based variants start after C7; the follow-on post
(density packing, memory-limit test, more cores) and the proxy start after
C11.

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
*   **Which Go collector**: Answered by research (see
    [`docs/research/gc-phases.md`](../research/gc-phases.md)): the "Green Tea"
    collector, which scans small objects span by span, is the default since
    Go 1.26. It changes how marking scans but not the phase structure, the
    pacer, the stop-the-world points or mark assists. Still open, answered at
    C1: the pinned Go version, and whether Green Tea can be switched off in it
    **(needs verification)**. `PREDICTIONS.md` states that the predictions
    assume Green Tea.
*   **Map growth behaviour**: How do the pinned Go and Rust versions grow a
    map (all at once or incrementally)? The design pre-sizes every map, so
    this affects only the explanation **(needs verification)**.
*   **Node fit**: Do the chosen node types fit `go-default` at 10M entries?
    Estimate: about 3 GB live heap, up to about 7 GB peak under default
    settings, a pod limit near 9 GB, which fits a 16 GB node minus system
    reserve. Answered at C1; if it does not fit, the entry-count sweep stops
    at 5M and the post says so.
*   **Storage key scope**: Can storage keys be limited to one bucket?
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
*   **Repository home**: Decided: the repository lives on GitHub as
    `memory-management-study`. The `github.com/<owner>/...` module-path
    placeholder is still replaced with the final owner before C3.

## Critical User Journeys (CUJs)

### CUJ 1: Run one measurement session end to end

*   **Given**: Both images are built, all five variants pass the conformance
    suite, a cloud account is configured, and rules v1 is frozen. (This
    describes every session after the first; the first session is the cloud
    smoke test, which freezes rules v1.)
*   **When**: The author starts one campaign naming the experiment, entry
    count(s), load levels and number of repetitions.
*   **Then**: A cluster is created with a time-to-live tag; a campaign Job on
    the support pool sizes each variant at the calibrated rates, then runs every
    variant in shuffled round-robin blocks on the same SUT node, each run with a
    fresh pre-filled pod, warm-up and measurement window; each run lands in its
    own folder with a manifest and validity verdict and is uploaded to the
    permanent results bucket as soon as its verdict (valid or not) is written; the cluster is
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
                 │                       destroys clusters past their
                 ▼                       time-to-live tag
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
 │  │ campaign Job (runs the session, uploads every run folder)           │  │
 │  │ Prometheus (scrapes /metrics, kubelet/cAdvisor, k6) — D1             │  │
 │  │ Datadog agents: installed only for the profiling campaign           │  │
 │  └──────────────────────────────────┬──────────────────────────────────┘  │
 └─────────────────────────────────────┼────────────────────────────────────┘
                                       │ each run uploaded once judged
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
    manifest, because dropped iterations depend on it. At the 25% and 70% levels,
    a run is invalid if k6 drops iterations or if
    the k6 node's CPU, averaged over 10-second windows, rises above 70% at any
    point in the measurement window. The load pool is sized from the expected
    capacity so k6 stays within that limit, with memory sized for k6's
    end-of-run percentile storage. At the 110% overload level, dropped
    iterations and their count are the recorded outcome, not a reason to
    discard the run.
4.  **Campaign Job and orchestrator.** `make session-up` runs Terraform, then
    starts a Kubernetes Job on the support pool that executes a campaign file
    run by run, so the campaign does not depend on the laptop staying awake.
    The orchestrator is a Python program (`campaign.py`), because it shuffles,
    calibrates and resumes. The Job uploads each run folder to the permanent
    bucket as soon as its verdict is written, and reads resume progress from
    the bucket. It holds only storage keys for the results bucket; no cloud
    token enters the cluster. `make collect` pulls run folders from the bucket; `make
    session-down` destroys the cluster and reconciles Terraform state if the
    reaper already deleted it.
5.  **Infrastructure as code.** Terraform with the DigitalOcean provider (or
    the replacement provider chosen at C1), in two stacks:
    *   **Persistent stack**, created once and never destroyed by a session:
        the results bucket and the container registry.
    *   **Session stack**, one per session: the cluster with three node pools
        separated by taints and node selectors: `sut` (exactly one
        dedicated-CPU node of 4 vCPU / 16 GB, size and price to verify at C1,
        so every variant in a comparison runs on the same physical node;
        Guaranteed QoS with requests equal to limits), `load` (k6), `support`
        (campaign Job, Prometheus). The cluster carries a provider-side
        `ttl-hours` tag set from the campaign's estimate plus 2 hours.
6.  **Metrics collection (decision D1: in-cluster Prometheus).** Prometheus on
    the support pool scrapes the cache pod's `/metrics`, kubelet/cAdvisor
    (working set, CPU, CFS throttling) every 2 seconds, and receives k6's
    statistics by remote write. At the end of
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
        layer: Go through a wrapped `net.Conn`, Rust through a wrapper implementing
        `AsyncRead` and `AsyncWrite` around the `TcpStream` (see the contract
        in the Detailed Implementation Plan). The stamp is the moment a read returns
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
    logs at startup; no extra GC poller is needed. `GODEBUG=scavtrace=1`
    adds one line per cycle about memory returned to the operating system.
    Nothing in gctrace or `runtime/metrics` labels the roughly 2-minute forced
    cycle as forced, so it is inferred from fields gctrace=1 prints: a cycle
    is periodic when it starts at least 115 seconds after the previous cycle
    and its heap at cycle start is below the lowest possible heap trigger,
    that is, the previous cycle's marked heap plus 45/64 of the way from
    there to this cycle's goal; the rule and the count per run are
    published. Client and server series
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
    *   **Memory management, Go:** reported in sub-categories that add up to
        the frozen total: STW (`runtime.gcStart`,
        `runtime.stopTheWorldWithSema`, `runtime.gcMarkTermination`,
        `runtime.gcMarkDone`), mark background (`runtime.gcBgMarkWorker` and
        `runtime.gcDrain*`), mark assists (`runtime.gcAssistAlloc*`,
        `runtime.gcParkAssist`), scanning including the Green Tea span
        scanner (`runtime.scanSpan`, `scan.ScanSpanPacked*`), write barriers
        (`runtime.gcWriteBarrier*`, `runtime.wbBufFlush*`), sweeping
        (`runtime.bgsweep`, `runtime.sweepone`,
        `runtime.(*sweepLocked).sweep`), scavenging (`runtime.bgscavenge`,
        `runtime.(*pageAlloc).scavenge*`) and allocation (matched by the
        prefix `runtime.mallocgc`, because Go 1.27 calls size-specialised
        allocators such as `runtime.mallocgcSmallScanNoHeaderSC…` directly),
        each with their children. Frame names are checked against the pinned
        version by the smoke test; the full list is in
        [`docs/research/gc-phases.md`](../research/gc-phases.md).
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
    --call-graph dwarf`. `perf` records with `-k CLOCK_MONOTONIC`, and each
    variant logs one paired monotonic and wall-clock reading at startup, so
    every sample can be tagged as inside a mark phase, inside a sweep window
    or outside GC by joining it with the gctrace intervals. Smoke checks in each language confirm that a
    synthetic allocation loop, a contended-mutex loop and a drop-heavy loop
    each attribute more than 90% of samples to the expected category. Each run
    reports its share of truncated stacks, and page-fault counts (kernel-side
    memory work outside both categories). Sampler
    overhead (sampler on versus off) is measured for one variant per language
    in the minimum campaign and published.
9.  **Results store.** Each run writes an immutable folder
    `results/cache/<env>/<timestamp>-<sha>/`, where `<env>` is `local` or the
    cloud provider's short name (for example `do`), containing `manifest.yaml`,
    `histograms/`, `profiles/` (folded stacks), `metrics/` (Prometheus
    exports), `logs/` and `derived.json`. The campaign Job uploads each folder
    to the permanent bucket as soon as its verdict is written; `make collect`
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
    provider API and destroys any cluster older than its `ttl-hours` tag
    (see Cost control under Non-Functional Requirements). `make extend-ttl`
    raises the tag.

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
    holding the inline key and `prev`/`next` pointers, plus one
    separate heap allocation for the value: two allocations per entry in both
    languages. The shard map goes from key to node pointer (Go
    `map[[16]byte]*node`; Rust `HashMap<[u8; 16], NonNull<Node>>`).
*   **Slab layout** (Go rung 3, Rust rung 5). Entry metadata lives in a
    pre-allocated array of fixed-size records (inline key, value
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
    SPEC.md. Entry counts 1M / 5M / 10M.
*   **Single source.** `experiments/cache/SPEC.md` is the single source for
    these values; the Technical Plan states the intent, and SPEC.md wins
    where they differ.

### Sizing table (how memory settings relate)

| Setting | Value |
| --- | --- |
| `CACHE_MAX_BYTES` | entry count × the logical charge per entry (16 + mean value length + 64) |
| Pod CPU (= `GOMAXPROCS` = tokio workers) | 2 vCPUs on a 4-vCPU / 16 GB dedicated node, same for all variants |
| Pod memory limit | per variant: its peak working set in the sizing pre-run × 1.3; `go-tuned` uses `go-default`'s limit |
| `GOMEMLIMIT` (`go-tuned`, `go-pointerfree`) | each tuned variant's chosen fraction of its pod limit |

**Sizing table notes.** All four values are recorded in every manifest. The
**sizing pre-run** is a short run per variant (prefill, then 70% of the rate
in use, for 5 minutes, at least two forced-GC intervals); it measures peak
working set (cgroup v2 `memory.peak`) and CPU per 1,000 requests and is not
itself a measurement run.
`go-tuned` runs the same code as `go-default`, so it always uses `go-default`'s
pod limit; `go-pointerfree`'s pre-run uses default GC settings, and its tuning
sweep then runs on the resulting limit. This removes any circular dependency
between a limit and its `GOMEMLIMIT`. The 2 vCPUs left on each node run the
kubelet, the CPU sampler and system pods, so they do not share the cache pod's
vCPUs. Cloud vCPUs are usually hyperthreads, so the cache pod may still share
physical cores, memory bandwidth and the last-level cache with them; every
manifest records `lscpu` topology and the pod's cpuset, and this is stated as
a limitation. All results are at 2 vCPUs per pod; how the gap changes with
more cores is a stated limitation and a Future Improvement.

### Runtime metrics exposed by each variant

*   **Go:** `runtime/metrics` values exported on `/metrics`: the GC CPU
    classes split into assist (`/cpu/classes/gc/mark/assist:cpu-seconds`),
    dedicated, idle and pause, plus scavenge CPU; `/gc/scan/*`;
    `/gc/heap/goal` and `/gc/heap/live`; `/gc/cycles/*`;
    `/gc/limiter/last-enabled`; `/sched/pauses/{total,stopping}/{gc,other}`;
    `/sched/latencies`; `/gc/heap/allocs:bytes`; `/memory/classes/heap/*`.
    The CPU-class values update only at a GC stop-the-world, so they step once
    per cycle and are used per cycle or per run, never per second. Also
    gctrace and scavtrace lines and the wall-clock process start time
    (component 7).
*   **Rust:** process RSS, mimalloc process info (current and peak committed
    memory) **(needs verification of what the crate exposes)**, live bytes
    tracked by the cache, request counts.
*   **Both:** per-second HDR latency histograms; CPU-sampler profiles with
    memory-management and lock-wait categories; CPU seconds and memory working
    set from the cgroup (read by kubelet / cAdvisor), CFS throttled periods,
    `memory.events` `oom_kill` counter; hits, misses and evictions.

### Run protocol

**Calibration (once per rate-setting entry count).** Capacity is calibrated
once at 5M entries (for the load-level sweep and the cross-node check) and
once at 10M entries (for the entry-count sweep's fixed rate). The resulting
absolute rates (iterations per second) are written into the campaign files and reused in
every later session, including sessions on other nodes. Calibration runs
against `go-default` (expected to be the weakest variant), with a generous
provisional pod limit (1.5 × the C6 local estimate). The ramp starts at 500
iterations per second (adjusted at C6 if that is already near capacity); the
client-side p99 of that first 60-second step is the **low-load value**. A
coarse pass raises the rate by 25% every 60 seconds to find the region; a fine
pass then raises it by 5% per step, each step held 150 seconds (longer than one
forced-GC interval) and judged on its worst 10-second p99. The ramp stops at
the first fine step whose worst 10-second p99 exceeds 4 times the low-load
value or where k6 drops iterations; the last step before it is 100% capacity.
Each other variant then runs a 150-second check at that rate; if any of them
drops iterations or exceeds 4 times its own low-load p99, the ramp is
repeated against that variant, the weakest one sets 100%, and the reason is
recorded.

**Session.** One cluster, one SUT node, at most 12 hours (campaign files are
split to fit). For each entry count in the session:

1.  **Size.** Run the sizing pre-run for each variant, then set the
    per-variant pod limits.
2.  **Measure.** For each repetition, run all five variants in a freshly
    shuffled order (one "block"), each as:
    1.  Deploy the variant with fixed env vars (recorded in the manifest).
    2.  Call `/admin/prefill`; wait for `/readyz`.
    3.  Warm up at the target level for a fixed period; discard.
    4.  `POST /admin/reset`, then measure for a window long enough to span
        several forced-GC intervals (at least 10 minutes; 6 minutes in tuning
        sweeps) and collect at least 10^5 samples for p99.9.
    5.  Collect histograms, GC events, logs, profiles, Prometheus exports,
        `/admin/stats`, `kubectl describe` output.
    6.  Validity checks: no dropped iterations (except at the overload level),
        k6 node CPU at or under 70% (10-second averages), no missed scrapes,
        no restarts, CFS throttling recorded, and the measured hit ratio
        within 2 percentage points of the `SPEC.md` target and of the other
        variants in the block.
    7.  Delete the pod.

Every main-sweep configuration gets at least 5 blocks. The cross-node check
uses 3 blocks per node (direction and size only); tuning sweeps use 3 shuffled
blocks of all 6 settings. An OOM kill at the 110% level is recorded as the
outcome "killed". An OOM kill at 25% or 70% invalidates the run and triggers
one resize (pod limit × 1.2) and rerun, recorded in the manifest.

**Load levels (decision D8).** 25% and 70% of capacity, plus 110% (overload).

**Latency target (decision D10).** One fixed client-side p99 threshold, the
same for every variant, decides every pass/fail in this design. The starting
value is 5 ms. At C6 it is checked against local runs (every variant's p99
at the 25% level must be at most half the target), then frozen as a dated v0
amendment. It is not changed after any cloud run; smoke-test amendments are
limited to operational items (timeouts, scrape configuration, sizing
factors). Next to every pass/fail, each variant's **slowdown factor** (its
p99 at that level divided by its own p99 at 25%) is reported; it decides
nothing. At the overload level the outcome per variant is:

*   **held:** client-side p99 at or under the target, and no dropped
    iterations
*   **degraded:** p99 over the target, but fewer than 1% of scheduled
    iterations dropped
*   **overloaded:** 1% or more of scheduled iterations dropped
*   **killed:** the pod was OOM-killed

**Campaign shape.** Instead of the full grid:

*   **Tuning sweeps (decision D12):** 6 GOGC × GOMEMLIMIT settings for
    `go-tuned` and, separately, for `go-pointerfree`, at 5M entries and the
    70% level, in 3 shuffled blocks each.
*   **Load-level sweep:** the three levels at 5M entries.
*   **Entry-count sweep:** 1M, 5M and 10M entries, all three at one fixed
    absolute rate: 70% of go-default's capacity at 10M entries.
    Changing only the entry count keeps heap size as the single variable. Its
    5M runs are separate from the load-level sweep's, because the rates
    differ.
*   **Cross-node check (decision D9):** the reference configuration (5M
    entries, 70% level) repeated in two more sessions, each on a different
    SUT node, with 3 blocks each.

### Budget

Estimated at about 17 minutes per measurement run (prefill, warm-up,
10-minute window, collection, teardown) and about 12 minutes per tuning run.
Figures are cluster-hours (one SUT, one load and one support node). The two
calibrations take about 75 minutes each; sizing pre-runs about 25 minutes
per session and entry count. Dollar figures per node type are added at C1
from the provider's price list.

| Campaign | Runs | Cluster-hours (approx.) |
| --- | --- | --- |
| Tuning sweep, go-tuned (6 settings × 3 blocks, 5M, 70%) | 18 | 4 |
| Tuning sweep, go-pointerfree (6 settings × 3 blocks, 5M, 70%) | 18 | 4 |
| Load-level sweep (3 levels × 5 variants × 5 blocks) | 75 | 21 |
| Entry-count sweep (3 counts × 5 variants × 5 blocks) | 75 | 21 |
| Sampler overhead (1 variant per language × on/off × 3) | 12 | 3 |
| Cross-node check (1 level × 5 variants × 3 blocks × 2 nodes) | 30 | 9 |
| Datadog profiling (5 variants, plus overhead once per language) | 9 | 3 |
| GC diagnostic campaign (6 Go + 3 overhead + 4 Rust runs) | 13 | 5 |
| Calibration (2) and sizing pre-runs | — | 6 |
| **Total** | **about 250** | **about 76** |

**Minimum publishable campaign:** both tuning sweeps, the load-level sweep,
the entry-count sweep and the sampler-overhead runs, plus calibration and
sizing (about 59 cluster-hours). This covers Goals 1–3, including the
entry-count chart that explains the mechanism; the GC stage charts come from
these same runs. The cross-node check, Datadog profiling and the GC diagnostic
campaign (C9) follow while credit remains. Each campaign file
repeats its own estimate, and `session-up` refuses to start a campaign whose
estimate exceeds the configured remaining credit.

### Calculated density and image size

*   **Calculated density.** For each variant, pods per node = node allocatable
    (after system-reserved memory and DaemonSets, reported separately) ÷ the
    variant's memory need at 1M entries (its peak working set in the sizing
    pre-run × 1.15), and separately ÷ its CPU need at a fixed per-replica rate
    (50% of go-default's capacity at 10M entries, from its CPU per 1,000
    requests × 1.2); the smaller of the two is reported. This is one table row
    in the post, computed from main-sweep measurements; packing a real node is
    a follow-on (decision D7).
*   **Image size.** Compressed image size from the registry, reported per
    language (start-up time appears once, as a footnote).

### GC stage analysis (Go only)

A Go GC cycle has five observable stages: a short stop-the-world at the start,
a concurrent mark phase (background workers, idle-time workers and mark
assists, with the write barrier on), a short stop-the-world at mark
termination, a concurrent sweep, and the background scavenger that returns
memory to the operating system. The pacer decides when the next cycle starts.
With `GOMAXPROCS=2`, the runtime starts no dedicated mark worker: its 25%
background target (half a vCPU) is met by a fractional worker and by idle
workers, so mark assists carry relatively more of the work than on larger
machines. Details, sources and the signals for each stage are in
[`docs/research/gc-phases.md`](../research/gc-phases.md).

Each rung is expected to change a different stage. Raising `GOGC` should make
cycles less frequent without shortening each mark, so latency spikes should
become rarer but not smaller. `GOMEMLIMIT` should cap the heap goal and make
cycles more frequent as memory nears the limit. The slab layout should remove
most of the scan work inside each mark, along with most write-barrier work and
per-entry allocation, so marks become short and assist-light while cycles
continue. These expectations, with the metric that should move for each, are
frozen in `PREDICTIONS.md` at C2.

`harness/analysis/gc_phases.py` turns gctrace, scavtrace, `runtime/metrics`,
latency histograms and timestamped `perf` samples into these charts, all from
the existing measurement runs:

| Chart | What it shows |
| --- | --- |
| Per-cycle phase timeline | STW, mark and sweep windows on a time axis for one representative window per rung, with per-second server p99.9 above and spike seconds marked |
| Cycle anatomy | Median mark wall time, assist, background and idle CPU, and STW per cycle, as one stacked bar per rung |
| Cycles per minute | Per rung, split into heap-triggered and inferred periodic cycles |
| Mark CPU vs scannable heap | Per cycle, across the entry-count sweep, coloured by layout |
| Assist pressure | Assist CPU divided by mark wall time per cycle, next to p99.9 inside and outside mark |
| Heap-goal sawtooth | Heap at cycle start and end, live heap, goal, `GOMEMLIMIT`, pod limit and working set over time |
| Latency inside vs outside mark | Distributions of per-second p99.9, per rung |
| GC CPU by stage | `perf` CPU per Go sub-category, inside vs outside mark, aggregated over all cycles in a run |
| Rust companion | Working set over time and memory-management CPU per 1,000 requests, on the same axes as Go |

The charts are descriptive: they explain the comparisons in component 10 and
add no new decision rules. Across languages, only shared totals and the
spike-second rate are compared, because only Go has a cycle to align with.

**GC diagnostic campaign (decision D13).** About 13 runs, in the C9 session
alongside Datadog profiling, using separate images built with a `diag` tag
whose numbers never enter the headline comparisons:

*   **Go, 3 rungs × 5M entries × 70% × 2 blocks (6 runs):** `GODEBUG=gctrace=2`
    and `gcpacertrace=1`, and `runtime/trace` captured with the flight
    recorder over windows that include at least one heap-triggered and one
    periodic cycle, with a `trace.WithRegion` around each request. These give
    assist time per request (latency distributions for requests with and
    without an assist), objects scanned per cycle, and minimum-mutator-
    utilisation curves per rung.
*   **Tracer overhead (3 runs):** `go-default` with the diagnostic tracing off,
    compared with the runs above, so the tracer's cost on this workload is
    published.
*   **Rust, 2 layouts × 2 blocks (4 runs):** `perf` with an `madvise`
    tracepoint count and mimalloc process information, to show when mimalloc
    returns memory next to the working set.

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
    and jemalloc, so the working set tracks live bytes more closely, and
    because it is actively maintained (both **need
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
*   **D7. Density: decided, calculated in this post; packing and the
    memory-limit test in a follow-on post** (author, revised after the second
    scope review). Calculated density uses the main-sweep measurements and a
    headroom factor (1.15) fixed in FAIRNESS before results. Accepted cost:
    this post has no real-node packing demonstration and no OOM section.
*   **D8. Load levels: decided, two levels plus overload** (author, revised
    after the second scope review). 25%, 70% and 110% of the weakest variant's
    measured capacity, with the overload level recorded as an outcome. This
    keeps the entry-count sweep inside the minimum campaign. Accepted cost:
    no 50% and 85% points on the load curve.
*   **D9. Statistical scope: decided, main sweeps on one node, headline
    checked on three** (author). Full sweeps run on one SUT node per session,
    analysed as paired differences within blocks, and stated as holding for
    one node. If credit allows, the reference configuration (70% level only)
    is repeated on two more nodes to check whether the headline differences keep their sign and
    rough size on other nodes. The test: the same sign on every node, and each
    node's mean paired difference within a factor of 2 of the main-sweep
    value. "Other nodes" means other VMs; any host identifier the provider
    exposes is recorded. Accepted cost: about 9 extra cluster-hours;
    generality is checked for the reference configuration only.
*   **D10. Latency target: decided, a fixed number plus a reported slowdown
    factor** (author). One fixed client-side p99 threshold (starting at 5 ms,
    checked at C6) decides every pass/fail; the slowdown factor is reported
    alongside and decides nothing. Accepted cost: the threshold is a
    judgement call, mitigated by freezing it at C6 from local runs only.
*   **D11. CPU size: decided, 2 vCPUs per pod** (author, revised after the
    second scope review). Fewer vCPUs are expected to make the collector's
    share of CPU more visible. The 4-vCPU comparison moves to Future
    Improvements; core count is stated as a limitation.
*   **D12. Tuning rule for go-tuned and go-pointerfree: decided, best p99.9
    with ties broken by CPU** (author). Each tuned variant gets its own sweep
    (go-tuned on go-default's pod limit; go-pointerfree on its own). The rule
    is mechanical: among settings with no OOM kill, take the one with the
    lowest mean client-side p99.9 over 6 settings (GOGC 100, 200, 400 ×
    GOMEMLIMIT 80%, 90% of the pod limit); every setting whose mean p99.9 is
    within 5%
    of that best counts as tied, and among ties the lowest memory-management
    CPU share wins. The rule is in rules v0 before the sweeps run. Limitation,
    stated in the post: each variant is tuned once (5M entries, 70% level, 2
    vCPUs) and those settings are reused everywhere. Giving go-pointerfree its
    own sweep means the headline compares Rust with Go's best layout and best
    settings; accepted cost about 4 cluster-hours.
*   **D13. GC stage analysis: decided, analysis of measurement-run data plus a
    small diagnostic campaign** (author). Stage-level charts are built from the
    gctrace, `runtime/metrics` and `perf` data the measurement runs already
    collect, with rung-by-stage predictions frozen at C2. Detailed tracing
    (`gctrace=2`, `gcpacertrace=1`, `runtime/trace`) runs only in a separate
    13-run diagnostic campaign at C9, with its overhead measured and
    published. Accepted cost: about 5 cluster-hours, one analysis module, and
    results that hold for one Go version at 2 vCPUs.

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

### Alternative O: Measure, then pack a real node, plus a memory-limit test, in this post (D7)

*   **Pros**: Real-node proof of density; a section on behaviour near the OOM
    limit.
*   **Cons**: A second campaign, checkpoint and two load scripts; every variant
    is expected to be OOM-killed, so the ramp is mostly descriptive.
*   **Reason for Rejection**: Rejected by author after the second scope
    review: moved to a follow-on post.

### Alternative P: One load level, or four levels plus overload (D8)

*   **Pros**: One level is cheapest; four levels give a fuller curve.
*   **Cons**: One level hides how the gap changes with load, and whoever picks
    it largely picks the result; four levels push the entry-count sweep out of
    the minimum campaign.
*   **Reason for Rejection**: Rejected by author in favour of 25%, 70% and an
    overload level.

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

### Alternative S: A 4-vCPU check in this post (D11)

*   **Pros**: Shows whether more cores close the gap.
*   **Cons**: About 14 cluster-hours, a larger node type and a sizing row.
*   **Reason for Rejection**: Rejected by author after the second scope
    review: moved to Future Improvements.

### Alternative T: Pick tuned settings by tail latency alone, or by GC CPU alone (D12)

*   **Pros**: Simpler single-criterion rules.
*   **Cons**: Latency alone has no rule for near-ties; GC CPU alone may not
    give Go its best latency, the headline metric.
*   **Reason for Rejection**: Rejected by author in favour of best p99.9 with
    ties broken by memory-management CPU.

### Alternative U: No stage analysis, or full tracing in every measurement run (D13)

*   **Pros**: No stage analysis keeps the plan smallest; tracing every run
    would give per-request assist data everywhere.
*   **Cons**: Without stage analysis, the post can report how much each rung
    changes but not why; tracing in measurement runs adds overhead to the
    numbers being compared.
*   **Reason for Rejection**: Rejected by author in favour of stage analysis
    from existing data plus a separate diagnostic campaign.

## Risks and Mitigations

| Risk | Type | Mitigation | Where enforced |
| --- | --- | --- | --- |
| Credit runs out partway through the campaigns | cost | Minimum publishable campaign first; later campaigns in Budget order; `session-up` refuses a campaign whose estimate exceeds remaining credit; the post lists which campaigns ran | Budget, C7 |
| A cluster is left running (laptop sleeps, script fails) | cost | Reaper destroys clusters past their TTL; TTL set from the campaign estimate; billing alert; validated runs already in the bucket | NFR: Cost control, component 12 |
| Provider does not allow `perf` on the SUT node | validity | Fallback order: change provider, then an eBPF profiler under the same frame rule, then Go-only with Goal 2 reworded before results | Open Questions, D6, C1 |
| Privileged sampler pod on the SUT node | security | Runs only on the `sut` pool of a short-lived cluster with no other tenants; no cloud token enters the cluster | component 8, NFR: Secrets |
| Storage keys broader than claimed | security | Bucket-scoped keys checked at C1; no cloud token in the cluster | Open Questions, NFR: Secrets |
| Variation between "identical" cloud hosts, shared physical cores | validity | All variants of a comparison on one node in shuffled blocks; cross-node check; `lscpu` and cpuset in every manifest; stated as a limitation | D9, Sizing table notes |
| The GC effect does not appear on 2-vCPU nodes | validity | Pre-registered alignment measure reported either way; a negative result is published; core count stated as a limitation | Goal 1, D11 |
| The frame list misattributes CPU | validity | Frozen, reviewed frame list; Go total cross-checked against `runtime/metrics`; synthetic allocation-loop smoke check | component 8, METRICS |
| Bugs in AI-generated code, or `unsafe` Rust corrupting memory under load | failure | Conformance suite in CI; allocation-count tests; Miri for `lru_pointer.rs`; Go race detector; AI mistakes logged | C4, D5 |
| `go-default` does not fit the node at 10M entries | failure | C1 fit check; sweep capped at 5M if it does not fit, and the post says so | Open Questions |
| Calibration overstates a variant's capacity | validity | Fine steps longer than a forced-GC interval, judged on worst 10 seconds; cross-check of every variant | Run protocol |
| GC internals differ between Go versions (gctrace format, Green Tea, frame names, worker split at other `GOMAXPROCS`) | validity | Go version pinned and recorded; the gctrace parser fails on an unknown format; prefix-based frame rules checked by the smoke test; stage results stated as holding for one Go version at 2 vCPUs | D13, component 8 |
| Diagnostic tracing changes the timing it observes | validity | Tracing only in the diagnostic campaign; tracer overhead measured and published | D13 |
| Periodic GC cycles misclassified | validity | Inference rule and per-run counts published; traces in the diagnostic campaign confirm a sample | component 7 |
| Go-only stage charts read as a one-sided comparison | validity | Languages compared only on shared totals and the spike-second rate; stage charts presented as explanation of Go's totals | Goal 9 |
| Public raw files incur egress cost | cost | Only large raw files are public; small results are in git; billing alert covers egress | Results store, NFR: Cost control |

## Non-Functional Requirements

*   **Measurement validity.** Every published number must come from a run that
    passed the validity checks in `docs/RUN_PROTOCOL.md`: no dropped k6
    iterations (except at the overload level, where they are the outcome),
    load-node CPU at or under 70% (10-second averages), no missed Prometheus
    scrapes of the SUT pod, no restarts, CFS throttling recorded, hit ratio
    within tolerance.
*   **Reproducibility.** Images are referenced by digest, toolchains are pinned
    (`go.mod` toolchain line, `rust-toolchain.toml`, `Cargo.lock`), and every run
    folder carries a manifest. Results are dated: "as of <date>, Go x.y /
    Rust x.y".
*   **Cost control.**
    *   One cluster per session, created and destroyed by script; `make
        session-down` is idempotent and safe to rerun.
    *   The cluster carries a provider-side time-to-live tag (campaign
        estimate plus 2 hours); the out-of-cluster reaper destroys it when it
        expires, so a sleeping laptop cannot leave it running. Validated runs are already in the
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
    secrets, and never written into manifests or results. No cloud token
    enters the cluster: the campaign Job holds only bucket-scoped storage keys
    (scope checked at C1). The reaper uses a GitHub Actions
    secret.
*   **Rules versioning.** `PREDICTIONS.md` freezes at C2; its prediction
    columns are never edited. `FAIRNESS.md`, `METRICS.md` and `RUN_PROTOCOL.md` share one rules
    version: v0 at C2, amendable until the cloud smoke test at the start of
    C7, each change with a dated reason in a published changelog, frozen as v1
    there. After v1, any change bumps the rules version; run folders record
    it; the analysis refuses to mix versions. If a change alters code or
    settings used by only some variants, the author reruns those variants and
    records the reason in the changelog.

## Additional Resources

*   [`concepts_brief.md`](../brainstorm/concepts_brief.md): the brainstorm
    brief this design responds to.
*   Design review findings and resolutions applied in this revision (stages 5–9):
    the [design reviews](../reviews/README.md), with each stage's findings and
    their resolutions.
*   [`system_design.excalidraw`](system_design.excalidraw): system design
    diagram.
*   Discord engineering blog post on moving Read States from Go to Rust
    **(URL and details to be recorded in `claims.yaml` at C1)**.

## Future Improvements

*   **Maximum throughput at a latency target** for every variant, beyond the
    single overload level.
*   **A uniform-key control run** at the same hit ratio, to bound how much
    hot-shard lock contention contributes.
*   **Library-based variants** (decision D2), after checkpoint C7.
*   **Follow-on post: density packing and the memory-limit test** (decision
    D7), with the measure-then-pack and memory-limit ramp designs kept in
    [`archive/cache-experiment-before-scope-cuts.md`](archive/cache-experiment-before-scope-cuts.md).
*   **More cores**: the reference configuration at 4 vCPUs (decision D11).
*   **The 50% and 85% load levels** (decision D8).
*   **A memory-pressure-aware rung**: a cache that reads cgroup `memory.current`
    or pressure-stall information and lowers its own bound, in both languages.
*   **The proxy experiment**, after the cache post is published, once its
    framing is settled (see Open Questions).
*   **An older Go version** run on the same workload, to show how much Go itself
    improved since the Discord post, including a collector comparison with
    Green Tea switched off if the pinned version still allows it.
*   **Stage analysis at other `GOMAXPROCS` values**, where Go does start
    dedicated mark workers, and per-stage tuning experiments.
*   **The labelled memory-limit demo**: shrink the limit live and show each
    variant's behaviour.

--------------------------------------------------------------------------------

## Detailed Implementation Plan

The repository does not exist yet; every file below is new. Paths are relative
to the repository root. Files are grouped by the checkpoint that creates them;
the order of groups is the order of work. No production code is written in
this design. The cross-language contract comes first because every
implementation and tool must match it exactly; file entries point to it rather
than repeat it. `SPEC.md` holds the byte-level detail and full JSON schemas.

### Cross-language contract

**Terms.** The **configured entry count E** (1M, 5M or 10M) sets
`CACHE_MAX_BYTES = E × 280` (bytes; 280 = 16 + mean value length 200 + 64) and
the prefill count N = E. The **resident entry count** is how many entries
remain after prefill, as computed by `hit_ratio.py`. Every **rate**, capacity
and load level is in iterations per second; request rate = rate × (1 + 0.9 ×
miss ratio). `value(i)` means the generator's value for key index `i`.

**HTTP.** Default `PORT` 8080. HTTP/1.1, keep-alive, no TLS, no server
timeouts in either stack.

| Request | Accepts | Returns |
| --- | --- | --- |
| `GET /v1/entries/{key}` | `key` = 32 lowercase hex characters (16 bytes) | 200 with raw value bytes, `Content-Type: application/octet-stream`, explicit `Content-Length`; 404 on miss; 400 on a malformed key |
| `PUT /v1/entries/{key}` | raw value bytes with `Content-Length` 100–300; optional header `X-Op: fill` | 204; 400 on a malformed key, a missing or chunked `Content-Length`, or a length outside 100–300 |
| `POST /admin/prefill` | JSON `{"count": N, "seed": S}` (`S` defaults to `PREFILL_SEED`) | 202 at once; prefill runs in the background; 409 if prefill already ran; 400 on bad JSON |
| `GET /admin/stats` | — | JSON: `entries`, `charged_bytes`, `hits`, `misses`, `evictions`, `shard_lock_acquisitions` (array of 64), `map_growth_since_prefill`, `layout`, `hash_path` (`"aes"` or `"fallback"`) |
| `GET /admin/histograms` | — | JSON array of `{"second": <unix seconds>, "count": n, "hist": "<base64>"}` for closed seconds since the last reset (the current partial second is excluded) |
| `POST /admin/reset` | — | 204; clears `hits`, `misses`, `evictions`, `shard_lock_acquisitions` and histograms; never touches `entries`, `charged_bytes` or the growth baseline |
| `GET /metrics` | — | Prometheus text; names listed in `METRICS.md` |
| `GET /healthz`, `GET /readyz` | — | 200 when alive / when prefill has finished; otherwise 503 |

Unknown paths return 404 and wrong methods 405. Entry requests during prefill
return 503. Prefill inserts are not counted as hits, misses or evictions.

**Cache operations.** Both languages expose the same interface (Go interface
`Cache`, Rust `dyn` trait):

*   `Get(key *[16]byte, dst []byte) (n int, ok bool)` / `fn get(&self, key:
    &[u8; 16], dst: &mut [u8]) -> Option<usize>`: copies the value into the
    caller-owned buffer **while holding the shard lock**. The handler reuses one
    300-byte buffer per connection; nothing is allocated per request. Every
    layout copies, including Go's pointer layout, so all layouts do the same
    work.
*   `Put(key *[16]byte, val []byte)` / `fn put(&self, key: &[u8; 16], val:
    &[u8])`: copies `val`; the caller keeps ownership.

**Cache semantics.** The shard is FNV-1a 64-bit of the 16 key bytes, modulo
`CACHE_SHARDS`, which must be 64 (any other value fails at startup); FNV-1a is
hand-written in both languages. Shard bound = floor(`CACHE_MAX_BYTES` / 64);
startup rejects `CACHE_MAX_BYTES` below 64 × 380. A PUT computes the new
entry's charge (16 + value length + 64); for a replace it applies the
difference from the old charge, then evicts least-recently-used entries from
that shard until the shard's charged bytes are at most the shard bound. A
replace moves the key to the front. Entries carry **no timestamp**.

*   **Pointer layout record:** a heap node with the inline key and `prev`/`next`
    pointers, plus a separate heap buffer for the value. Nodes are not reused.
*   **Slab layout record:** an array entry with the inline key, size class,
    slot number and `prev`/`next` as `uint32`. Four size classes, 128, 192,
    256 and 320 bytes; a value goes to the smallest class that fits. Each class
    grows in 64 KiB chunks holding floor(65536 / class) slots, the remainder
    unused. Free lists exist for both slots and records. On a class change or
    an eviction, the old slot is freed before a new one is taken, in both
    languages.
*   **Pre-size:** shard maps and record arrays are created with capacity =
    shard bound ÷ 180 (the minimum charge: 16 + 100 + 64).

**Allocation counts** (cache layer only; slab counts assume no chunk growth
during the measured call; an evicting PUT is a new key whose insertion evicts
one or more entries).

| Case | Pointer layout | Slab layout |
| --- | --- | --- |
| GET hit | 0 | 0 |
| GET miss | 0 | 0 |
| PUT new key | 2 | 0 |
| PUT replace, same length | 1 | 0 |
| PUT replace, different length | 1 | 0 |
| Evicting PUT | 2 | 0 |

**Generator.** splitmix64 seeded with `S XOR i` produces, for key index `i`:
key = the first 16 output bytes (two outputs, little-endian); value length =
100 + (next output mod 201); value bytes = the following outputs,
little-endian, truncated to the length. Prefill inserts keys `0..N−1`.

**Workload.** Uniform source: splitmix64; a double is `u = (x >> 11) ×
2^-53`. Each k6 iteration draws once for the operation (`u < 0.9` → GET,
otherwise PUT) and once for the key: a Zipf rank `r` in `1..K` with exponent
1.1, K = floor(1.25 × E), sampled by rejection inversion
(Hörmann–Derflinger), mapped to key index `r − 1`. A PUT sends `value(i)`. A
GET that misses is followed in the same iteration by a cache-aside PUT of
`value(i)` with header `X-Op: fill`. k6 tags every request with `op` = `get`,
`put` or `fill`; latency percentiles use `get` and `put` only, and the server
excludes `X-Op: fill` requests from its histograms. k6 computes splitmix64
with BigInt and seeds each VU with `WORKLOAD_SEED XOR (vu << 32) XOR
iteration`.

**Golden data.** `experiments/cache/testdata/golden.json` holds keys and
values for `i = 0..99` with seed 42, the first 1,000 Zipf ranks for
`WORKLOAD_SEED = 1`, the expected `large`-pass resident entry count, and the
expected replay hit count (replay: seed 7, 200,000 iterations, run serially).
Go tests, Rust tests, `hit_ratio.py` and a k6 check must all reproduce it.

**Latency stamping** (both languages, in FAIRNESS). Only `/v1/entries/*`
requests are recorded. Start, three cases: (a) first request on a connection:
the first read after accept that returns bytes; (b) later request not yet
buffered: the first read that returns bytes after the previous response's last
byte was written; (c) later request already buffered (pipelined): the moment
the previous response's last byte was written. End: the moment the last
response byte is written to the connection.

*   **Go:** a wrapped `net.Listener` returns wrapped `net.Conn`s; the
    per-connection stamp reaches the handler through `http.Server.ConnContext`.
    The handler sets `Content-Length`, writes once, calls
    `http.NewResponseController(w).Flush()`, and records the end stamp after
    `Flush` returns.
*   **Rust:** the `TcpStream` is wrapped in a type implementing `AsyncRead` and
    `AsyncWrite` under `hyper_util::rt::TokioIo`, sharing per-connection state
    with the service. `http1::Builder::pipeline_flush(false)` is set
    explicitly. The service pushes each response's byte count into a
    per-connection FIFO; `poll_write` stamps the end when the running byte
    count crosses a response boundary.

**Request bodies.** Go reads with `io.ReadFull` into a `[300]byte` from a
`sync.Pool`; Rust collects the body and copies it into a fixed `[u8; 300]`.
Responses are written once, with an explicit `Content-Length` (Rust uses
`Full<Bytes>`).

**Histograms.** Each variant keeps two preallocated HdrHistograms (values in
nanoseconds, 1 ns to 10 s, 3 significant digits), each protected by a mutex,
and swaps them once per second. A sample goes into the slot of the second
containing its end time. The closed histogram is encoded (HdrHistogram V2
compressed, base64, a few KB), stored, then reset for reuse; at most 1,200
encoded slots are kept, oldest dropped first. Same scheme in both languages,
written into FAIRNESS. Libraries: Go `github.com/HdrHistogram/hdrhistogram-go`,
Rust `hdrhistogram`, Python `hdrh`.

**Bucket layout.** `campaigns/<id>/schedule.json`,
`campaigns/<id>/blocks/<n>/block_verdict.json`,
`campaigns/<id>/campaign-done`, `calibration/<entries>.json`,
`runs/<env>/<run>/`, `raw/<run>/perf.data` (public-read), `reaper/last_run`.
`<env>` is `local` or `do`; `<run>` is `<YYYYMMDDTHHMMSSZ>-<sha>`.

### Target Files to Create

#### Repository root (C2)

1.  **`README.md`**. *Contents:* what the project is, the five variants, how
    to run locally and in the cloud, where results and posts live, link to the
    methodology. *Why:* entry point for readers who follow a link from the
    post.
2.  **`Makefile`**. *Contents:*
    *   `build`: Go, Rust, campaign, sampler and node-facts images.
    *   `build-profiling`: the two Datadog profiling images and the Go `diag`
        image used by the GC diagnostic campaign.
    *   `test`, `conformance VARIANT=…`, `conformance-stub`, `persistent-up`,
        `analyze`, `charts`: commands run inline.
    *   `local-run VARIANT=… ENTRIES=… LEVEL=…`: calls `local-run.sh`.
    *   `session-up ENV=… CAMPAIGN=…`, `collect`, `extend-ttl HOURS=…`,
        `session-down`: call scripts in `harness/scripts/`.
    *   `status`: prints the active cluster, its age and TTL, and the
        campaign's progress read from the bucket.

    *Why:* one command per user journey (CUJ 1).
3.  **`.gitignore`** and **`.env.example`**. *Contents:* ignore `.env`,
    Terraform state, build outputs and caches; `.env.example` names
    `DIGITALOCEAN_TOKEN`, `SPACES_ACCESS_KEY_ID`, `SPACES_SECRET_ACCESS_KEY`
    (broad key, used only by Terraform on the laptop), `DD_API_KEY`,
    `DD_SITE`, `REGISTRY`, `OBJECT_STORE_BUCKET`, `REMAINING_CREDIT_USD`, with
    no values. *Why:* keep secrets and local state out of git.
4.  **`go.work`**. *Contents:* includes `experiments/cache/go` and
    `experiments/cache/conformance`; module paths use the placeholder
    `github.com/<owner>/memory-management-study/...`, replaced with the final GitHub owner
    before C3. *Why:* `go build ./...` works from the root.
5.  **`.github/workflows/ci.yml`**. *Contents:* on push, build all images, run
    Go and Rust unit tests (including Miri for the pointer layout), linters,
    golden-vector tests in every language, the `small` and `large`
    conformance passes against all five variants and the stub, Python and bats
    tests, `terraform fmt -check` and `terraform validate`. *Why:* a variant
    that breaks the contract fails before any cloud run.
6.  **`.github/workflows/reaper.yml`**. *Contents:* every 15 minutes, lists
    clusters tagged `mms-session` through the provider API and destroys
    any whose age exceeds the hours in its `ttl-hours:<n>` tag, or whose
    `campaign:<id>` tag points to a campaign with `campaigns/<id>/campaign-done`
    in the bucket; deletes volumes and load balancers carrying the same
    session tag; never touches the persistent stack; writes `reaper/last_run`;
    logs what it destroyed. *Why:* cost safety that does not depend on the
    laptop.

#### Rules and definitions (C1–C2, written before any service code)

7.  **`docs/METHODOLOGY.md`**. *Contents:* the question, the ladder and layout
    pairs, environments, links to the rule files, how to read the charts.
    *Why:* the single page every post links to.
8.  **`docs/FAIRNESS.md`**. *Contents:* the shared **rules version** (v0 at
    C2, frozen as v1 at the C7 smoke test, bumped on every later change) and a
    dated changelog. Clauses:
    *   **Toolchains and build flags:** pinned Go and Rust versions; no PGO in
        either language; Rust profile `lto = "fat"`, `codegen-units = 1`,
        `opt-level = 3`, `debug = "line-tables-only"`, `panic = "abort"`;
        frame pointers and debug info kept in both binaries; Rust target
        `x86_64-unknown-linux-gnu` with `-C force-frame-pointers=yes` and
        `-C target-feature=+aes`; `-fno-omit-frame-pointer` for the mimalloc C
        build.
    *   **Code shape:** allocator (mimalloc, D4); HTTP stacks (D3);
        hand-written LRU (D2); the contract's cache operations, layouts,
        allocation counts, request-body handling and dynamic dispatch.
    *   **Cache parameters:** logical charge; 64 shards; standard mutexes; map
        hashing (Go built-in, Rust `ahash`); pre-sizing.
    *   **Instrumentation:** the contract's latency stamping, histogram scheme
        and admin endpoints, identical in every variant.
    *   **`unsafe`:** allowed only in `lru_pointer.rs` (including `unsafe impl
        Send` for the node pointer type), each block with a safety comment and
        Miri-tested; one named exemption for the mimalloc process-info call in
        the binary-only module `src/bin_metrics.rs`.
    *   **Threads:** `GOMAXPROCS=2` and `TOKIO_WORKER_THREADS=2`.
    *   **Node:** transparent huge pages `madvise` for both `enabled` and
        `defrag`, chosen because neither runtime is then forced into huge
        pages it did not request.
    *   **Factors:** pod limit = peak working set × 1.3; density memory need =
        peak working set × 1.15; density CPU need = CPU per 1,000 iterations ×
        rate × 1.2; OOM resize: pod limit × 1.2; `go-tuned` uses `go-default`'s
        pod limit.
    *   **Latency target** (D10), added as a dated v0 amendment at C6.
    *   **k6 VUs:** `preAllocatedVUs = ceil(rate × latency target in seconds ×
        4)`, `maxVUs = 2 ×` that.
    *   **AI process:** one prompt template per language, at most 3 fix
        iterations per component, prompts committed as
        `docs/prompts/<lang>-<component>.md`.
    *   **Tuning:** `go-tuned` and `go-pointerfree` each sweep GOGC over 100,
        200 and 400 and GOMEMLIMIT at 80% and 90% of their own pod limit (6
        settings). Each sweep runs at 5M entries and the 70% level, in 3
        shuffled blocks, with 6-minute windows. The winner (D12) is reused for
        every entry count. Rust uses `MIMALLOC_*` defaults with no sweep, an
        accepted asymmetry stated in the post.

    *Why:* a sceptical reader must see the rules were fixed before the
    results.
9.  **`docs/METRICS.md`**. *Contents:* numbered definitions with formula and
    source:
    *   p99 and p99.9: client side from k6's end-of-run summary (scenario
        `measure`, tags `op:get` and `op:put`); server side from the encoded
        per-second histograms merged over the window.
    *   The frozen frame list mapping each frame to exactly one category
        (memory management, lock wait, other), with a one-line reason per
        frame; precedence lock wait > memory management > other; the
        truncated-stack share; Go `runtime/metrics` GC CPU as a cross-check;
        the CPU sample count (reported, not a validity rule).
    *   Working set (cgroup v2 `memory.peak` for sizing, cAdvisor for series);
        bytes per entry including slot slack; CPU seconds per 1,000
        iterations; calculated pods per node; image size per language; page
        faults.
    *   `map_growth_since_prefill`: Go counts insertions that take a shard's
        `len(map)` above its pre-size; Rust compares `HashMap::capacity()`
        with its value after prefill. The two measures differ and must both be
        zero.
    *   The latency target, the slowdown factor and the overload categories
        (held, degraded, overloaded, killed).
    *   The GC alignment measure (Goal 1).
    *   The `/metrics` names each variant must export.
    *   The applicability matrix (Primary = a headline result in the post;
        Secondary = supporting or explanatory; N/A with reason; GC rows are
        N/A for Rust, not zero) and the fixed list of comparisons, each with
        its baseline variant.
    *   The rule for a real difference: the Student's t 95% interval of the
        mean paired within-block difference (variant minus the comparison's
        baseline variant, at least 5 blocks) excludes zero, with a descriptive
        sign count. Cross-node results (3 blocks) get no interval; they
        confirm a result only if the mean has the same sign as on the primary
        node and is within a factor of 2 of it (D9).

    *Why:* every published number has one written definition.
10. **`docs/RUN_PROTOCOL.md`**. *Contents:*
    *   Calibration (owned by `campaign.py`): a fresh pod per ramp with a
        60-second warm-up; 10-second p99s computed exactly from k6's raw CSV
        samples.
    *   The session procedure: sizing pre-run, shuffled blocks, a fresh
        Deployment per run. The block is the pairing unit; long sweeps span
        several sessions, and each block records its node.
    *   Load levels and the overload rule; warm-up 3 minutes; the reset at 3
        minutes; the measurement scenario starting at 3 minutes 2 seconds;
        measurement window at least 10 minutes (tuning sweeps: 6 minutes); at
        least 10^5 requests per run.
    *   Invalid runs: rerun once at the end of the block; a block with a
        variant still invalid is repeated once; if still invalid, the block is
        dropped, the configuration flagged as failed, and the campaign
        continues.
    *   OOM: at 110%, outcome "killed"; at 25% or 70%, one resize (× 1.2) and
        rerun.
    *   The campaign shape: tuning sweeps, a load-level sweep at 5M entries,
        an entry-count sweep at one fixed absolute rate, and the cross-node
        check; the budget and the minimum publishable campaign.

    *Why:* runs are comparable only if they follow one procedure.
11. **`docs/PREDICTIONS.md`** (frozen at C2). *Contents:* for each layout pair,
    load level and entry count, the expected direction, size bucket (under
    10%, 10–50%, over 50%) and mechanism. Prediction columns are never edited;
    only an Outcome column is appended after C7. Includes the rung-by-stage
    table from [`docs/research/gc-phases.md`](../research/gc-phases.md)
    (change, stage affected, mechanism, metric, expected direction), stating
    that it assumes the Green Tea collector. *Why:* records the expected
    mechanisms before any result, so they can be checked against outcomes.
12. **`docs/AI_LOG.md`** and **`docs/prompts/<lang>-<component>.md`**.
    *Contents:* dated mistakes found in AI-generated code and how they were
    caught; the prompt templates and prompts used. *Why:* evidence of judgement
    and a FAIRNESS audit trail.
13. **`docs/designs/cache-experiment.md`** and **`docs/designs/SITEMAP.md`**.
    *Contents:* this design and the design index. *Why:* the project's
    design-document folder.
14. **`experiments/cache/claims.yaml`** (C1). *Contents:* one entry per external
    claim (source URL, quoted claim, conditions, verification status, our
    reproduction after C7), plus the C1 checklist results: Datadog plan and
    profilers, Go forced-GC behaviour and collector, dedicated-CPU nodes,
    static CPU manager, THP control, bucket-scoped keys, `perf` on the SUT
    node, cAdvisor refresh interval, frame pointers in the Rust standard
    library. *Why:* no figures quoted from memory.

#### Cache specification and conformance (C3)

15. **`experiments/cache/SPEC.md`**. *Contents:* the cross-language contract in
    full, with byte layouts and JSON schemas; resident entry counts and
    target hit ratios for 1M, 5M, 10M and the `large` pass, from
    `hit_ratio.py`; environment variables (`CACHE_MAX_BYTES`, `CACHE_SHARDS`,
    `CACHE_LAYOUT`, `PORT`, `PREFILL_SEED`, `GOMAXPROCS`,
    `TOKIO_WORKER_THREADS`). *Why:* one spec, five implementations.
16. **`experiments/cache/testdata/golden.json`**. *Contents:* as in the
    contract. *Why:* every implementation of the generator and workload is
    checked against it.
17. **`experiments/cache/sim/hit_ratio.py`**, **`sim/test_hit_ratio.py`** and
    **`sim/pyproject.toml`**. *Contents:* replays the seeded workload against a
    reference LRU with the same sharding, per-shard bounds and logical charge;
    prints the resident entry count, hit ratio and eviction rate for each
    configured entry count and the `large` pass, and the replay hit count; the
    test checks `golden.json`; pinned `pytest`. *Why:* the targets the
    validity checks compare against.
18. **`experiments/cache/conformance/`** (Go module: `go.mod`, `go.sum`,
    `main.go`, `cases.go`, `README.md`) and **`conformance/stub/`** (an
    in-memory reference implementation of the contract). *Contents:* a
    black-box CLI taking a base URL and a pass name.
    *   Pass `small` (variant started with `CACHE_MAX_BYTES = 64 × 380`): get/put
        round trip; 404, 405 and 400 cases; replace semantics; after writing
        1,000 keys forced into one shard, every key still resident returns its
        own value, the evicted keys are exactly the least recently used, and
        the shard never holds more than floor(380 / 180) entries.
    *   Pass `large` (`CACHE_MAX_BYTES = 100,000 × 280`, prefill 100,000):
        resident entry count matches `golden.json`; 100 sampled values match
        the generator; the replay yields the exact expected hit count; reset
        clears the right counters; histograms fill; a pipelining client
        sending 3 requests per write with 50 ms delays gets one latency sample
        per request with start ≤ end; `/readyz` only after prefill; 409 on a
        second prefill.

    Non-zero exit code on any failure. *Why:* behavioural equivalence across
    five implementations; the stub lets C3 finish before C4.

#### Go implementation (C4)

One Go module builds one binary; `CACHE_LAYOUT=pointer` or `slab` selects the
layout at startup. Variants `go-default` and `go-tuned` use `pointer`;
`go-pointerfree` uses `slab`.

19. **`experiments/cache/go/go.mod`** and **`go.sum`**. *Contents:* pinned
    `toolchain` line; dependencies `github.com/prometheus/client_golang`,
    `github.com/HdrHistogram/hdrhistogram-go`, `golang.org/x/sys/cpu` (hash-path
    log line), and `github.com/DataDog/dd-trace-go/v2` (pinned; compiled only
    under `-tags ddprof`, so measurement binaries do not contain it). *Why:*
    reproducible builds.
20. **`experiments/cache/go/cmd/cache/main.go`**. *Contents:* read env vars,
    validate `CACHE_SHARDS` and `CACHE_MAX_BYTES`, pick the layout, pre-size
    maps, start the `net/http` server on the wrapped listener with
    `ConnContext`, log configuration, `hash_path` and wall-clock process start
    time. *Why:* entry point.
21. **`experiments/cache/go/cmd/cache/profiling_ddprof.go`**. *Contents:*
    behind the `ddprof` build tag, starts the dd-trace-go profiler with CPU and
    heap profiles and the service name from env. *Why:* the Go profiling
    image.
22. **`experiments/cache/go/internal/lru/cache.go`**. *Contents:* the `Cache`
    interface from the contract. *Why:* dynamic dispatch, matching Rust.
23. **`experiments/cache/go/internal/lru/pointer.go`** (hot path). *Contents:*
    the pointer layout from the contract: 64 shards, each a `sync.Mutex`, a
    `map[[16]byte]*node` and a doubly linked list; growth tracking. *Why:*
    rungs 1 and 2; the pointer-heavy layout the GC must scan.
24. **`experiments/cache/go/internal/lru/slab.go`** (hot path). *Contents:* the
    slab layout from the contract: 64 shards, each with a
    `map[[16]byte]uint32`, a record array, four size-class slabs and free
    lists. *Why:* rung 3; entry records, map and slabs hold no pointers, so GC
    marking does not visit entries.
25. **`experiments/cache/go/internal/lru/lru_test.go`**. *Contents:* for both
    layouts: eviction order, byte bound, same-shard keys, replace semantics,
    slot and record reuse, concurrent get/put under the race detector, a churn
    test (10 × capacity of evicting PUTs) asserting no map growth, and
    `testing.AllocsPerRun` for every row of the allocation table. *Why:* catch
    bugs and prove the allocation counts.
26. **`experiments/cache/go/internal/prefill/prefill.go`** and
    **`prefill_test.go`**. *Contents:* the contract's generator; the test checks
    `golden.json`. *Why:* identical data across languages.
27. **`experiments/cache/go/internal/server/server.go`**,
    **`server_test.go`** (hot path) and **`diag_trace.go`** (built only with
    the `diag` tag). *Contents:* the wrapped listener and
    connection, the handlers, body handling and stamping from the contract;
    tests cover routing, status codes, stamping and fill exclusion. *Why:*
    contract and latency measurement.
28. **`experiments/cache/go/internal/metrics/`** (`metrics.go`,
    `histograms.go`, `histograms_test.go`). *Contents:* the `runtime/metrics`
    names listed in `METRICS.md` (including the GC CPU classes, scan, heap,
    cycle, limiter, pause and scheduling-latency metrics), a startup log line
    with paired monotonic and wall-clock readings, cache stats including per-shard lock
    acquisitions, and the histogram scheme from the contract; tests cover
    rotation, encoding and the 1,200-slot cap. *Why:* METRICS.md sources.
29. **`experiments/cache/go/cmd/smoke/main.go`**. *Contents:* three loops run
    for 60 seconds each: allocation-heavy, contended mutex, and allocate-and-
    discard. *Why:* checks that the sampler's categories attribute correctly
    in Go.
30. **`experiments/cache/go/Dockerfile`**, **`Dockerfile.profiling`**,
    **`Dockerfile.diag`** and **`.golangci.yml`**. *Contents:* multi-stage build with the pinned Go image,
    `CGO_ENABLED=0`, `-trimpath`, onto `gcr.io/distroless/static`; the
    profiling variant adds `-tags ddprof`; a `Dockerfile.diag` variant adds
    `-tags diag`, which enables the flight recorder and a `trace.WithRegion`
    per request through `internal/server/diag_trace.go`; linters `govet`, `staticcheck`,
    `errcheck`, `ineffassign`, `unused`. *Why:* same multi-stage distroless
    pattern as Rust; CI.

#### Rust implementation (C4)

One Rust binary; `CACHE_LAYOUT=pointer` or `slab` selects the layout at
startup. Variant `rust-pointer` uses `pointer`; `rust-index` uses `slab`.

31. **`experiments/cache/rust/Cargo.toml`**, **`Cargo.lock`**,
    **`rust-toolchain.toml`** and **`.cargo/config.toml`**. *Contents:*
    dependencies `hyper` 1.x, `hyper-util`, `http-body-util`, `tokio` 1.x
    (multi-thread runtime), `mimalloc`, `libmimalloc-sys` (feature
    `extended`, used only by the binary), `ahash`, `hdrhistogram`,
    `prometheus-client`, and `stats_alloc` (dev); the release profile from
    FAIRNESS. `.cargo/config.toml` puts `rustflags = ["-C",
    "force-frame-pointers=yes", "-C", "target-feature=+aes"]` under
    `[target.x86_64-unknown-linux-gnu]` and `CFLAGS =
    "-fno-omit-frame-pointer"` under `[env]`; the `RUSTFLAGS` environment
    variable is never set, because it would override this file. Pinned stable
    toolchain and a pinned `nightly-YYYY-MM-DD` for Miri. *Why:* D3, D4,
    reproducible builds.
32. **`experiments/cache/rust/src/lib.rs`**. *Contents:* declares `lru`,
    `lru_pointer`, `lru_slab`, `server`, `prefill`, `metrics`, `fnv`,
    `alloc_setup`. The library contains no mimalloc FFI. *Why:* tests and Miri
    link against the library without loading C mimalloc.
33. **`experiments/cache/rust/src/alloc_setup.rs`**. *Contents:* a macro that
    declares mimalloc as `#[global_allocator]`, used by both binaries. *Why:*
    the smoke binary uses the same allocator as the service.
34. **`experiments/cache/rust/src/main.rs`** and **`src/bin_metrics.rs`**.
    *Contents:* thin binary: invokes the allocator macro, reads env vars,
    validates `CACHE_SHARDS` and `CACHE_MAX_BYTES`, builds the tokio runtime
    with `TOKIO_WORKER_THREADS` workers, starts the server, logs configuration,
    `hash_path` and process start time; `bin_metrics.rs` holds the mimalloc
    process-info FFI call (the named exemption). *Why:* entry point.
35. **`experiments/cache/rust/src/lru.rs`**. *Contents:* the cache trait from
    the contract, used through `dyn` dispatch. *Why:* matches the Go
    interface.
36. **`experiments/cache/rust/src/lru_pointer.rs`** (hot path). *Contents:* the
    pointer layout: 64 shards, each a `std::sync::Mutex` around a
    `HashMap<[u8; 16], NonNull<Node>, ahash::RandomState>` and a doubly linked
    list of separately allocated nodes with each value in its own `Box<[u8]>`;
    capacity tracking; all `unsafe` confined here, each block with its safety
    argument. *Why:* rung 4, twin of Go rungs 1–2 (D5).
37. **`experiments/cache/rust/src/lru_slab.rs`** (hot path). *Contents:* the
    slab layout: 64 shards, each with a `HashMap<[u8; 16], u32,
    ahash::RandomState>`, a `Vec` of records, four size-class slabs and free
    lists; safe Rust only. *Why:* rung 5, twin of Go rung 3 (D5).
38. **`experiments/cache/rust/src/server.rs`** (hot path). *Contents:* the
    stream wrapper, response-size FIFO, hyper service with
    `pipeline_flush(false)`, handlers and body handling from the contract.
    *Why:* contract and latency measurement.
39. **`experiments/cache/rust/src/prefill.rs`** and **`src/fnv.rs`**.
    *Contents:* the contract's generator and FNV-1a. *Why:* identical data and
    sharding.
40. **`experiments/cache/rust/src/metrics.rs`**. *Contents:* process RSS, live
    bytes, per-shard lock acquisitions, the `METRICS.md` names, and the
    histogram scheme from the contract. *Why:* METRICS.md sources.
41. **`experiments/cache/rust/tests/`**: `lru.rs` (the Go test cases for both
    layouts, smaller sizes under `cfg(miri)`), `alloc_counts.rs` (every row of
    the allocation table via `stats_alloc`, in its own test binary so other
    tests cannot disturb the process-wide counts), `golden.rs`, `server.rs`
    (routing, status codes, stamping, fill exclusion) and `histograms.rs`
    (rotation, encoding, cap). Miri runs `cargo +nightly-YYYY-MM-DD miri test
    --test lru pointer_`. *Why:* parity with Go, proof of allocation counts,
    evidence the `unsafe` code is sound.
42. **`experiments/cache/rust/src/bin/smoke.rs`**. *Contents:* invokes the
    allocator macro and runs the same three loops as the Go smoke program.
    *Why:* checks sampler attribution in Rust.
43. **`experiments/cache/rust/Dockerfile`** and **`Dockerfile.profiling`**.
    *Contents:* builder `rust:<pinned>-bookworm`, runtime
    `gcr.io/distroless/cc-debian12`, both by digest, so the glibc versions
    match; the profiling variant copies Datadog's `ddprof` binary from its
    pinned release into the image and starts the service through it **(needs
    verification at C1)**. *Why:* glibc rather than musl, to avoid musl
    allocator and `memcpy` differences **(needs verification)**; measurement
    images stay free of Datadog code.
44. **`experiments/cache/HOTPATH_NOTES.md`**. *Contents:* the author's
    line-by-line notes on `pointer.go`, `slab.go`, `server.go`,
    `lru_pointer.rs`, `lru_slab.rs` and `server.rs`, including the safety
    argument for each `unsafe` block and the hot-path line counts. *Why:*
    interview readiness; the code-complexity measure.

#### Variants, load and campaigns (C4–C5)

45. **`experiments/cache/variants/{go-default,go-tuned,go-pointerfree,rust-pointer,rust-index}.yaml`**
    and **`harness/schema/variant.schema.json`**. *Contents:* image (one per
    language), `CACHE_LAYOUT`, `GOMAXPROCS=2` or `TOKIO_WORKER_THREADS=2`,
    `GOGC` and `GOMEMLIMIT` (as a fraction of the pod limit) for the tuned
    variants, and which sizing-table row the requests and limits come from.
    `run.sh` turns the fraction into bytes (floor of fraction × limit) and
    sets `CACHE_MAX_BYTES` from E. The run manifest embeds this file. *Why:* a
    variant is data, not code.
46. **`experiments/cache/TUNING.md`** (after the sweeps). *Contents:* both
    sweeps' results, the chosen values, links to run folders. *Why:* shows the
    tuned variants were tuned by procedure.
47. **`experiments/cache/load/lib/generator.js`**, **`load/steady.js`** and
    **`load/test_generator.js`**. *Contents:* `generator.js` implements the
    contract's splitmix64 (BigInt), Zipf sampler and value generator.
    `steady.js` takes `RATE`, `ENTRIES`, `WORKLOAD_SEED`, `TARGET_URL`,
    `LATENCY_TARGET_MS` and `VUS`; runs one k6 test with a 3-minute warm-up
    scenario, a one-iteration `reset` scenario at 3 minutes, and the `measure`
    scenario from 3 minutes 2 seconds; uses `constant-arrival-rate`; tags `op`;
    sets thresholds on `http_req_duration{scenario:measure,op:get}` and
    `{scenario:measure,op:put}` at the latency target (recorded, not
    aborting); sets `summaryTrendStats` with p(99) and p(99.9); writes the
    summary through `handleSummary`; outputs `experimental-prometheus-rw` with
    `K6_PROMETHEUS_RW_PUSH_INTERVAL=1s`, and `--out csv` for calibration
    runs. `test_generator.js` checks `golden.json`. *Why:* main measurement
    runs at every load level.
48. **`experiments/cache/campaigns/`**: `smoke.yaml`, `calibration.yaml`,
    `tuning-go-tuned.yaml`, `tuning-go-pointerfree.yaml`, `load-level.yaml`,
    `entry-count.yaml`, `sampler-overhead.yaml`, `cross-node.yaml`,
    `datadog-profiling.yaml`, `gc-diagnostics.yaml`, and
    **`harness/schema/campaign.schema.json`**.
    *Contents:* campaign ID, schedule seed, configurations with levels as
    percentages of capacity, the calibrated absolute rates (committed after
    calibration by `make collect`), block counts, sessions of at most 12
    hours, and a header with run count, estimated cluster-hours and cost
    matching the Budget table. *Why:* input to the campaign Job; the plan of
    record for each session.

#### Harness images and cluster (C5)

49. **`harness/images/campaign/Dockerfile`** and
    **`harness/campaign/pyproject.toml`**. *Contents:* pinned Python base,
    `kubectl`, the aws CLI configured for Spaces, `inferno`, `envsubst`,
    `curl`, `jq`, and the pinned Python dependencies (`pyyaml`, `jsonschema`,
    `boto3`, `pytest`). *Why:* the image the campaign Job runs.
50. **`harness/images/sampler/Dockerfile`**. *Contents:* `perf` from the
    distribution matching the DOKS node kernel; if C1 finds no match, the
    sampler runs the host's `perf` through `nsenter` instead (decided and
    recorded at C1). *Why:* the image the sampler DaemonSet runs.
51. **`harness/images/node-facts/Dockerfile`**. *Contents:* `chrony` client,
    `util-linux` (`lscpu`), and a script reading kernel, cgroup version, THP
    mode and host identifiers through host mounts. *Why:* the image the
    node-facts DaemonSet runs.
52. **`harness/local/compose.yaml`** and **`harness/scripts/local-run.sh`**.
    *Contents:* compose runs one variant with `cpus: 2` and `mem_limit`, a k6
    container and Prometheus. `local-run.sh` brings compose up, calibrates
    locally with the same rule as the cloud (using `campaign.py --local`),
    prefills, runs `steady.js`, collects histograms, stats, gctrace output and
    Prometheus exports, and writes a manifest whose cloud-only fields are
    `"n/a (local)"` (allowed by the schema). No sampler runs locally; profile
    checks happen in the cloud smoke test. *Why:* the C5 pipeline check.
53. **`harness/infra/terraform/do/persistent/`** and
    **`harness/infra/terraform/do/session/`** (each with `main.tf`,
    `variables.tf`, `outputs.tf`, `versions.tf`, `.terraform.lock.hcl`).
    *Contents:*
    *   **Persistent stack**, never destroyed by a session: the results bucket
        (with a public-read `raw/` prefix), the container registry, and a
        bucket-scoped storage key created with `digitalocean_spaces_key`
        (scope checked at C1), output for `session-up.sh`.
    *   **Session stack:** the DOKS (DigitalOcean Kubernetes) cluster with
        node pools `sut` (one 4-vCPU / 16 GB dedicated-CPU node), `load`
        (default one 4-vCPU dedicated node) and `support` (default one 2-vCPU
        node), as variables; taints and labels; tags `mms-session`,
        `ttl-hours:<n>` and `campaign:<id>` with `lifecycle { ignore_changes =
        [tags] }`; kubeconfig output.

    Terraform state is local per stack and git-ignored. *Why:* cloud
    environment; replaced by another provider folder if C1 rules out
    DigitalOcean.
54. **`harness/k8s/sut.yaml.tmpl`**. *Contents:* Deployment and Service for one
    variant in namespace `bench`; node selector and toleration for `sut`;
    Guaranteed QoS sized from the sizing table; image by digest; env from the
    variant file; readiness probe on `/readyz`; `GODEBUG=gctrace=1,scavtrace=1`
    for Go (plus `gctrace=2,gcpacertrace=1` in diagnostic runs).
    *Why:* the system under test.
55. **`harness/k8s/sampler-daemonset.yaml`**. *Contents:* privileged
    DaemonSet in `bench` on the `sut` pool with `hostPID`; an init container
    that writes `madvise` to the THP `enabled` and `defrag` files; started and
    stopped by `run.sh` through `kubectl exec`; finds the cache PID by its
    cgroup path; records with `perf record -k CLOCK_MONOTONIC -F 99 -g` for Go
    and `perf record -k CLOCK_MONOTONIC -F 99 --call-graph dwarf` for Rust (in
    Rust diagnostic runs, also counting the `madvise` syscall tracepoint); after stopping, and before the cache
    pod is deleted, runs `perf script` on the node (where the binaries are
    reachable through `/proc/<pid>/root`) and writes the text next to
    `perf.data`. *Why:* D6.
56. **`harness/k8s/node-facts-daemonset.yaml`**. *Contents:* runs the
    node-facts image on the `sut` and `load` pools; records `chronyc
    tracking`, kernel, cgroup version, THP mode, `lscpu` and any host
    identifier. *Why:* manifest fields and NTP offsets.
57. **`harness/k8s/k6-job.yaml.tmpl`** and a **k6 script ConfigMap** (rendered
    from `experiments/cache/load/`). *Contents:* Job in `bench` on the `load`
    pool running `steady.js` with the rate for the current level; the
    ConfigMap carries `steady.js` and `lib/generator.js`. *Why:* load
    generation.
58. **`harness/k8s/campaign-job.yaml.tmpl`**. *Contents:* Job in `bench` on the
    `support` pool running the campaign image; a namespaced Role with `get`,
    `list`, `create`, `delete` on Deployments, Services, Jobs, Pods and
    ConfigMaps, `get` and `create` on `pods/exec`, `get` on `pods/log`, and
    `get`, `list` on events; a ClusterRole with `get`, `list` on nodes; a
    Secret holding the bucket-scoped storage key (no cloud token). *Why:*
    campaigns survive the laptop going to sleep.
59. **`harness/k8s/prometheus/values.yaml`**. *Contents:* values for the
    pinned `prometheus-community/prometheus` chart on the `support` pool;
    `server.persistentVolume.enabled: false` (so no untagged volume is
    created); scrape SUT `/metrics` and kubelet/cAdvisor every 2 seconds;
    `--web.enable-remote-write-receiver` for k6. *Why:* D1.
60. **`harness/k8s/datadog/values.yaml`**. *Contents:* values for the pinned
    `datadog/datadog` chart, used only by the profiling campaign: cluster
    agent on `support`, node agent on `sut`. *Why:* Datadog is not installed
    in measurement sessions.

#### Harness scripts (C5)

61. **`harness/scripts/session-up.sh`**. *Contents:* refuse to start if a
    cluster already exists, if `reaper/last_run` is older than 24 hours, or if
    the campaign's estimate exceeds `REMAINING_CREDIT_USD`; if the provider
    reports the state's cluster missing (deleted by the reaper), run
    `terraform state rm` for it; Terraform apply for the session stack; set
    the TTL tag from the campaign estimate plus 2 hours and the `campaign`
    tag; create namespace `bench`; Helm installs; write the storage-key
    Secret from the persistent stack's output; verify node types; start the
    campaign Job. *Why:* cost control and setup.
62. **`harness/scripts/session-down.sh`** and **`extend-ttl.sh`**. *Contents:*
    `session-down.sh` destroys the session stack; `extend-ttl.sh` rewrites the
    `ttl-hours` tag through the provider API. *Why:* manual control.
63. **`harness/campaign/campaign.py`** and **`campaign/tests/test_campaign.py`**.
    *Contents:* runs inside the campaign Job (or locally with `--local`).
    *   Calibration campaign: for each step, runs a short k6 test at the given
        rate with `--out csv` on a fresh pod after a 60-second warm-up,
        computes exact 10-second p99s from the raw samples, passes the step if
        the worst is within 4 × the low-load value and no iterations drop; runs
        each variant's 150-second check; writes `calibration/<entries>.json`.
    *   Other campaigns: read the calibrated rates; generate the shuffled
        schedule once from the campaign's seed and store
        `campaigns/<id>/schedule.json`; run sizing pre-runs; call `run.sh` for
        each run; after each block, run the block-level hit-ratio check and
        write `block_verdict.json`; apply the invalid-run and OOM rules; resume
        against the stored schedule; warn if k6 CPU exceeds 60% (an early
        warning below the 70% validity limit; the next session's load node is
        added by hand); write `campaign-done` at the end.

    Tests cover schedule determinism, resume, calibration step decisions, and
    the invalid and OOM rules. *Why:* the procedure from `RUN_PROTOCOL.md`.
64. **`harness/scripts/run.sh`**. *Contents:* one run: render templates with
    `envsubst`; deploy; trigger prefill on the pod IP; poll `/readyz`; start
    the k6 Job; poll `/admin/stats` until the reset is observed, then start
    perf; stop perf at the end of the measurement window; run `perf script`
    through the sampler; call `collect-run.sh` and `validate.sh`; upload the
    run folder (valid or not) to `runs/<env>/<run>/`; delete the Deployment and
    Service. Records the server, client and perf window bounds in the
    manifest. *Why:* the per-run steps.
65. **`harness/scripts/collect-run.sh`**. *Contents:* inside the cluster,
    after each run: fetch `/admin/histograms` and `/admin/stats`; export
    Prometheus range queries and the k6 summary; capture `kubectl logs
    --timestamps`; fetch the `perf script` text, fold it with `inferno`, and
    record page faults; upload `perf.data` to `raw/<run>/`; save `kubectl
    describe` and node facts; record checksums. No other script folds
    profiles. *Why:* everything a run needs lands in its folder.
66. **`harness/scripts/collect.sh`**. *Contents:* on the laptop, pulls run
    folders from `runs/` into `results/` and writes calibrated rates from
    `calibration/` into the campaign files. *Why:* `make collect`.
67. **`harness/scripts/validate.sh`** and **`scripts/tests/validate.bats`**.
    *Contents:* per-run checks (dropped iterations, k6 node CPU from the
    cAdvisor root cgroup `id="/"`, missed scrapes, restarts, throttling,
    `map_growth_since_prefill` = 0, hit ratio against the `SPEC.md` target,
    `/gc/limiter/last-enabled` unchanged during the window for the tuned Go
    variants at the 25% and 70% levels (at the 110% level a limiter
    engagement is recorded in the overload outcome instead of invalidating
    the run), and `/sched/pauses/total/other` under the threshold frozen in
    `METRICS.md` at C2);
    writes the verdict and overload outcome into the manifest; the
    block-level check belongs to `campaign.py`. Tests cover each check. *Why:*
    invalid runs are kept but never charted.
68. **`harness/schema/manifest.schema.json`**. *Contents:* required fields
    (each may be `"n/a (local)"` where the field is cloud-only): campaign ID,
    session ID, run key, schedule seed and block size, block, position and
    node, git SHA, rules version, image digests, toolchains, variant file, env
    vars (including `GOMAXPROCS`, `TOKIO_WORKER_THREADS` and `MIMALLOC_*`),
    sizing values and OOM resize history, load level and absolute rate, VUs,
    latency target, configured entry count, node ID and type, kernel, cgroup
    version, THP mode, `lscpu` topology, pod cpuset, startup working set, NTP
    offsets, `hash_path`, k6 script hash, Datadog agent mode, sampler on/off,
    the server, client and perf window bounds, hit ratio, missed scrapes, CFS
    throttled periods, CPU sample count, page faults, overload outcome,
    checksums, validity. *Why:* every run is self-describing.

#### Analysis (C5, used from C6)

69. **`harness/analysis/pyproject.toml`**. *Contents:* pinned `polars`,
    `matplotlib`, `hdrh`, `scipy`, `pyyaml`, `jsonschema`, `pytest`. *Why:*
    reproducible analysis.
70. **`harness/analysis/analyze.py`**. *Contents:* reads run folders, validates
    manifests, merges histograms into run percentiles, writes each run's
    `derived.json`, and writes the campaign-level
    `results/cache/<env>/summary.json` with paired within-block differences
    (Student's t intervals and sign counts), the cross-node test from D9, and
    the GC alignment measure (by calling `gc_align.py`); refuses to mix
    environments or rules versions. *Why:* one shared script for every number.
71. **`harness/analysis/cpu_share.py`**. *Contents:* applies the frame list from
    `METRICS.md` to folded stacks; outputs memory-management and lock-wait
    shares, CPU sample count, truncated-stack share and page faults per run.
    *Why:* D6.
72. **`harness/analysis/gc_align.py`**. *Contents:* converts gctrace offsets to
    wall-clock time using the logged process start time (cross-checked against
    `kubectl logs --timestamps`); computes the Goal 1 measure; overlays GC
    cycles on per-second latency. *Why:* Goal 1.
73. **`harness/analysis/charts.py`**. *Contents:* the post's charts from
    `summary.json` only: latency by load level, latency by entry count, CPU
    share, bytes per entry, calculated pods per node, and overload outcomes.
    *Why:* charts are reproducible.
74. **`harness/analysis/gc_phases.py`**. *Contents:* parses gctrace and
    scavtrace into a per-cycle table (start, the three clock durations,
    assist, background and idle CPU, heap at start, end and live, goal,
    stacks, globals, inferred trigger kind), failing on an unknown gctrace
    format; joins it with per-second histograms, Prometheus exports and
    timestamped `perf` samples; produces the charts in "GC stage analysis";
    reads the diagnostic campaign's traces for assist time per request and
    objects scanned per cycle. *Why:* Goal 9 and D13.
75. **`harness/analysis/tests/`**: `test_analyze.py` (a mixed-environment and a
    mixed-version case that must fail), `test_cpu_share.py` (a
    frame-attribution case per language), `test_gc_align.py` (a synthetic run
    with known alignment), `test_gc_phases.py` (fixed gctrace lines from the
    pinned Go version, an unknown-format line that must fail, and the
    periodic-cycle inference). *Why:* analysis bugs would corrupt every result.

#### Results and posts (C6–C11)

76. **`results/README.md`**. *Contents:* folder layout, where large files live,
    how to rerun a run from its manifest. *Why:* CUJ 2.
77. **`results/cache/<env>/<run>/`** (created by runs) and
    **`results/cache/<env>/summary.json`** (created by `analyze.py`).
    *Contents:* `manifest.yaml`, `histograms/`, `profiles/`, `metrics/`,
    `logs/`, `derived.json`; the campaign summary. *Why:* source of truth.
78. **`posts/methodology.md`** (C10), **`posts/big-cache.md`** (C11) and
    **`posts/img/`** (authored output). *Contents:* the posts, Datadog
    screenshots as static images, a predictions-versus-outcomes table, and a
    "where the gap is smaller than the story claimed" section. *Why:* the
    deliverable.

### Verification Plan

*   **Build:** `go build ./...` and `go vet ./...` from the root (via
    `go.work`); `cargo build --release`; `make build` produces the Go, Rust,
    campaign, sampler and node-facts images; `make build-profiling` the two
    profiling images.
*   **Unit tests:** `go test -race ./...`; `cargo test`; the pinned-nightly
    Miri command from the Rust tests item; `pytest` in `harness/analysis`,
    `harness/campaign` and `experiments/cache/sim`; `bats
    harness/scripts/tests`; golden-vector tests in Go, Rust, Python and k6;
    every row of the allocation table in both languages.
*   **Static analysis:** `golangci-lint run`; `cargo clippy -- -D warnings`;
    `cargo fmt --check`; `terraform fmt -check` and `terraform validate`.
*   **Conformance:** `make conformance VARIANT=<each>` passes `small` and
    `large` for all five variants, and `make conformance-stub` for the stub,
    with identical hit counts. Required in CI.
*   **Pipeline (local, C5):** `make local-run VARIANT=go-default ENTRIES=1000000
    LEVEL=70` produces a run folder whose manifest validates against the
    schema, with histograms and gctrace, and `make charts` renders from it;
    k6 CPU per iteration is measured, to confirm the BigInt generator leaves
    headroom under the 70% limit.
*   **Mechanism (C6):** the alignment measure and chart are computed for local
    `go-default` runs, regardless of result.
*   **Cloud smoke test (start of C7, 5-minute runs):** `perf` works on the SUT
    node; the three smoke loops attribute more than 90% of samples to the
    expected category in each language; folded profiles exist for every
    variant; one short run per language passes all validity checks and
    appears in the bucket; the reaper destroys a test cluster past its TTL and
    the bucket survives. Rules v1 freezes here.
*   **Overhead checks:** sampler on versus off for one variant per language
    (C7), recorded in `TUNING.md` and the methodology post; Datadog profiler
    on versus off once per language (C9), recorded in the methodology post.

--------------------------------------------------------------------------------

<!-- Provenance: Indicates initial structural drafting via EGM. -->

*Drafted with EGM*
