# Concept Brief: Rebuilding Go-to-Rust Migration Conditions as Measured Experiments

**Author:** Radesh Govind
**Date:** 2026-11-23

---

## Overview

This brief covers a portfolio project, not a graded deliverable. The goal is to show what Rust changes compared with a modern, tuned Go service, using measurements and explained mechanisms instead of one-line claims. The output is a short blog series the author can defend in interviews with companies doing Rust migrations. There is no fixed schedule. Work is organised as ordered checkpoints, each with a done-condition, and the author paces them.

**Author decision (2026-11-23):** cache first, proxy behind a checkpoint gate. The big cache is the one main experiment. The packed-node work becomes a campaign over the cache images. The proxy starts only after the cache post is published (see Key Decisions).

The anchored hypothesis is three separate experiments, each built in Go and in Rust:

1. **The big cache.** An in-memory LRU with millions of entries, modelled on Discord's Read States service, to expose Go GC scan cost and forced periodic GC as tail-latency spikes.
2. **The proxy.** A reverse proxy (the system under test) in front of a trivial backend, modelled on Pingora and Linkerd, measuring CPU and memory per connection and tail latency at high connection counts.
3. **The packed node.** Pod density under tight memory limits: replicas per node at a fixed latency target, behaviour near the memory limit, startup time and image size.

All three would be AI-written under fairness rules, run on Kubernetes (kind/k3d locally, then a cloud provider such as DigitalOcean), loaded with k6 from separate nodes, observed in Datadog, and reported on seven metrics against a tuned Go baseline.

Seven reviewer lenses examined the concept in BRAINSTORM mode. All seven completed.

| Reviewer | Status |
| --- | --- |
| `egm-design` | completed |
| `egm-codebase-details` | completed |
| `egm-zoom-out` | completed |
| `egm-comprehension` | completed |
| `egm-implementation-readiness` | completed |
| `egm-structural` | completed |
| `egm-inventive-catalyst` | completed |

The reviewers had no web access. Every external fact in this brief is marked **(needs verification)**.

---

## Analysis

### Consensus across lenses

The reviewers agree on the core idea: reproducing the conditions behind a migration story and explaining the mechanism is a sound portfolio piece. They do not agree that it works as currently scoped. Six of seven lenses raised scope independently, and five raised the same three structural problems: experiment 3 has no workload of its own, the seven-metric grid does not apply evenly, and local cluster numbers cannot be published.

### Findings by severity

#### Blockers

**B1. Three experiments are too wide for the depth the goal needs.** (zoom-out, design, inventive-catalyst, structural, implementation-readiness, codebase-details)
- 3 experiments x 2 languages x 7 metrics x 2 environments is 6 services and about 42 to 84 result cells.
- With AI-written code, writing the code is cheap. The expensive parts are checking fairness and understanding the hot paths well enough to explain them. Spread across three experiments, all three come out shallow, which defeats the career goal.

**B2. Experiment 3 has no system under test.** (structural, design, implementation-readiness, zoom-out, inventive-catalyst)
- Density, memory-limit behaviour, startup time and image size are ways to measure a deployment, not a workload. As written, experiment 3 either reuses another experiment's binaries without saying so, or needs a third Go/Rust pair.
- Consensus recommendation: run experiment 3 as a measurement campaign over experiment 1's cache images (smaller per-pod working set), with a declared dependency on those image digests.

**B3. The Go baseline needs a ladder, not a single point.** (codebase-details, inventive-catalyst, design, structural, zoom-out)
- Modern Go has changed a lot since the Discord post: pacer redesign (1.18), GOMEMLIMIT (1.19), Swiss-table maps (1.24), and the "Green Tea" GC, experimental in 1.25 with its default status by late 2026 unknown **(all needs verification)**.
- The standard Go answer to a pointer-heavy cache is a pointer-free layout (bigcache/freecache style, byte slabs with integer indexes), which the GC does not scan.
- Proposed ladder per experiment: Go idiomatic with default settings, then Go with GOGC/GOMEMLIMIT tuned, then Go with a GC-friendly layout, then Rust. The post reports how much of the gap each rung closes and at what code-complexity cost. A Go-literate interviewer will dismiss a result that skips the third rung.
- Decide up front that "tuned Go matches Rust" is a publishable result.

**B4. The headline GC metric for experiment 1 is likely the wrong one.** (codebase-details)
- Go's stop-the-world pauses are typically well under a millisecond even on large heaps **(needs verification)**. The Discord-style spikes come from concurrent mark CPU and mutator mark assists while scanning millions of pointers. A post headlined "GC pause time" may show Go looking fine and give the wrong explanation.
- Measure GC CPU share, mark-assist time, and the STW pause histogram separately, and align p99.9 against GC cycle timestamps. Use `GODEBUG=gctrace=1` and `runtime/metrics` as ground truth.
- Forced periodic GC (`forcegcperiod`, about 2 minutes) only appears when the allocation rate is too low to trigger GC first, so the load profile must reproduce a large live heap with little garbage. Whether it still exists in current Go, and whether `GOGC=off` disables it, **needs verification**.

**B5. Experiment 2's migration framing may be factually wrong.** (codebase-details, inventive-catalyst)
- Reviewers believe Pingora replaced NGINX (C/Lua) at Cloudflare, and linkerd2-proxy replaced Linkerd 1.x (Scala/JVM). Neither would be a Go-to-Rust migration **(needs verification)**. Claiming to "rebuild a documented Go-to-Rust migration" would then damage credibility with exactly the audience the project targets.
- Options: reframe as "proxies are the workload class where Rust is often chosen", treat Go as a counterfactual ("what would they have given up by choosing Go, as Traefik and Caddy did?"), or find a real Go-to-Rust proxy migration to cite.
- Experiment 3 also has no named source story. Cite one or call it a synthetic density experiment.

**B6. Local kind/k3d numbers cannot be published.** (all lenses except comprehension, which raised it as Major)
- kind and k3d nodes are containers on one host. On macOS they run inside a VM. k6, the system under test and the Datadog agent share CPU, memory bandwidth and thermal limits, so "k6 on separate nodes" is not real isolation locally.
- Shared-vCPU cloud nodes have noisy neighbours of the same size as the tail effects being measured.
- Local use: correctness and pipeline checks only. Published numbers: cloud only, dedicated-CPU nodes, separate node pools for system under test and load generator, Guaranteed QoS pods.

**B7. Datadog cannot be the source of truth, and its availability is unconfirmed.** (design, structural, codebase-details, implementation-readiness, inventive-catalyst, zoom-out)
- Unknowns **(needs verification)**: whether the GitHub Student Pack offer includes Continuous Profiler and APM; host count and retention; how mature Rust profiling is. Reviewers believe Go uses the in-process pprof-based profiler (dd-trace-go) while Rust would use the native profiler (ddprof) or an eBPF profiler, which means different mechanisms and different overheads, so "one profiler for both languages" may not hold.
- The agent DaemonSet uses CPU and memory on the measured node, which distorts pods-per-node and CPU-per-request. Profiler overhead may differ by language and bias tail latency.
- Datadog APM percentiles come from sampled traces and aggregated metrics. Whether they match client-side histograms **needs verification**.
- Datadog data expires, and readers cannot access it. In two years, dashboard links in the posts will be dead.
- Resolution: separate *measurement runs* (profiler off, agent overhead recorded or agent kept off the measured node) from *profiling runs* (only to explain where time goes). Numbers in posts come from files committed to the repo. Datadog is the diagnosis and presentation layer. Make the Datadog go/no-go the first checkpoint (C1), and choose a fallback in advance: pprof for Go plus perf/flamegraph for Rust, or a single eBPF profiler such as Parca or Pyroscope for both **(needs verification)**.

**B8. "Fairness rules" and "tuned baseline" have no content.** (comprehension, implementation-readiness, structural, design)
- A sceptical reader will assume the author chose whatever made Rust win. If a rule changes after runs start, every earlier run becomes incomparable.
- A versioned `FAIRNESS.md`, frozen before code generation, must decide:
  - toolchain pins and build flags (Go version, PGO; Rust release profile, LTO, codegen-units)
  - Rust allocator (system, jemalloc or mimalloc)
  - HTTP stack class and whether frameworks like Pingora are allowed
  - library vs hand-written data structures
  - whether Go may use pointer-free layouts
  - `unsafe` policy
  - the AI process (same prompt template, iteration cap, prompts committed)
  - a matching tuning budget for each language: GOGC sweep, GOMEMLIMIT fraction and GOMAXPROCS for Go; allocator, tokio worker count and runtime flavour for Rust

#### Majors

**M1. The seven-metric grid does not apply evenly.** (structural, comprehension, design, zoom-out, codebase-details)
- GC pause and GC CPU share are *N/A* for Rust, not zero. A "Rust: 0" cell looks rigged. Replace with "memory-management CPU share" for both: GC plus allocator for Go, allocator plus drop for Rust.
- Pods per node is derived from memory per pod and the latency target.
- Startup time and image size are build-artefact properties. Both languages produce small static binaries, and pod startup is dominated by scheduling and image pull **(needs verification)**. Report them once per service in one table.
- "Memory per pod" means bytes per entry in experiment 1 and bytes per connection in experiment 2. Use different names.
- Publish an applicability matrix (Primary / Secondary / N/A with a reason) before any runs. Give each post one headline metric and put the full table in an appendix.

**M2. Run protocol and load-generator correctness.** (implementation-readiness, design, codebase-details, inventive-catalyst)
- Use the open model only (k6 `constant-arrival-rate` or `ramping-arrival-rate`) to avoid coordinated omission. A run with dropped iterations is invalid.
- Discard 2-5 minutes of warm-up. Pre-fill the cache before measuring. Experiment 1 needs a measurement window of at least 10 minutes so it spans several forced-GC intervals.
- At least 5 repetitions with fresh pods, interleaving Go and Rust (ABAB). Report the spread across runs.
- p99.9 needs roughly 10^5 or more samples per run.
- A run is invalid if the k6 node's CPU goes above about 70%.
- High connection counts (experiment 2) hit kernel limits first: `ulimit -n`, ephemeral ports, conntrack, somaxconn. Whether k6 can hold tens of thousands of connections **needs verification**. Consider several k6 pods, or wrk2/oha for that tier.

**M3. CPU throttling looks like GC.** (design, codebase-details)
- CFS quota throttling (100 ms period) produces tail spikes that can be mistaken for GC pauses, especially when GC workers burst. Record `container_cpu_cfs_throttled_periods` for both languages. Use Guaranteed QoS, and the static CPU manager if the provider allows it **(needs verification)**.
- Container-aware GOMAXPROCS: reviewers believe Go 1.25 made the default respect the cgroup CPU limit, with uber-go/automaxprocs the earlier fix **(needs verification)**. Pin the value explicitly either way and record it.

**M4. Metric definitions.** (comprehension, implementation-readiness, codebase-details)
- *Memory per pod:* cgroup working set (`memory.current` / `working_set_bytes`), which the kubelet and OOM killer act on, plus Go `HeapInuse` as explanation.
- *Pods per node:* largest N such that N replicas at R rps each keep p99 under T ms for D minutes on node type S with requests = limits = M. Subtract system-reserved memory and DaemonSet overhead, including Datadog, and report them separately.
- *Behaviour at memory limit:* a scripted ramp of the working set from about 50% to 110% of the limit, recording p99, GC CPU share, restarts and cgroup `memory.events`. Report outcome categories: graceful degradation, GC thrash, OOMKilled. Reviewers believe Go caps GC CPU at about 50% under GOMEMLIMIT to avoid a death spiral **(needs verification)**. Rust has no equivalent knob, so bound the cache in bytes, not entries, in both languages. Rust RSS can also grow from allocator fragmentation, which belongs in the "gap is smaller" section.
- *Startup:* process exec-to-ready, measured separately from pod Ready, with images pre-pulled.
- Pinned `runtime/metrics` names (`/cpu/classes/gc/total:cpu-seconds`, `/sched/pauses/total/gc:seconds`) **need verification** for the chosen Go version. Datadog's Go runtime metrics may not include GC CPU share or assist time, so a small exporter may be needed.

**M5. Proxy fairness: framework vs language.** (design, codebase-details, comprehension, inventive-catalyst, zoom-out)
- Pingora or hyper against Go `httputil.ReverseProxy` compares library design (pooling, buffer reuse, HTTP/2 handling), not runtimes. Declare one rule: "best production library per ecosystem" (labelled as such) or "equal-level primitives" (net/http vs hyper). Fix protocol, keep-alive, pool sizes, timeouts and TLS on/off on both sides.
- Prove the backend is not the bottleneck with a direct k6-to-backend baseline. Put the backend on its own node.
- Split "many mostly idle connections" (memory per connection) from "high request rate" (CPU per request, tail latency).
- Pingora (edge proxy) and Linkerd (mTLS sidecar) are different models. Pick one.

**M6. Behavioural equivalence for AI-written code.** (structural, design, implementation-readiness)
- Each experiment needs a language-neutral spec and a black-box conformance suite run against both images. For the cache, assert identical hit ratios: a different hit ratio confounds the latency comparison.
- Keep a checklist of known traps: Go body draining and client pooling, mutex contention vs sharding, blocking calls in tokio, a Rust LRU that never evicts.
- Before measuring, profile each implementation and confirm the hot path matches the spec.

**M7. Explainability in interviews.** (inventive-catalyst, comprehension, design)
- Split the code into a *rig* (IaC, k6, dashboards, scripts), which AI writes freely, and a *hot path* of about 100-200 lines per language (node layout, eviction, request loop), which the author annotates line by line or writes from an AI draft.
- Pre-register predictions (direction, rough size, mechanism) before each run. Explaining a missed prediction makes an interview story that does not depend on who typed the code.
- Keep a log of AI mistakes the author caught. It is evidence of judgement and works as blog material.
- In posts, describe the author's role as "designed and measured", not "built optimised services".
- Have a Go or Rust practitioner review the baseline once if possible. If no review happens, say so in the post.

**M8. Results provenance and reproducibility.** (structural, implementation-readiness, design)
- Each run writes an immutable directory `results/<experiment>/<env>/<timestamp>-<sha>/` containing:
  - `manifest.yaml`: git SHA, FAIRNESS version, image digests, toolchains, runtime env vars, node type, kernel, cgroup version, k6 script hash
  - `raw/`: k6 output plus cgroup, kube and runtime metrics
  - `derived.json`: produced by one shared script
- Charts are generated from committed files and refuse to mix environments.
- Keep a `claims.yaml` per experiment (source URL, quoted claim, conditions such as Go version, our reproduction). Do not quote Discord, Cloudflare or Linkerd figures from memory.
- Define in advance when a difference counts as real (non-overlapping ranges across repetitions, or bootstrap confidence intervals).

**M9. Budget and cost control.** (implementation-readiness, design, zoom-out)
- DigitalOcean Student Pack credit amount and expiry, dedicated-CPU node availability on DOKS, and Datadog per-host billing beyond the student allowance all **need verification**.
- Create and destroy the cluster per session by script, set a billing alert, and never leave a cluster running between sessions.

**M10. Undefined terms for readers.** (comprehension)
- Readers will not know "Read States", "Pingora/Linkerd-style" or "rebuilds the conditions". Each post should open with the sourced claim, its original baseline and Go version, and a list of reproduced and non-reproduced parameters (entry count, key/value size, request rate).
- Publish one methodology page, linked from every post.

#### Nits

- Remove the optional database entirely. Off the hot path it adds nothing. On the hot path it hides runtime differences. (zoom-out, design, implementation-readiness)
- Use the same multi-stage, static, scratch or distroless build pattern in both languages, so image size reflects the binary. (implementation-readiness)
- Date-stamp every result "as of <date>, Go x.y / Rust x.y". Toolchains will move on. (structural)
- Record kernel, cgroup v2 and container runtime in each manifest. (implementation-readiness)
- Number the seven metrics explicitly. Several are currently bundled pairs. (codebase-details)
- Record the rejected parser experiment and the demoted A/B switch in a non-goals list so they do not creep back. (structural)

### Inventive resolutions of core contradictions

| Contradiction | Resolution |
| --- | --- |
| Breadth vs depth | Build one shared harness first, so each experiment is a service plus a config. Collapse experiment 3 into a campaign over existing images. Ship one post at a time. |
| Fair Go baseline vs visible Rust gap | Invert the story: the Go ladder is the content. "This mechanism costs X; Go recovers Y at complexity cost Z; the rest is the real gap." A small gap is a publishable result. |
| AI-written code vs interview credibility | Separate the rig from the hot path. The author owns the hot path, pre-registers predictions and logs AI mistakes. |
| Not a specialist vs explaining mechanisms | Sweep the causal variable instead of measuring one point: entry count 1M/2M/5M/10M and pointer-bearing vs pointer-free values (experiment 1), connection count (experiment 2), memory limit (experiment 3). The shape of the curve is the explanation. |
| Datadog showcase vs measurement independence | Ground truth comes from client-side and kernel counters. Datadog shows how the results were diagnosed. Measure profiler overhead once per language and publish it, which is itself useful content for a Datadog audience. |
| Demo appeal of live A/B switch vs rigour | The demo uses results but does not produce any. Repurpose it as a live memory-limit demo: shrink the limit and show Go without GOMEMLIMIT OOM-killed, Go with GOMEMLIMIT degrading, and Rust holding. Label it "illustrative, not a measurement". Note that keep-alive connections stay pinned to old pods, so a live LB switch is not clean anyway. |
| Adversarial fairness | Optionally give each language its own AI session prompted to "beat the other implementation, given its numbers", with the same fixed round budget for both and prompts logged. This gives an auditable fairness procedure instead of a claim. |

### Proposed system design (for an Excalidraw diagram)

**Components**

1. **Developer laptop**
   - Local Docker with `--memory`/`--cpus` (or kind/k3d) for functional and pipeline checks only.
   - Repo containing `harness/`, `experiments/`, `results/`, `docs/`, `posts/`, `showcase/`.
2. **Run orchestrator** (script, e.g. `make run EXP=cache VARIANT=go-tuned ENV=do`). Calls IaC to create the cluster, deploys, warms up, runs, collects data, then tears the cluster down.
3. **Cloud Kubernetes cluster** (e.g. DOKS) with three node pools, each pinned by taints and nodeSelectors:
   - **SUT pool:** dedicated-CPU nodes running one variant at a time (Go-default, Go-tuned, Go-pointer-free or Rust), Guaranteed QoS.
   - **Load pool:** k6 pods using the open-model executor.
   - **Support pool:** for experiment 2, the trivial backend; also the Datadog cluster agent and metrics scraping, where possible.
4. **Datadog node agent** on SUT nodes, with fixed and recorded overhead. Profiler off in measurement runs, on in separate profiling runs.
5. **Collectors:** k6 raw output; cgroup and kubelet/cAdvisor metrics (working set, `memory.events`, CFS throttling); Go `runtime/metrics` exporter and gctrace logs; Rust allocator stats.
6. **Results store:** committed `results/<exp>/<env>/<run>/` with `manifest.yaml`, `raw/` and `derived.json`.
7. **Analysis script:** produces charts and tables from `derived.json`.
8. **Blog posts:** methodology post, experiment posts, Datadog screenshots stored as static images.

**Data flow**

1. Orchestrator -> IaC creates the cluster -> deploys the SUT image (pinned by digest) and the k6 job.
2. k6 (load pool) -> HTTP requests -> SUT pod (SUT pool). For experiment 2, SUT -> backend (support pool).
3. k6 -> raw latency samples -> orchestrator -> `results/raw/`.
4. SUT node -> cgroup, kubelet and runtime metrics -> collectors -> `results/raw/`.
5. SUT node -> Datadog agent -> Datadog (dashboards and profiles) -> screenshots into `posts/`. This path explains results and does not produce the published numbers.
6. `results/raw/` -> analysis script -> `derived.json` -> charts and tables -> blog posts.
7. Orchestrator -> IaC destroys the cluster.

---

## Goals vs. Non-Goals

### In-Scope Goals

- One flagship experiment (the big cache), done end to end, with a Go ladder (default, tuned, pointer-free) against Rust and an explanation of the mechanism that the author can defend line by line.
- A density and memory-limit campaign over the cache images, covering what experiment 3 aimed for without a third codebase.
- A shared, reusable measurement harness, with published numbers that can be reproduced from committed raw files.
- A methodology page published first: fairness rules, metric definitions, applicability matrix, run protocol, environment spec, pre-registered predictions.
- Datadog used as the diagnosis and presentation layer, in a way that shows working knowledge of the target employer's product.
- An honest "where the gap is smaller than the original story claimed" section, with sourced claims.

### Out-of-Scope Non-Goals

- Publishing any number from local kind/k3d or laptop runs.
- Using Datadog as the system of record for published latency or resource numbers.
- A database, whether on or off the hot path.
- The fourth experiment: CPU-heavy parser with a Python baseline.
- A live A/B load-balancer switch as a source of evidence. It may stay as an illustrative demo only.
- A general "Go vs Rust" verdict. Claims are limited to the measured workloads and pinned versions.
- Experiment 2 (the proxy) before the cache post is published (checkpoint C11). After that it is a follow-on, not part of this scope.

---

## Alternatives Weighed

1. **Live A/B switch at the load balancer from Go to Rust pods as the main evidence.**
   Rejected by author: the facilitator argued it is a demo, not a measurement, and that isolated repeated runs are needed. It may be kept as a blog showcase. Reviewers added that keep-alive connections pin to old pods and mix warm-up effects.

2. **Fourth experiment: CPU-heavy parser with a Python baseline (Pydantic v2 / ruff / uv story).**
   Rejected by author: scope concern. Reviewers agree. It could become a separate short post later.

3. **Three independent experiments, each with its own Go/Rust pair (the anchored hypothesis).**
   Rejected by author: too wide for the depth the goal needs (B1, B2). Cache first instead.

4. **One flagship experiment (cache) with experiment 3 folded in as a section or campaign, and the proxy deferred.** (zoom-out, design, implementation-readiness, structural)
   **Chosen by author.** The proxy is gated behind checkpoint C11 (cache post published) rather than a date.

5. **Two builds (cache and proxy) plus a packed-node campaign over both images, published as three posts.** (inventive-catalyst, structural)
   Rejected by author: planning two builds up front spreads the fairness and explanation work too thin. The proxy can still follow after the gate.

6. **Two-way comparison (tuned Go vs Rust) vs a Go ladder (default, tuned, pointer-free, Rust).**
   Open. Reviewers strongly favour the ladder (B3).

7. **Datadog as the single observability and profiling stack vs Datadog as display layer with an independent ground truth.**
   Open. Reviewers favour the second option (B7), pending the checkpoint C1 verification.

8. **Profiler choice: Datadog per-language profilers vs a single eBPF/perf profiler for both languages.**
   Open. One eBPF or perf profiler makes "one profiler" literally true but may lose Go goroutine and GC-phase detail (codebase-details M3). Needs verification of Datadog Rust support.

9. **Development environment: kind/k3d locally vs plain Docker with cgroup limits.**
   Open. Plain Docker reproduces the cgroup limits that GOMEMLIMIT reacts to, with less setup (zoom-out). Kubernetes stays for the cloud runs.

10. **Proxy implementation rule: best production library per ecosystem (e.g. Pingora) vs equal-level primitives (net/http vs hyper).**
    Open. Must be decided and labelled before code generation if the proxy goes ahead (M5).

11. **Proxy framing: re-test of a migration story vs counterfactual ("what if they had picked Go").**
    Open. Depends on verifying the Pingora and Linkerd origins (B5).

---

## Key Decisions & Staging for Next Steps

### Decisions needed before writing code

1. **Scope (resolved).** Cache first, with the density and memory-limit campaign run over the cache images. The proxy gate is a checkpoint, not a date: proxy work starts only after the cache post is published (C11).
2. **Baseline shape.** Adopt the Go ladder and pre-commit that a small or zero gap is publishable.
3. **Fairness.** Write and freeze `FAIRNESS.md` v1 (B8). Any later change bumps the version and invalidates earlier runs.
4. **Metrics.** Write `METRICS.md` with formulas, sources and the applicability matrix (M1, M4). Replace GC metrics with memory-management CPU share for cross-language rows.
5. **Run protocol.** Open-model load, warm-up, window, repetitions, ABAB order, validity checks (M2, M3).
6. **Source of truth.** Committed raw files. Datadog for diagnosis only. Profiler off in measurement runs.
7. **Experiment 1 parameters.** Entry counts, key and value sizes, key distribution, read/write/evict ratio, pointer-bearing vs pointer-free values, cache bounded in bytes, memory limit relative to heap.
8. **Definition of done per experiment.** Conformance suite passes on both images; run matrix complete with N clean repetitions; metrics filled in or N/A with a reason; raw results committed under a tag; post published with sourced claims, a "gap is smaller" section and a predictions-vs-outcome table.

### Verification checklist (checkpoint C1, before any design depends on it)

- Datadog Student Pack: APM, Continuous Profiler, host count, retention, and Rust profiler support and maturity.
- DigitalOcean Student Pack credit amount and expiry, dedicated-CPU nodes on DOKS, CPU manager policy.
- Discord Read States post: date, Go version, spike interval, entry count, whether GOGC tuning helped.
- Pingora and linkerd2-proxy origins (believed to be NGINX/C and Scala/JVM, not Go).
- For the pinned Go version: `forcegcperiod`, GOMEMLIMIT GC CPU cap, container-aware GOMAXPROCS, Green Tea GC default status, `runtime/metrics` names.
- k6: open-model executor behaviour, connection ceilings per instance.
- Parca or Pyroscope eBPF as the fallback profiler for both languages.
- Current maintenance of candidate libraries (hashicorp/golang-lru, bigcache/freecache, the `lru` and `moka` crates).

### Checkpoints (ordered tasks, no schedule)

Each checkpoint is a self-contained piece of work with a done-condition. Do them in order; the author decides the pace. Where two checkpoints do not depend on each other, that is noted.

1. **C1. Verify facts and decide on Datadog.** Work through the verification checklist above.
   *Done when:* every item is confirmed or refuted with a source link recorded in `claims.yaml`, and there is a written Datadog go/no-go with a named fallback profiler.
2. **C2. Write the rules before any code.** `FAIRNESS.md` v1, `METRICS.md` (formulas, data sources, applicability matrix with N/A for Rust GC rows), the run protocol (open-model load, warm-up, window, repetitions, ABAB order, validity checks), and pre-registered predictions.
   *Done when:* all four are committed and v1 is frozen. Any later change bumps the version and invalidates earlier runs.
3. **C3. Specify the cache and its conformance suite.** API contract, entry counts, key and value sizes, key distribution, read/write/evict ratio, pointer-bearing vs pointer-free values, cache bounded in bytes.
   *Done when:* the conformance suite runs against any implementation through its HTTP API.
4. **C4. Build the four variants.** Go default, Go tuned, Go pointer-free, Rust.
   *Done when:* all four pass conformance, and the author has annotated each hot path and can explain it. An "AI mistakes" log records what was wrong in generated code.
5. **C5. Build the harness locally.** Docker with `--memory`/`--cpus` (or kind/k3d), orchestrator script, k6 open-model scripts, collectors, results schema, analysis script.
   *Done when:* one end-to-end local run writes a complete `results/` folder and the analysis script turns it into a chart. These numbers are never published.
   *Can run alongside C3–C4.*
6. **C6. Reproduce the mechanism locally.** Show the default-Go spike pattern and GC mark CPU in gctrace and `runtime/metrics`; produce flame graphs for each variant.
   *Done when:* the mechanism is visible and the author can explain it. If it does not reproduce, record that as a finding and revisit C3 before going further.
7. **C7. Cloud measurement campaign for the cache.** IaC creates the cluster with three node pools. Run the entry-count sweep with repeated runs in ABAB order and validity checks (CPU throttling, load generator saturation). Commit the raw files, then destroy the cluster.
   *Done when:* the run matrix is complete with N clean repetitions per cell, raw results are committed under a tag, and the cluster is gone.
8. **C8. Density and memory-limit campaign.** Sweep the memory limit on the cache images: pods per node at the latency target, OOM vs GOMEMLIMIT behaviour, startup time, image size.
   *Done when:* the same standard as C7.
9. **C9. Profiling runs and Datadog.** Separate profiler-on runs to explain where time goes, Datadog dashboards and screenshots, and measured profiler overhead per language.
   *Done when:* screenshots are saved under `posts/` and the profiler overhead numbers are recorded.
   *Can run alongside C8.*
10. **C10. Publish the methodology post.** Can be published any time after C7.
11. **C11. Publish the cache post** (including the density section, a predictions-vs-outcome table, and a "where the gap is smaller than the story claimed" section).
    *Done when:* the post is published with every external claim sourced. **This is the proxy gate.**

**After the gate (optional follow-ons, in any order):** the proxy experiment (first settle its framing, which depends on the B5 verification), an old-Go-version comparison, the live memory-limit demo video, and the CPU-heavy parser post with a Python baseline.

### Suggested repository layout

- `harness/` (cluster IaC with local and cloud overlays, orchestrator, observability config, results schema, analysis)
- `experiments/cache/` and later `experiments/proxy/` (spec, contract, `go/`, `rust/`, conformance, load scripts, tuning configs, `claims.yaml`)
- `results/`
- `docs/` (METHODOLOGY.md, FAIRNESS.md, METRICS.md, non-goals)
- `posts/`
- `showcase/` (demo material, kept out of results)
