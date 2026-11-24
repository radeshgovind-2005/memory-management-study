# Goldfish Review Report (Stage 5 re-audit after author revisions)

**Target:** `docs/designs/big_cache.md` (whole file, lines 1-1124)
**Mode:** REVIEW. **GoldfishContext:** none. Each reviewer ran in a fresh context and was not given any earlier report.

## Reviewer Status

| Reviewer | Status |
| --- | --- |
| `egm-zoom-out` | completed (persona file used: `agents/zoom_out.md`; standard: `resources/document_standards/zoom-out-standard.md`) |
| `egm-inventive-catalyst` | completed (persona file used: `agents/inventive_catalyst.md`) |

## Summary

| Severity | Count |
| --- | --- |
| Blocker | 0 |
| Major | 11 |
| Nit | 12 |

Notes:
- In REVIEW mode the inventive-catalyst persona may only give advisory severities (Major or Nit). That reviewer said that if the gate needs a Blocker, the strongest candidates are its findings on the byte bound (M1) and on the session bootstrap (M2).
- Zoom-out's verdict: the design passes with Major findings. The core idea is sound and well scoped. The weak spots are in the harness around it.
- Zoom-out could not check the design against project docs, because the workspace has no README.md or context.md. It could not search for code reuse either, because the repository does not exist yet.
- "Needs verification" items: neither reviewer found any that the design relies on unsafely, with one exception. The fallback for the `perf` check (N12) needs a stated rule.

## Prioritized Findings (Blockers -> Major -> Nits)

### M1. The byte bound makes entry count and hit counts differ between layouts - Severity: Major
* **Reviewer Lens:** `egm-inventive-catalyst`
* **Affected Scope:** Cache service contract, "Bounded in bytes, not entries" (lines 357-362); Conformance suite (lines 249-253); Sizing table, main sweeps row (line 382); Goal 3 (lines 93-96).
* **Discrepancy Description**: Eviction depends on a byte bound, but each layout counts bytes differently:
  - The slab layout charges each entry its full slot size (128, 192, 256 or 320 bytes).
  - The pointer layout charges the real value length.
  - With 100-300 byte values, slab entries are charged about 30 bytes more on average.

  Under the same `CACHE_MAX_BYTES`, the slab variants therefore hold fewer entries and evict at different moments. That breaks the rule that all five variants give identical hit counts on a replayed sequence, and it breaks the 2-point hit-ratio check. The sizing table sets `CACHE_MAX_BYTES` from "measured bytes per entry". But bytes per entry is also a published result (Goal 2), so an output feeds its own input, and the design never says where that number is measured before the first run.
* **Rationale & Recommendation**: Define one logical charge per entry that is the same in every layout, for example 16 + logical value length + a fixed metadata constant, and evict on that charge. Treat physical cost (slot slack, allocator overhead, GC headroom) only as an output: working set and bytes per entry, where bytes per entry is computed after the run as working set divided by entry count. State in the memory-limit ramp that the application bound is logical. Keep reporting slot slack separately.

### M2. The "resample whole sessions" rule cannot work with one session per comparison - Severity: Major
* **Reviewer Lens:** `egm-zoom-out`, `egm-inventive-catalyst`
* **Affected Scope:** Goals item 2 (lines 81-84); Components item 10, Analysis (lines 312-315); Session and run protocol (lines 406-429); `sut` pool of one node (lines 270-272); METRICS.md real-difference rule (lines 800-801); item 58 `analyze.py` (lines 1058-1060).
* **Discrepancy Description**: The protocol puts every variant of a comparison in one session on one node. The real-difference rule bootstraps by resampling whole sessions, so with one session there is nothing to resample. If blocks are resampled instead, the confidence interval leaves out node-to-node and session-to-session variance, which is exactly what a reader who reruns the test on another node will see (CUJ 2). Inventive-catalyst adds that "non-overlapping 95% intervals" is a known conservative, poorly calibrated test.
* **Rationale & Recommendation**: The two reviewers propose different fixes, and the author must pick one and state it in the Technical Plan, METRICS.md and RUN_PROTOCOL.md:
  - **Cheaper (zoom-out):** resample blocks within a session and say plainly that each result holds for one node. Optionally add one short repeat session on a different node, at one configuration, as a robustness check.
  - **Stronger (inventive-catalyst):** run at least 3 independent sessions on different SUT nodes for each headline comparison, each paired and shuffled as now. Bootstrap the paired within-session differences (variant minus reference), resampling sessions and then blocks within them. This costs about 3 times the node-hours. That can be offset by repeating only the load-level sweep.

  Either way, test on paired differences rather than on whether intervals overlap. The choice also affects M5 (budget).

### M3. Results and raw files can be destroyed along with the cluster - Severity: Major
* **Reviewer Lens:** `egm-zoom-out`, `egm-inventive-catalyst`
* **Affected Scope:** CUJ 1 (lines 173-179); Components items 4, 5, 9 and 12 (lines 262-275, 305-311, 321-323); Terraform (lines 993-998); item 48 campaign Job (lines 1011-1015); item 54 `collect.sh` (lines 1037-1041); NFR cost control (lines 672-681).
* **Discrepancy Description**: There are two data-loss paths:
  - Finished run folders (manifest, histograms, folded stacks, exports) stay in the cluster until `make collect`. The reaper is designed to destroy clusters that have been abandoned, which is exactly when nobody has collected them. A reaped 40-hour session would lose every finished run.
  - The object-storage bucket holds the raw files that committed manifests reference by checksum. It is created by the session's Terraform, so `make session-down` (Terraform destroy) would delete it.

  As written, the cost-safety mechanism works against reproducibility (Goal 5, CUJ 2). There are also two output paths to keep straight.
* **Rationale & Recommendation**:
  - Move the bucket and the registry into a separate, long-lived Terraform stack that sessions never destroy.
  - Have the campaign Job upload each run folder, whole, to the bucket as soon as the run is validated.
  - Have `make collect` pull from the bucket, and have `campaign.sh` read resume progress from it.
  - Have `session-up` reconcile Terraform state when the reaper has deleted a cluster outside Terraform.

### M4. The reaper cannot read the heartbeat as specified - Severity: Major
* **Reviewer Lens:** `egm-zoom-out`, `egm-inventive-catalyst`
* **Affected Scope:** Components item 4 (lines 262-267), item 5 (lines 273-274), item 12 (lines 321-323); `reaper.yml` (lines 753-756); item 48 (lines 1011-1015); secrets NFR (lines 685-688).
* **Discrepancy Description**: The campaign Job writes the heartbeat as an "annotation on the cluster", but the reaper lists clusters "through the provider API". The provider API cannot see Kubernetes annotations, and Kubernetes has no cluster object to annotate anyway. Making it work needs one of two things, and neither is specified:
  - a kubeconfig for every cluster in the reaper, or
  - a provider tag updated by the Job, which puts a cloud token inside the cluster and conflicts with the rule that keeps secrets out of it.

  Line 273 calls `ttl-hours` and `heartbeat` "labels", while line 265 says "annotation". Three protections also overlap: TTL plus heartbeat, a billing alert, and refusing to start if a cluster already exists.
* **Rationale & Recommendation**: Choose one of these:
  - Store both TTL and heartbeat as provider-side cluster tags (or as a heartbeat object in storage). State that the Job holds a narrowly scoped token for this, and amend the secrets NFR to match.
  - Drop the heartbeat and enforce "creation time + TTL" from provider metadata only, relying on the Job's own exit path to tear the cluster down.

  Use the same terms in components 4, 5 and 12 and in files 5 and 48.

### M5. No total cost or time estimate, and the full plan may not fit the credit - Severity: Major
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Goals item 2 (lines 81-92); Run protocol (lines 417-420); Campaign shape (lines 435-442); NFR cost control (lines 672-681); Open Questions, cloud provider (lines 159-161).
* **Discrepancy Description**: Each run is prefill, warm-up, at least 10 minutes of measurement, then teardown: roughly 15-18 minutes. The 200 main runs alone come to about 50-60 cluster-hours. On top of that come:
  - the tuning sweep,
  - sampler on/off checks,
  - Datadog on/off checks for each variant,
  - density packing on a larger pool,
  - the memory-limit ramp.

  The design only asks for a cost header in each campaign file and never adds the campaigns up against the Student Pack credit, which is itself still unknown. The size of the campaign is also what forces sessions lasting tens of hours, which in turn drives the heavier cleanup machinery (M3, M4).
* **Rationale & Recommendation**:
  - Add a "Budget" subsection with total node-hours and cost per campaign, based on a stated minutes-per-run figure.
  - Name a minimum publishable campaign, for example the load-level sweep at 5M entries, all five variants, 5 blocks. Run the entry-count sweep, tuning sweep and density work only if credit remains after it.
  - Consider shorter measurement windows at 1M and 2M entries.
  - Include whichever M2 option is chosen in the estimate.

### M6. The memory-limit ramp's outcome is fixed in advance, and "graceful degradation" has no mechanism - Severity: Major
* **Reviewer Lens:** `egm-inventive-catalyst`
* **Affected Scope:** Sizing table, ramp row and following paragraph (lines 384-388); Memory-limit ramp (lines 457-461); Goal 4 (lines 100-103); Future Improvements, memory-pressure-aware rung (lines 710-711).
* **Discrepancy Description**: The application bound is 1.5 times the pod limit, and writes push the working set toward 110% of the limit. No core variant has any way to shed memory, so all of them must end OOMKilled, and Go variants may thrash first. "Graceful degradation" is listed as an outcome category that no core variant can reach. The only design that could reach it is deferred to Future Improvements. The ramp therefore measures how close each runtime gets to the limit before it dies, which is not what Goal 4 claims.
* **Rationale & Recommendation**: Redefine the ramp's metrics as things the design can produce:
  - the working-set fraction at the first sign of distress (p99 above target, GC CPU share above a threshold, `memory.events` high/max counts),
  - the time and entry count reached before OOM,
  - the latency during the approach.

  Then either drop the "graceful degradation" category, or bring the memory-pressure-aware rung into this campaign for both languages. Record the change as a FAIRNESS amendment before v1.

### M7. Server-side latency timing misses the queueing delay that GC causes - Severity: Major
* **Reviewer Lens:** `egm-inventive-catalyst`
* **Affected Scope:** Components item 7 (lines 284-294); METRICS.md p99 definition (lines 789-790); Goal 1 (lines 76-80).
* **Discrepancy Description**: Headline percentiles come from in-handler service time, chosen so they share a wall clock with the GC events. Much of Go's GC cost is CPU taken by mark workers (about 25% of GOMAXPROCS) plus mark assists, and the worker share delays goroutines before the handler's timer starts (accept, read, scheduling). Handler-scoped timing misses that delay. On Rust it misses delay in the tokio task queue in the same way. The design gives up accuracy for alignment when it could have both.
* **Rationale & Recommendation**:
  - Start the server clock at the earliest point each stack exposes, and spell it out in FAIRNESS. On Go, that is a wrapped `net.Conn` or connection-state hook that stamps when request bytes are first read. On Rust, it is a stamp in the hyper service before routing.
  - Make k6 client-side per-second percentiles a co-headline, aligned to the server clock using the NTP offset already recorded in the manifest. A gap between client and server timing then becomes evidence of queueing.

### M8. Hot-shard lock contention is a confound that differs by language - Severity: Major
* **Reviewer Lens:** `egm-inventive-catalyst`
* **Affected Scope:** Concurrency (lines 363-367); Workload (lines 371-376); The Problem (lines 63-66).
* **Discrepancy Description**: With Zipf s of about 1.1, the hottest key takes roughly 9-10% of all requests, and the top keys land on a few of the 64 shards. Every GET takes the write lock, so at 70-110% load the hot shard's mutex can become the bottleneck. Go's `sync.Mutex` (spin, then park, with starvation mode) and Rust's `std::sync::Mutex`, which blocks a tokio worker thread, behave very differently under contention. The design calls this contention "the same in both languages", but it can open a gap that has nothing to do with memory management. That is exactly the accidental rigging the Problem section warns about.
* **Rationale & Recommendation**:
  - In `cpu_share.py`, count lock-wait frames as their own category: Go `sync.(*Mutex).lockSlow`, `semacquire`, `runtime.lock2`; Rust futex and parking frames.
  - Add per-shard lock-acquire counts to `/admin/stats`.
  - Optionally, run one control configuration with uniform keys at the same hit ratio to bound the effect.
  - Do not change the LRU semantics.

### M9. The entry-count sweep changes request rate along with entry count - Severity: Major
* **Reviewer Lens:** `egm-inventive-catalyst`
* **Affected Scope:** Run protocol, Calibrate (lines 408-412); Campaign shape (lines 435-438); Goal 3 (lines 93-96).
* **Discrepancy Description**: Capacity is recalibrated for each entry count, and the sweep runs at 70% of each. At 10M entries go-default's capacity, and so the absolute request rate, is probably lower than at 1M. The latency-versus-entry-count curve therefore changes heap size and offered load at the same time, and the falling rate can flatten the very curve that is meant to explain the mechanism.
* **Rationale & Recommendation**: Run the entry-count sweep at one fixed absolute rate: 70% of go-default's capacity at the largest entry count, used for every count. Report each count's own calibrated capacity as a separate result. Explain in the chart text that differences may shrink at small entry counts.

### M10. Density packing is a second campaign inside the core deliverable - Severity: Major
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Goals item 4 (lines 97-104); Density and memory-limit campaign (lines 444-464); D7 (lines 509-512); item 41 `density.js` (lines 979-981); item 44 Terraform, variable SUT pool size (lines 993-998).
* **Discrepancy Description**: Packing a real node needs its own infrastructure: a multi-node SUT pool, per-replica load routing, its own pod sizing and its own pass rule. Calculated density already falls out of the main-sweep numbers at no extra cost, as Alternative O's own "Pros" say. Library variants are deferred until after C7 to control scope (D2), so treating density packing differently is inconsistent. It is the largest optional piece of scope, even though the author decided D7.
* **Rationale & Recommendation**: Keep calculated density and the memory-limit ramp in the core post. Move the packing run, `density.js` and the variable SUT pool behind a checkpoint after the core results, or into Future Improvements. This also helps M5.

### M11. Per-clause FAIRNESS versioning is built before it is needed - Severity: Major
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** NFR fairness versioning (lines 689-694); item 56 manifest schema (lines 1045-1050); item 58 `analyze.py` (lines 1058-1061); item 62 analysis tests, "changed-clause case" (lines 1071-1073).
* **Discrepancy Description**: Each clause gets its own version, recorded in every manifest. The analysis has to work out which comparisons a changed clause affects, and there are tests for that logic. This is a dependency-tracking system for amendments that may never happen.
* **Rationale & Recommendation**: Use one document version number. Any change after v1 bumps it, and the analysis refuses to mix versions. If a change clearly affects only some variants, the author reruns those and records the reason in the changelog by hand. Drop clause versions from the manifest schema and drop the changed-clause test.

### N1. Go's frame list undercounts memory-management cost - Severity: Nit
* **Reviewer Lens:** `egm-inventive-catalyst`
* **Affected Scope:** Components item 8 (lines 298-302); METRICS.md (lines 791-793).
* **Discrepancy Description**: The Go list leaves out three kinds of memory-management work:
  - sweeping (`runtime.bgsweep`, `runtime.sweepone`, `mspan.sweep`),
  - write barriers (`runtime.gcWriteBarrier*`, `wbBufFlush`),
  - scavenging (`bgscavenge`).

  On the Rust side, `drop_in_place` includes user drop logic that is not allocator work. The counting rule is the same, but the coverage is not.
* **Rationale & Recommendation**: Freeze a reviewed frame list for each language in METRICS.md, with a one-line reason per frame. Cross-check the Go total against the `runtime/metrics` GC CPU classes and publish the difference.

### N2. Rust stack unwinding through mimalloc depends on a C build flag - Severity: Nit
* **Reviewer Lens:** `egm-inventive-catalyst`
* **Affected Scope:** Rust Dockerfile (lines 952-955); FAIRNESS build flags (lines 765-767).
* **Discrepancy Description**: mimalloc is C code built by `libmimalloc-sys`. Rust's frame-pointer flag does not reach it, so `mi_*` frames may not unwind properly and Rust's memory-management share may be undercounted.
* **Rationale & Recommendation**: Add `-fno-omit-frame-pointer` through `CFLAGS` for the mimalloc build and record it in FAIRNESS. Add a C1/C7 smoke check that a synthetic allocation loop attributes more than 90% of its samples to `mi_*` frames.

### N3. Hash maps can resize mid-run, and the hit-ratio simulation ignores sharding - Severity: Nit
* **Reviewer Lens:** `egm-inventive-catalyst`
* **Affected Scope:** Slab and pointer layouts (lines 341-356); `hit_ratio.py` (lines 846-848).
* **Discrepancy Description**: Rust's `HashMap` resizes all at once, while Go's map grows incrementally. A resize during warm-up or measurement would show up as a latency spike in one language only. Separately, a reference LRU with one global list will not match 64 shards that each have their own budget.
* **Rationale & Recommendation**: Create every shard map at its final capacity before prefill, in both languages, and record this in FAIRNESS. Make `hit_ratio.py` simulate the same FNV-1a sharding with a per-shard bound.

### N4. The per-variant pod size has no defined source in the main sweeps - Severity: Nit
* **Reviewer Lens:** `egm-inventive-catalyst`
* **Affected Scope:** Sizing table, main sweeps row (line 382); density CPU request (lines 446-450).
* **Discrepancy Description**: The pod limit is "measured peak working set x 1.3, measured in the session's calibration step", but calibration only runs go-default. The density sizing also uses a "per-replica rate" that is never defined.
* **Rationale & Recommendation**: Add an explicit per-variant sizing pre-run (prefill plus 70% load for a short window) to the session procedure. Define the per-replica rate as a fixed fraction of go-default's capacity at 1M entries.

### N5. Datadog is wired into every session, though it only serves illustration - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Goal 8 (lines 117-119); diagram (lines 226-229); Components item 11 (lines 316-320); item 50 (lines 1019-1023); overhead checks (lines 1115-1117).
* **Discrepancy Description**: The cluster agent runs in every measurement session and collects almost nothing without node agents. Checking profiler overhead on every variant adds 10 run types, even though no number from a profiling run is quoted.
* **Rationale & Recommendation**: Install Datadog (cluster agent and node agent) only in a dedicated profiling campaign. Measure profiler overhead once per language, or describe it qualitatively.

### N6. The 50 ms Go GC poller duplicates gctrace and contradicts "identical instrumentation" - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Components item 7 (lines 284-294); FAIRNESS instrumentation bullet (lines 773-774); item 24 (lines 900-901).
* **Discrepancy Description**: gctrace lines carry each GC cycle's offset from program start, and the wall-clock start time is already logged, so wall-clock GC times need no extra code. The poller is also Go-only, while line 293 claims the instrumentation is identical in all variants.
* **Rationale & Recommendation**: Drop the poller and derive GC event times from gctrace plus the logged start time. Or keep it and reword line 293 to "identical request-path instrumentation".

### N7. Spec details are repeated in up to four places - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Cache service contract (lines 338-376); FAIRNESS contents (lines 765-777); SPEC.md contents (lines 829-844); file entries 19-20 and 29-30 (lines 872-882, 926-937).
* **Discrepancy Description**: Key size, size classes, shard count, hash functions and allocation counts are restated in several places, and the copies will drift apart.
* **Rationale & Recommendation**: Make SPEC.md the single source of these values. The Technical Plan should keep only the fairness-relevant shape, and the FAIRNESS and file entries should point to SPEC.md.

### N8. Startup time doesn't serve the root problem - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Goals item 4, third bullet (line 104); Startup and image size (lines 462-464).
* **Discrepancy Description**: Start-to-ready time with prefill excluded says nothing about memory management.
* **Rationale & Recommendation**: Drop it, or report it once as a footnote.

### N9. The freecache reference run adds a library variant before publication - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** D2 (lines 478-480).
* **Discrepancy Description**: It breaks the design's own rule that library variants come only after C7.
* **Rationale & Recommendation**: Move it to the post-C7 library work, or label it optional.

### N10. The optional k6 per-request output is speculative - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Components item 9 (line 309).
* **Discrepancy Description**: The design already rules it out as a source for any published number. (If M7's client-side co-headline is adopted, per-second k6 aggregates replace this anyway.)
* **Rationale & Recommendation**: Remove it.

### N11. Standard tools aren't named for some custom pieces - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Components item 8 (folded stacks); Components item 10 and item 58 (bootstrap); item 52 `campaign.sh`.
* **Discrepancy Description**: Several pieces would be written by hand although standard tools already do the job.
* **Rationale & Recommendation**: Name a standard stack-folding tool, such as `inferno` or FlameGraph's `stackcollapse-perf`. Use `scipy.stats.bootstrap` instead of a hand-written bootstrap. Consider writing the campaign orchestrator in Python rather than shell, since it shuffles, calibrates and resumes, and the analysis is already in Python.

### N12. The fallback if the `perf` check fails would undermine a headline metric - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out`, `egm-inventive-catalyst`
* **Affected Scope:** Open Questions, CPU sampler (lines 150-154); Goal 2 (lines 88-90); D6 (lines 503-508).
* **Discrepancy Description**: This is a "needs verification" item gated at C1. The concern is how much the design depends on it, not that it is unverified. The fallback "metric reported for Go only" is the option Alternative N rejected. It would remove the cross-language half of a Goal 2 headline metric and break "measured the same way for both languages".
* **Rationale & Recommendation**: State the order of fallbacks at C1: change provider first, then an eBPF profiler, then Go only. Either block Go-only for the headline metric, or say how Goal 2 and the Rust rows of METRICS.md are reworded if it happens. Both reviewers found the other "needs verification" items safe: each has a fallback or is not depended on.
