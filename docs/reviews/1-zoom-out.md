# Goldfish Review Report: Stage 5, Initial Zoom Out Review

**Target:** `docs/designs/big_cache.md` (lines 1-848)
**Mode:** REVIEW. GoldfishContext: none.

## Reviewer Status

| Reviewer | Status | Raw findings |
| --- | --- | --- |
| `egm-zoom-out` | completed | 8 Major, 4 Nit. Verdict: Revise |
| `egm-inventive-catalyst` | completed | 8 Major, 3 Nit (one Nit groups three items). Verdict: false compromises found |

**After deduplication:** 0 Blockers, 14 Major, 6 Nits.

Neither reviewer raised a Blocker. Both say the objective, motivation and the core ladder idea are sound. The problems are gaps in measurement method and contradictions between sections, and several of them could make headline numbers hard to defend.

About the "needs verification" items: both reviewers say most are handled safely, because they are gated at C1 or have fallbacks. Two exceptions are noted: the Rust memory-management CPU share depends on Datadog Rust profiler support, which has not been verified (finding M1). The time alignment assumes gctrace timestamps are wall-clock time, but the document never flags that assumption (finding M10).

---

## Blockers

None.

---

## Major

### M1. Memory-management CPU share is measured differently for Go and Rust, and the Rust figure depends on Datadog
* **Reviewer:** `egm-zoom-out`, `egm-inventive-catalyst` (both raised it)
* **Location:** Goals item 2 (lines 84-85); Non-Goals (line 125); Open Questions (lines 135-137); Components item 9 (lines 255-259); D4 (lines 351-354); METRICS.md item 8 (lines 578-580)
* **Problem:** Goal 2 promises this metric for all five variants from the measurement runs. The profiler is off in those runs, though. D4 takes the Rust figure from "profiler samples", and the only profiler in the design is the Datadog Continuous Profiler, used in separate profiling runs that carry overhead. So the Rust number:
  - in practice comes from Datadog, against the Non-Goal;
  - depends on the open question of whether the Student Pack includes Rust profiler support, with no fallback;
  - is measured under different conditions from the Go number, which comes from `runtime/metrics` in clean runs.

  The Go definition also misses its own goal. Goal 2 says "GC plus allocator", but METRICS.md counts GC CPU classes only and leaves out `mallocgc`.
* **Suggested fix:** Use one profiler for both languages that runs outside the process and writes raw files into the run folder. For example, run low-frequency Linux `perf` or eBPF sampling (49-99 Hz) on the SUT node during every measurement run. Attribute matching frames in each language: Go `gcBgMarkWorker`, `gcAssist*` and `mallocgc`; Rust `mi_*` and `drop_in_place`. Commit the raw profiles. Keep Go `runtime/metrics` as a cross-check, and treat Datadog profiles as illustrations only. Measure the sampling overhead with the C9 check. If you do not adopt this, rename the Go metric "GC CPU share" and reword Goal 2 so it does not promise a Rust number from the same run.

### M2. The Go slab and the Rust index layout are not real twins
* **Reviewer:** `egm-zoom-out`, `egm-inventive-catalyst` (both raised it)
* **Location:** Overview (lines 24-34); Cache contract "Values" (lines 279-285); D5 (lines 356-362); item 20 `slab.go` (lines 649-655); item 30a `lru_index.rs` (lines 696-699)
* **Problem:** The Go slab uses `map[uint64]uint32` keyed by hash, with keys and values in byte slabs, so there is no allocation per entry. The Rust index layout uses `HashMap` from key to index with "entries in a `Vec`". As written, each entry probably still owns a heap key and a heap value, which is one or two allocations per entry. Any gap between the two would then come from layout, not language, which is exactly the confusion D5 exists to prevent. Separately, a map keyed only by hash returns the wrong entry when two keys share a hash, unless the stored key is compared. The document does not say it is.
* **Suggested fix:** Keys are exactly 16 bytes, so in D5 and SPEC.md give both layouts an inline fixed key (`[16]byte` / `[u8; 16]`) and the same value-slab scheme. State how many allocations each entry costs, for each layout and language. Require a key comparison on lookup, and add a forced-collision unit or conformance test. If any difference is intentional, list it in FAIRNESS.md.

### M3. The slab reclaim strategy is left open, and both options bring back the cost the rung is meant to remove
* **Reviewer:** `egm-inventive-catalyst` (`egm-zoom-out` also noted it under M2)
* **Location:** item 20 `slab.go` (lines 653-654), "compaction or a free list", deferred to C3
* **Problem:** Values range from 100 to 300 bytes. Compaction holds the shard lock while it moves memory, which causes latency spikes that look like GC pauses. A plain free list with variable sizes fragments memory and inflates bytes-per-entry.
* **Suggested fix:** Decide this in the design, not at C3. Use a memcached-style size-class allocator: fixed slot arrays per class (for example 128, 192, 256 and 320 bytes), each with its own free list, and no compaction. Use it in both languages (see M2). Fix the class sizes in FAIRNESS, and count the unused space inside slots in bytes-per-entry.

### M4. The memory-limit ramp cannot happen while the application's byte bound holds
* **Reviewer:** `egm-zoom-out`, `egm-inventive-catalyst` (both raised it)
* **Location:** "Bounded in bytes, not entries" (lines 272-275); Memory-limit campaign (lines 324-327); item 39 `memlimit-ramp.js` (lines 734-736)
* **Problem:** The cache never holds more than `CACHE_MAX_BYTES`. A ramp to 110% of the pod limit only works if that bound is set above the limit. The document never says how `CACHE_MAX_BYTES`, the pod memory limit and `GOMEMLIMIT` relate. Whether a variant ends in graceful degradation or OOMKilled depends on that choice, so the outcome labels are currently undefined.
* **Suggested fix:** Add a sizing table to the design and RUN_PROTOCOL.md. For each campaign, give `CACHE_MAX_BYTES`, the pod limit and `GOMEMLIMIT`, and how each is derived from entry count and measured bytes-per-entry. For the ramp, say explicitly that the application bound is set above the pod limit (for example 1.5x, recorded in the manifest). Optionally, add a labelled rung whose cache reacts to memory pressure: it reads cgroup `memory.current` or PSI and lowers its own bound, in both languages.

### M5. The design contradicts itself on whether the Datadog node agent runs on SUT nodes during measurement runs
* **Reviewer:** `egm-zoom-out` (Major), `egm-inventive-catalyst` (Nit). Merged at Major.
* **Location:** Components diagram (line 203); Components item 9 (lines 255-259); Density campaign (lines 322-323); item 47 Datadog values (lines 765-768)
* **Problem:** The diagram says the agent runs only in profiling runs. Item 9 installs it for all runs and records its CPU. The density section subtracts its usage. An agent on the SUT node adds noise to the very metrics being compared.
* **Suggested fix:** Pick one rule: no Datadog node agent on the `sut` pool during measurement runs (enforce it with a taint or nodeSelector), installed only by the profiling overlay. Make the diagram, item 9 and the density text match. If it must stay, pin it to reserved CPUs and state that it is the same for every variant.

### M6. The pods-per-node campaign measures the scheduler, not the runtime
* **Reviewer:** `egm-zoom-out` (the entry-count Nit is from `egm-inventive-catalyst`)
* **Location:** Goals item 4 (lines 91-95); Density campaign (lines 320-323); item 40 `density.js` (lines 738-739)
* **Problem:** Pods use Guaranteed QoS on dedicated-CPU nodes, so pods per node is node allocatable divided by the chosen request. That is the same for every variant unless requests are sized per variant, and extra replicas simply will not schedule. The rule "stop when p99 crosses the target" then never fires for runtime reasons. The campaign also does not say which entry count each replica holds.
* **Suggested fix:** Pick one:
  - Define a per-variant sizing rule. For example, memory request = measured peak working set at entry count N plus a fixed headroom, and CPU request from measured CPU per 1,000 requests. Report density as allocatable divided by that size, and confirm it with one packed run.
  - Drop the campaign for now (YAGNI). Derive density from the bytes-per-entry and CPU-per-1,000-requests figures that Goal 2 already produces, and defer the empirical run until after C7.

  Either way, state the entry count per replica.

### M7. The steady workload may never exercise eviction, which is the churn that drives GC
* **Reviewer:** `egm-zoom-out`
* **Location:** Default workload parameters (lines 286-288); item 38 `steady.js` (lines 730-733); Conformance (lines 219-223)
* **Problem:** Keys come from the prefilled key space, and writes replace existing keys. If the bound fits the prefill, nearly every GET hits and eviction rarely runs, so the allocation rate depends only on the replace path. The Discord mechanism and the LRU framing both assume churn. Hit ratio and eviction rate are neither specified nor checked during load runs.
* **Suggested fix:** Specify a target hit ratio and eviction rate, for example a key space larger than capacity by a fixed factor. Record hits, misses and evictions from `/admin/stats` in every manifest. Add a validity check that the hit ratio is within a tolerance of the target, the same for all variants.

### M8. Variance between cloud nodes and sessions is not controlled
* **Reviewer:** `egm-zoom-out`
* **Location:** CUJ 1 (lines 150-159); Orchestrator (lines 228-231); Cost control (lines 490-493); Makefile targets (lines 539-545); session-up/down (lines 769-771); Run protocol step 7 (lines 315-316)
* **Problem:** CUJ 1 creates and destroys a cluster for each configuration, but the Makefile runs many configurations per session. When repetitions span sessions or physical hosts, host variance gets mixed into the comparison, and the bootstrap does not account for it. "ABAB" is also ambiguous with five variants.
* **Suggested fix:** Make the session the unit. Run all variants of a comparison in one session on one pinned SUT node, in randomised round-robin blocks (a shuffled order of all five per round). Record the node ID in the manifest, treat session as a block in the analysis, and rewrite CUJ 1 to match.

### M9. One fixed request rate decides the result, and the document does not say how that rate is chosen
* **Reviewer:** `egm-inventive-catalyst`
* **Location:** Goals item 2 (line 83); Run protocol (lines 308-311); Future Improvements "Throughput at a latency target" (lines 511-513)
* **Problem:** Go's concurrent mark mostly uses spare cores. At low load the GC cost is hidden; near saturation, mark assists dominate. Whoever picks the rate largely picks the outcome, and the curve that would show this has been pushed to Future Improvements.
* **Suggested fix:** Add a calibration run per session to find the saturation rate of the weakest variant. Then measure all variants at the same two or three absolute rates (for example 30%, 60% and 85% of that rate). Write the rule into RUN_PROTOCOL.md and PREDICTIONS.md before any results exist. Note that this roughly doubles or triples measurement time (see M11).

### M10. Lining up GC cycles with latency depends on clocks that are not anchored
* **Reviewer:** `egm-inventive-catalyst`
* **Location:** Goal 1 (lines 75-79); Components item 6 (lines 242-247); Go runtime metrics (lines 294-296); item 56 `gc_align.py` (lines 797-798)
* **Problem:** `gctrace` reports time since the process started (`@N.NNNs`), not wall-clock time. k6 timestamps come from another node's clock. No process-start anchor is recorded, and clock skew between nodes is not discussed. This is an unflagged assumption, not one of the "needs verification" items.
* **Suggested fix:** Record a per-second latency histogram inside each variant, using wall-clock time. In Go, also log GC start and end events with wall-clock timestamps. Alternatively, capture logs with `kubectl logs --timestamps` and record the process start time. Align on the server clock, use k6 only for client-side percentiles, and record the NTP offset of both nodes in the manifest. The instrumentation must be identical in all five variants and covered by FAIRNESS.

### M11. Committing raw per-request output to git conflicts with load-generator headroom, repo size and budget
* **Reviewer:** `egm-inventive-catalyst`
* **Location:** Components items 3, 6, 7 (lines 224-251); item 38 (line 733); item 60 (lines 809-810); Cost control (lines 490-493)
* **Problem:** A rough count: 5 variants x 4 entry counts x 2 value types x 5 repetitions is about 200 runs. At 10+ minutes each plus prefill and warm-up, that is about 50 cluster-hours, before the tuning sweep (12 configurations) and any extra rates. Per-request JSON is gigabytes per run. Writing it costs k6 CPU, which threatens the 70% validity limit, and it bloats git. There is no cost estimate checked against the Student Pack credit.
* **Suggested fix:** Have k6 or a sidecar write per-second HDR histograms, which are small and still allow exact percentiles and time alignment. Commit those, and put any raw output in object storage with its checksum in the manifest. Run the tuning sweep at one entry count only. Add a run-count and cost estimate to the campaign files before C7.

### M12. A cluster started from a laptop has no safety net if the laptop drops off
* **Reviewer:** `egm-inventive-catalyst`
* **Location:** Components item 4 (lines 228-231); Cost control (lines 490-493); item 52 `campaign.sh` (lines 781-782)
* **Problem:** Cost safety is passive: a billing alert plus a refusal to start the next session. If the laptop sleeps or loses network mid-campaign, a dedicated-CPU cluster keeps billing until someone notices. Multi-hour campaigns also depend on the laptop's kubectl session.
* **Suggested fix:** Add a dead-man switch. Put a TTL label on the cluster, and have a scheduled job outside it (for example a GitHub Actions cron) destroy any cluster older than N hours that has no recent heartbeat. Add an "extend TTL" command. Optionally, run `campaign.sh` as an in-cluster Job on the support pool, so the laptop only starts it and collects the results.

### M13. Freezing FAIRNESS before any code exists means every later fix invalidates all runs
* **Reviewer:** `egm-inventive-catalyst`
* **Location:** Goal 6 (lines 100-102); item 7 FAIRNESS (lines 563-575); Fairness versioning NFR (lines 497-499)
* **Problem:** FAIRNESS freezes at C2. Problems found in C4-C7 are almost certain (shard count `S`, whether `parking_lot` is allowed). Each one forces either a full rerun or living with a bad rule.
* **Suggested fix:** Freeze in stages:
  - Freeze predictions and analysis rules at C2.
  - Allow FAIRNESS v0 amendments, each with a dated reason in a published log, until the cloud smoke test at the start of C7. Freeze v1 then.
  - After v1, version each clause, so the analysis refuses only comparisons that a changed clause actually affects.

### M14. The hand-written Go slab weakens the credibility argument the document itself makes
* **Reviewer:** `egm-zoom-out` (the reviewer notes it could be argued down to Nit, since D2 is a deliberate author decision)
* **Location:** The Problem (lines 57-61); D2 (lines 334-341)
* **Problem:** The document argues that Go practitioners would dismiss an untuned, pointer-heavy comparison, then hand-writes the "standard Go answer". A practitioner can just as easily dismiss a home-made slab as a weak version of freecache or bigcache.
* **Suggested fix:** Keep D2, but before publishing, run one existing pointer-free library (freecache is closest to LRU) as a sanity reference for rung 3, labelled as not a strict LRU. If the hand-written slab is much worse, explain why before publishing.

---

## Nits

### N1. The "pointer-bearing versus pointer-free values" sweep axis is undefined
* **Reviewer:** `egm-inventive-catalyst` (becomes Major if it doubles the campaign)
* **Location:** Goal 3 (lines 88-90), compared with "Values" (lines 279-285)
* **Problem:** The cache contract defines only one value shape, and pointer-free storage is already what rung 3 does.
* **Suggested fix:** Either define the second value shape in SPEC.md (for example a fixed inline `[N]byte` value in the pointer layout) and say which variants run it, or remove the axis and state that the rung 1 versus rung 3 comparison covers it.

### N2. Every read takes the shard lock for writing
* **Reviewer:** `egm-inventive-catalyst`
* **Location:** Cache contract / LRU recency on GET; FAIRNESS shard count `S`
* **Problem:** GET updates recency, so every read takes the shard mutex. With 90% reads and Zipf-distributed keys, contention on hot shards could hide GC effects.
* **Suggested fix:** Set `S` well above the core count and record lock contention. Otherwise, state explicitly that the contention is identical in both languages and accepted.

### N3. The document contradicts itself on image count
* **Reviewer:** `egm-zoom-out`
* **Location:** Components item 1 (lines 215-218); Goals item 4 (line 95); Startup and image size (lines 328-330); Verification Plan, Build (lines 822-825)
* **Problem:** One section says five images, one per variant. Another says two, one per language, with the variant chosen by an environment variable.
* **Suggested fix:** Say two images everywhere and report image size per language.

### N4. GOMAXPROCS is both swept and fixed, and the tuning-sweep entry count is unstated
* **Reviewer:** `egm-zoom-out`
* **Location:** Overview rung 2 (lines 22-23); D3 (lines 346-347); FAIRNESS tuning budget (lines 571-574)
* **Suggested fix:** Remove GOMAXPROCS from rung 2's description, since D3 fixes it at the CPU limit. State the entry count(s) for the GOGC x GOMEMLIMIT sweep, and whether the chosen values are reused for every entry count.

### N5. Too much file-level detail while fairness-relevant choices stay open
* **Reviewer:** `egm-zoom-out`
* **Location:** Target files 3-4 (lines 546-551); 15 SPEC.md (lines 611-620); 26 `.golangci.yml` (lines 673-674); 27 Cargo dependencies (lines 678-683)
* **Problem:** The design covers `.gitignore`, linter config and crate lists. Meanwhile it leaves open `parking_lot` versus `std::sync::Mutex`, and never mentions the map hasher in each language (Rust std SipHash versus Go's map hash).
* **Suggested fix:** Collapse the repository-hygiene items into one line. Decide the mutex choice and the hasher for each language in FAIRNESS.md.

### N6. Broken diagram line and odd numbering
* **Reviewer:** `egm-zoom-out`
* **Location:** Components diagram (line 196); target files 30a and 30b (lines 696-703)
* **Suggested fix:** Redraw line 196, which has stray vertical bars through the k6 box. Renumber the file list in sequence.
