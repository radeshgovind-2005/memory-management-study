# Goldfish Review Report: Implementation Readiness, Round 2

**Target:** `docs/designs/big_cache.md` (lines 1091-1705, "## Detailed Implementation Plan" to end)
**Stage:** 9, Implementation Readiness Test. Review-loop round 2 (re-audit after author revisions).
**Test applied:** Could an implementer build this from the plan alone without making impactful decisions? Leaving byte-level layouts and full JSON schemas to SPEC.md (C3) is accepted and is not reported. Items marked "needs verification" (gated at C1) are not reported for being unverified.

## Reviewer Status

| Reviewer | Status | Raw findings (B / M / N) |
| --- | --- | --- |
| `egm-implementation-readiness` | completed | 1 / 13 / 12 |
| `egm-codebase-details` | completed | 0 / 17 / 10 |
| `egm-style` | completed | 0 / 17 / 20 |

No reviewer needed a retry.

## Summary

| Severity | Count (after deduplication) |
| --- | --- |
| Blocker | 1 |
| Major | 28 |
| Nit | 31 |

**Verdict: not ready.** Most of the plan is concrete, and codebase-details confirmed that the arithmetic, variant names and most library and flag names are correct. But several choices on the hot path and in the harness are still left to the implementer, and they affect fairness or correctness. The main ones:

- what GET and PUT copy and allocate;
- how much memory the latency histograms take;
- how a response's end is stamped;
- how the workload's random numbers are generated;
- whether records carry a timestamp.

**Deduplication notes:**

- When several reviewers raised the same issue, it appears once, under the reviewer who described it most fully. The others are listed as "Also raised by".
- Two style nits were folded into Majors:
  - line 1235 (THP) into egm-implementation-readiness-r2-10;
  - line 1531 (`raw/`) into egm-implementation-readiness-r2-9.
- One style finding was dropped by rule: line 1464, a CALIBRATE caveat marked "needs verification" that names no checkpoint. Such items are gated at C1.

**What was checked and found consistent (codebase-details):**

- Variant names, layouts and rungs.
- Value length 100-300, the 180-byte minimum charge and the 64 shards.
- Budget arithmetic: 237 runs, 71 and 59 cluster-hours.
- The FAIRNESS factors and the VU formula.
- The tuning grid.
- The reaper interval and TTL.
- These names: `hyper_util::rt::TokioIo`, `http.Server.ConnContext`, HdrHistogram V2 compressed, `experimental-prometheus-rw`, `K6_PROMETHEUS_RW_PUSH_INTERVAL`, `--web.enable-remote-write-receiver`, `summaryTrendStats`, `debug = "line-tables-only"`, the compose keys `cpus` and `mem_limit`, Terraform `lifecycle ignore_changes`, and colons in DigitalOcean tags.

---

## Blocker

### egm-implementation-readiness-r2-1: What GET and PUT copy and allocate is undefined
- **Reviewer:** `egm-implementation-readiness`. Also raised by: `egm-codebase-details` (M2).
- **Location:** Lines 1163-1165 (allocation contract), 1362-1364 (Go `Cache` interface), 1428-1429 (Rust trait), and the layout entries at 1365-1376 and 1430-1441.
- **Problem:**
  - The `Cache` interface and trait have no method signatures.
  - The allocation contract covers only PUT, yet 90% of traffic is GET.
  - In the slab layout, returning a slice into a slab after the shard lock is released is a data race, because the slot can be freed and reused. The slab layout must therefore copy under the lock.
  - The Go pointer layout can safely return its immutable `[]byte` with no copy.
  - The Rust pointer layout frees the `Box<[u8]>` on eviction, so it must copy too, or switch to `Arc`, which changes the allocation counts.
  - Each implementer would pick differently (fresh buffer, pooled buffer, zero-copy), and that choice is exactly the mechanism being measured. It also changes lock hold times.
- **Fix:**
  - Give the signatures in both languages, for example Go `Get(key *[16]byte, dst []byte) (n int, ok bool)` and `Put(key *[16]byte, val []byte)`, and Rust equivalents taking `&mut [u8]` and `&[u8]`.
  - State that `Get` copies the value into a caller-owned buffer while holding the shard lock. The buffer is reused per connection or handler and is never allocated per request.
  - State that `Put` copies `val`, so the caller keeps ownership.
  - State that the counts cover the cache layer only.
  - Add GET hit and GET miss rows (expected 0 / 0) to the allocation table and test them with `testing.AllocsPerRun` and `stats_alloc`.

---

## Major

### egm-implementation-readiness-r2-2: Retained per-second histograms inflate memory metrics
- **Reviewer:** `egm-implementation-readiness`. Also raised by: `egm-codebase-details` (M3).
- **Location:** Lines 1156-1161.
- **Problem:**
  - A histogram from 1 ns to 60 s at 3 significant digits holds about 27,000 counters, roughly 210-220 KB.
  - Keeping one per second across a 10-minute window means about 600 slots, or 120-135 MB in each pod. That is about half the logical size of the 1M-entry cache.
  - This inflates working set and bytes per entry, which are headline results.
  - In Go it also adds live heap, which shifts GC pacing and the `GOMEMLIMIT` headroom, the mechanism under test.
  - Allocating a fresh histogram every second adds GC work in Go only.
- **Fix:**
  - Keep two preallocated live histograms and swap them every second.
  - Encode the closed slot to V2-compressed bytes (a few KB), then `Reset()` and reuse the histogram.
  - Serve only the stored encodings from `/admin/histograms`.
  - Cap retention, for example 1,200 slots, dropping the oldest.
  - State this in the contract and in FAIRNESS. Optionally lower the 60 s maximum.

### egm-implementation-readiness-r2-3: How a response's end is stamped, and which requests are recorded, is not designed
- **Reviewer:** `egm-implementation-readiness`. Also raised by: `egm-codebase-details` (M10), `egm-style` (M5).
- **Location:** Lines 1146-1158, 1253-1257, 1385-1390, 1442-1445.
- **Problem:**
  - A `net.Conn` or `AsyncWrite` wrapper sees bytes, not request boundaries, so it cannot tell which write ends a response.
    - Go flushes its bufio after the handler returns.
    - hyper may merge pipelined responses into one write unless `pipeline_flush` is pinned.
  - Go needs a wrapped `net.Listener`, which is not listed.
  - The plan does not say which second slot a sample goes into (by start time or end time).
  - It does not say whether `/metrics`, `/admin/*` and health requests are recorded. The 2-second scrapes would otherwise enter the histograms.
  - The start rule at 1146-1150 packs two conditions into one sentence, and the pipelining clause repeats part of it.
- **Fix:**
  - Specify the mechanism in each language.
    - Go: wrap the listener in `server.go`. The handler sets `Content-Length`, writes, calls `http.NewResponseController(w).Flush()`, and records the end stamp after `Flush` returns.
    - Rust: the service pushes each expected response byte count into a per-connection FIFO. Set `http1::Builder::pipeline_flush(false)` explicitly. Stamp the end in `poll_write` when the running byte count crosses a response boundary.
  - Put each sample in the slot of the second that contains its end time.
  - Record only `/v1/entries/*` requests.
  - Rewrite the start rule as three cases:
    - (a) First request on a connection: the first read after accept that returns bytes.
    - (b) Later request not yet buffered: the first read that returns bytes after the previous response's last byte was written.
    - (c) Later request already buffered (pipelined): the moment the previous response's last byte was written.

### egm-implementation-readiness-r2-4: Workload random source, Zipf parameters, replay and golden data are under-specified
- **Reviewer:** `egm-implementation-readiness`. Also raised by: `egm-codebase-details` (M6), `egm-style` (M4, M11).
- **Location:** Lines 1127-1144, 1321-1323, 1338-1340, 1492-1501.
- **Problem:** Go conformance, Python `hit_ratio.py`, the stub and k6 JavaScript must agree byte for byte. The plan does not fix:
  - the Zipf exponent (1.1 appears only at line 550);
  - the uniform random source for the Zipf sampler and for the 90/10 GET/PUT choice;
  - how a u64 becomes a double;
  - whether ranks are 0-based or 1-based. Hörmann-Derflinger produces 1..n, but the plan maps rank r to index r starting from 0;
  - how each k6 VU is seeded. With the same seed, VUs replay identical sequences;
  - the replay's seed, length and op mix, and where the "exact expected hit count" comes from.

  Also:
  - Line 1322 says `golden.json` holds "the first 1,000 Zipf ranks for a fixed seed", but the contract (1133-1135) lists only keys and values.
  - splitmix64 in k6 needs BigInt or 32-bit-pair arithmetic. This is slow and may push k6 past the 60%/70% CPU limits.
- **Fix:**
  - Fix splitmix64 as the uniform source, with `u = (x >> 11) * 2^-53`, and say k6 uses BigInt.
  - Ranks 1..K map to index `r-1`. K = `floor(1.25 × N)`, where N is the configured entry count.
  - Draw once for the op (`u < 0.9` means GET), then once for the Zipf rank.
  - Seed k6 as `WORKLOAD_SEED XOR (vu_id << 32) XOR iteration`.
  - Restate the exponent 1.1 in the contract.
  - Define the replay, for example seed 7, 200,000 iterations, run in the `large` pass. `hit_ratio.py` prints the expected hit count, which is written into SPEC.md and `golden.json`.
  - Add the Zipf section to the contract's description of `golden.json`.
  - Add a C1 or C5 check of k6 CPU per iteration.

### egm-implementation-readiness-r2-5: How request bodies are read and responses written is unspecified
- **Reviewer:** `egm-implementation-readiness`.
- **Location:** Lines 1385-1390, 1442-1445.
- **Problem:**
  - "Bodies read fully" leaves the buffering open. Go could use `io.ReadAll` (several growing allocations), a pooled buffer, or a buffer sized from `Content-Length`. Rust could use `collect().to_bytes()` or a fixed buffer.
  - Per-request allocations outside the cache drive Go GC and Rust allocator CPU, so this affects fairness.
- **Fix:**
  - Require `Content-Length`. Reject chunked or missing bodies with 400.
  - Go: `io.ReadFull` into a fixed `[300]byte` buffer from a `sync.Pool`, and write the response with one `Write` and an explicit `Content-Length`.
  - Rust: collect the body and copy it into a fixed `[u8; 300]`, and respond with `Full<Bytes>`.
  - Record both choices in FAIRNESS.

### egm-implementation-readiness-r2-6: Calibration's p99 and its data path are undefined
- **Reviewer:** `egm-implementation-readiness`.
- **Location:** Lines 1584-1587.
- **Problem:**
  - "Worst 10-second p99 from k6's per-second output" means combining per-second percentiles, which the spec forbids (lines 396-397).
  - The plan does not say how `campaign.py` gets that output.
  - It does not say whether one pod serves the whole ramp, or how each step warms up.
  - Calibration sets every rate in the study.
- **Fix:**
  - Choose one method:
    - k6 `--out csv`, with exact 10-second p99s computed from raw samples in `campaign.py`; or
    - `K6_PROMETHEUS_RW_TREND_AS_NATIVE_HISTOGRAM=true`, with `histogram_quantile(0.99, ...[10s])` over non-overlapping windows.
  - State the pod and warm-up policy for each step.

### egm-implementation-readiness-r2-7: Perf stacks cannot be symbolized where they are folded
- **Reviewer:** `egm-implementation-readiness`. Also raised by: `egm-codebase-details` (M14).
- **Location:** Lines 1520-1523, 1545-1550, 1601-1606.
- **Problem:**
  - `inferno-collapse-perf` reads `perf script` text.
  - Producing that text needs the container's binaries, which are reachable through `/proc/<pid>/root` only on the SUT node.
  - The campaign image has no `perf` and runs on the support pool.
- **Fix:**
  - The sampler pod runs `perf script` on the node, through `kubectl exec`, after stopping and before the cache pod is deleted. It writes the text next to `perf.data`.
  - `collect-run.sh` fetches the text and folds it.
  - Update the step order in `run.sh`.

### egm-implementation-readiness-r2-8: The local pipeline has no implementation
- **Reviewer:** `egm-implementation-readiness`. Also raised by: `egm-codebase-details` (M14).
- **Location:** Lines 1176-1179, 1524-1527, 1684-1688.
- **Problem:**
  - `make local-run` is in neither the "inline" list nor the "scripts" list.
  - `run.sh` and `collect-run.sh` are Kubernetes-specific.
  - Nothing calibrates the local capacity that `LEVEL=...` refers to.
  - Nothing produces the manifest's node facts locally.
  - C5's exit criterion depends on this path.
- **Fix:**
  - Add `harness/scripts/local-run.sh`. It brings up compose, prefills, runs k6, and collects histograms, stats, gctrace output and Prometheus exports.
  - Add local calibration, either a script or a `--local` mode in `campaign.py`.
  - State which manifest fields are "n/a (local)", and confirm the schema allows that.

### egm-implementation-readiness-r2-9: Bucket key layout and the cluster-to-campaign link are undefined
- **Reviewer:** `egm-implementation-readiness`. Also raised by: `egm-style` (M9, N29), `egm-codebase-details` (N3).
- **Location:** Lines 1198-1204, 1531, 1536-1537, 1575-1581, 1588-1594, 1608-1609.
- **Problem:**
  - The reaper, `session-up.sh`, `campaign.py`, `collect.sh` and `make status` all depend on bucket object keys that are never defined. These include `reaper/last_run`, the `campaign-done` marker, the schedule, `calibration.json`, `block_verdict.json`, run folders and `raw/`.
  - Clusters carry only `bigcache-session` and `ttl-hours:<n>` tags, so the reaper cannot find a cluster's campaign.
  - "Tag has expired" names no reference time.
- **Fix:**
  - Give the key layout:
    - `campaigns/<id>/schedule.json`
    - `campaigns/<id>/blocks/<n>/block_verdict.json`
    - `campaigns/<id>/campaign-done`
    - `calibration/<entries>.json`
    - `runs/<env>/<run>/`
    - `raw/<run>/perf.data`
    - `reaper/last_run`
  - Add a `campaign:<id>` tag in item 51 and in `session-up.sh`.
  - Reword expiry as "cluster age exceeds n hours", matching line 486.

### egm-implementation-readiness-r2-10: The THP mode is never chosen
- **Reviewer:** `egm-implementation-readiness`. Also raised by: `egm-style` (N25).
- **Location:** Line 1235.
- **Problem:** "Pinned" does not say to which value. Go's runtime and mimalloc react differently to `always`, `madvise` and `never`, so this choice affects fairness.
- **Fix:** Choose the value, for example `madvise` for both `enabled` and `defrag`. Give the init container's exact writes and record the reason in FAIRNESS.

### egm-codebase-details-r2-1: Two seed sources, undefined "entry count", and unnamed load-script inputs
- **Reviewer:** `egm-codebase-details`. Also raised by: `egm-style` (M1, M2), `egm-implementation-readiness` (N1).
- **Location:** Lines 1108, 1127, 1131, 1137, 1317-1319, 1337, 1621.
- **Problem:**
  - The prefill body carries `"seed": S`, but the generator is seeded from `PREFILL_SEED`. Nothing says which one wins.
  - "Entry count" means two things:
    - the configured count (1M, 5M or 10M), which also sets `CACHE_MAX_BYTES`;
    - the count expected after prefill, which is lower.
  - The Zipf key range, the prefill N and the `large` pass all depend on which one is meant.
  - `steady.js` inputs are not listed.
- **Fix:**
  - Keep one seed source, for example the body seed, with the environment variable as its default only.
  - Define two terms and use them consistently:
    - **configured entry count E:** sets `CACHE_MAX_BYTES = E × 280` and prefill N = E;
    - **resident entry count:** the value `hit_ratio.py` reports.
  - List the `steady.js` environment variables, for example `RATE`, `ENTRIES`, `SEED`, `TARGET_URL`, `LATENCY_TARGET_MS`.

### egm-codebase-details-r2-2: Plan and spec disagree on record timestamps
- **Reviewer:** `egm-codebase-details`. Also raised by: `egm-implementation-readiness` (M13), `egm-style` (M7).
- **Location:** Lines 1124-1125, against spec lines 507 and 512-513.
- **Problem:**
  - The plan says "Entries carry no timestamp".
  - The spec says pointer nodes and slab records hold one.
  - SPEC.md is written from this plan, so neither text settles it.
  - Record size feeds the published bytes-per-entry and the GC-scan results.
- **Fix:** Pick one and correct the other text. Then list the record fields once, in the contract.

### egm-codebase-details-r2-3: `map_growth_since_prefill` cannot be measured in Go and may not stay zero in Rust
- **Reviewer:** `egm-codebase-details`. Also raised by: `egm-implementation-readiness` (M5).
- **Location:** Lines 1226-1227, 1365-1369, 1434, 1612.
- **Problem:**
  - Go maps expose no capacity.
  - In Rust, hashbrown can resize under constant delete-and-insert churn. The resize would allocate and break the zero-allocation slab cases.
  - `validate.sh` does not check the stat, although spec line 497 requires it to be zero.
- **Fix:**
  - Go: define the stat as a count of insertions that take `len(map)` above the pre-size hint.
  - Rust: use `HashMap::capacity()` after prefill compared with the current capacity.
  - Note in METRICS that the two measures differ.
  - Add a churn test (10× capacity of evicting PUTs) that asserts no growth. If it fails, pre-size to 2 × bound ÷ 180.
  - Add the check to `validate.sh`.

### egm-codebase-details-r2-4: Kubernetes namespace and RBAC do not match what the scripts do
- **Reviewer:** `egm-codebase-details`. Also raised by: `egm-implementation-readiness` (M8).
- **Location:** Lines 1540-1544, 1556-1563.
- **Problem:**
  - `sut.yaml.tmpl` creates a Namespace, but the RBAC has no verbs on namespaces.
  - Nodes are cluster-scoped and need a ClusterRole, but the plan does not say Role or ClusterRole.
  - A k6 script ConfigMap needs `configmaps` verbs.
  - WebSocket exec can be authorised as `get` on `pods/exec`.
  - The k6 Job's namespace is not given.
- **Fix:**
  - `session-up.sh` pre-creates one namespace, for example `bench`, for the SUT, k6 and the sampler. Remove the Namespace from the template.
  - Use a namespaced Role for the listed verbs plus `configmaps` create and delete, and `get` and `create` on `pods/exec`.
  - Add a ClusterRole with `get` and `list` on nodes.

### egm-codebase-details-r2-5: The Datadog profiling build breaks the Go dependency and file lists
- **Reviewer:** `egm-codebase-details`. Also raised by: `egm-implementation-readiness` (M11).
- **Location:** Lines 1352-1356, 1473-1479.
- **Problem:**
  - `go mod tidy` records dd-trace-go in `go.mod` regardless of build tags. That contradicts "dependencies limited to" three modules.
  - `profiling_ddprof.go` has no path.
  - How Rust's `ddprof` gets into its image is not stated.
- **Fix:**
  - Name the module and pin its version (`github.com/DataDog/dd-trace-go/v2` or `gopkg.in/DataDog/dd-trace-go.v1`). Amend item 19 to say it is compiled only under `-tags ddprof`.
  - Add `experiments/cache/go/cmd/cache/profiling_ddprof.go` with its contents: CPU and heap profiles, with the service name from env.
  - State how `ddprof` is installed in the Rust image.

### egm-codebase-details-r2-6: k6 threshold and client percentiles include fill requests
- **Reviewer:** `egm-codebase-details`. Also raised by: `egm-style` (M13), `egm-implementation-readiness` (M2, server-side part).
- **Location:** Lines 1497-1499, against 1140-1143 and 1255.
- **Problem:**
  - The threshold `http_req_duration{scenario:measure}` gives no value.
  - It includes the cache-aside PUTs tagged `fill`, which the contract excludes from percentiles.
  - The tag key is never named.
  - The scenario is called "measure" in one place and "measurement scenario" in another.
  - The server cannot tell fills apart, so its percentiles include fills while the client's exclude them.
- **Fix:**
  - Name the tag key (for example `op`). Set thresholds on `{scenario:measure,op:get}` and `{scenario:measure,op:put}`, using the latency target as the value.
  - Say what a breached threshold triggers.
  - Use one scenario name throughout.
  - Either send an `X-Tag: fill` header that the server uses to exclude fills, or state and justify the difference in METRICS.

### egm-codebase-details-r2-7: The reset, the measurement window and the perf window are not aligned
- **Reviewer:** `egm-codebase-details`. Also raised by: `egm-implementation-readiness` (N11).
- **Location:** Lines 1493-1496, 1596-1598.
- **Problem:**
  - The reset scenario runs at `startTime` = 3 min, and the measurement scenario's `startTime` is not given. If both are 3 min, early requests land in the histogram before the reset.
  - `run.sh` starts perf "at the 3-minute mark" without saying what the clock starts from.
  - The k6 pod starts with some delay, so the server, client and perf windows can all differ.
- **Fix:**
  - Set the measurement `startTime` to 3 min plus a fixed gap, for example 2 s.
  - Start perf when the reset is observed, by polling `/admin/stats` or the k6 log, and stop it at the end of the window.
  - Record all three window bounds in the manifest.

### egm-codebase-details-r2-8: One conformance case in the `small` pass cannot pass
- **Reviewer:** `egm-codebase-details`. Also raised by: `egm-style` (M10), `egm-implementation-readiness` (N8).
- **Location:** Lines 1333-1337.
- **Problem:** With `CACHE_MAX_BYTES` = 64 KiB, each shard's bound is 1,024 bytes, which fits at most 5 entries. "1,000 keys forced into one shard return their own values" is therefore impossible.
- **Fix:** Rewrite the check. After writing 1,000 same-shard keys:
  - every key still resident returns its own value;
  - the evicted keys are exactly the least recently used;
  - the shard never holds more than floor(1024/180) entries.

  Or run this case with a larger bound.

### egm-codebase-details-r2-9: The Rust mimalloc FFI dependency is missing, and `lib.rs` contradicts itself
- **Reviewer:** `egm-codebase-details`. Also raised by: `egm-style` (M12).
- **Location:** Lines 1413-1416, 1420-1422, 1449-1451.
- **Problem:**
  - `mi_process_info` is not in the `mimalloc` crate's safe API. It needs `libmimalloc-sys` with the `extended` feature, which is not listed.
  - `lib.rs` declares `metrics`, which makes the FFI call, yet line 1422 says tests and Miri link the library "without loading C mimalloc".
- **Fix:**
  - Add `libmimalloc-sys = { features = ["extended"] }`.
  - Move the FFI call into a binary-only module, or gate it behind a feature that tests and Miri turn off.
  - Correct the rationale at line 1422.

### egm-codebase-details-r2-10: The Rust smoke binary would not use mimalloc
- **Reviewer:** `egm-codebase-details`.
- **Location:** Line 1459.
- **Problem:** `src/bin/smoke.rs` is a separate binary. Only `main.rs` sets `#[global_allocator]`, so the smoke allocation loop would run on glibc malloc. The check that more than 90% of stacks are attributed to `mi_*` frames would then fail.
- **Fix:** Set `#[global_allocator]` in `smoke.rs`, or in a shared module that both binaries use.

### egm-codebase-details-r2-11: Several components have no file or owner
- **Reviewer:** `egm-codebase-details`.
- **Location:** Lines 1175-1181, 1492-1501, 1551-1557, 1674.
- **Problem:**
  - The node-facts image is never named, and `make build` lists only four images.
  - No ConfigMap or image delivers the k6 script to the Job.
  - `steady.js` and `test_generator.js` both need the generator, but there is no shared module for it.
- **Fix:**
  - Name the node-facts image and its tools (`chronyc`, `lscpu`, or host mounts), and add it to `make build`.
  - Add a k6 script ConfigMap, or bake the script into a k6 image.
  - Add `load/lib/generator.js` and import it from both files.

### egm-codebase-details-r2-12: Nothing creates the bucket-scoped storage key
- **Reviewer:** `egm-codebase-details`.
- **Location:** Lines 1183-1187, 1528-1532, 1558-1562.
- **Problem:** `.env.example` has one broad Spaces key pair, for Terraform. The campaign Job needs a key limited to one bucket, and no file creates that key or puts it into the Secret.
- **Fix:**
  - Create it in the persistent stack, for example with a `digitalocean_spaces_key` resource (the key-scope question is already gated at C1), and output it.
  - Have `session-up.sh` write the Kubernetes Secret.

### egm-codebase-details-r2-13: The 50,000-sample perf rule cannot be met at the 25% load level
- **Reviewer:** `egm-codebase-details`.
- **Location:** Line 1285, with `validate.sh` at 1611.
- **Problem:**
  - At 99 Hz on about 0.5 busy cores for 600 s, perf collects about 30,000 samples, below the 50,000 minimum.
  - The rule is not in the spec, and `validate.sh` does not check it.
- **Fix:**
  - Lower the minimum (for example 20,000), lengthen the window, or report the sample count instead of using it as a validity rule.
  - Make `validate.sh` match the choice.

### egm-style-r2-1: Rate units conflict across the document
- **Reviewer:** `egm-style`. Also raised by: `egm-implementation-readiness` (N10), `egm-codebase-details` (N1).
- **Location:** Lines 1143-1144, 1508, 1585-1588, against spec lines 605-609 and 675.
- **Problem:**
  - The contract defines rate as iterations per second.
  - The run protocol speaks of requests per second and an "absolute request rate" (500 rps start).
  - The spec keeps calibrated rates in git, while the plan stores `calibration.json` in the bucket.
- **Fix:**
  - State once that every rate, capacity and level is in iterations per second.
  - Rewrite the conversion as "request rate = rate × (1 + 0.9 × miss ratio)".
  - Commit the calibrated rates into the campaign files after calibration.

### egm-style-r2-2: Slab size classes and slot rules are missing from the contract
- **Reviewer:** `egm-style`. Also raised by: `egm-implementation-readiness` (N3), `egm-codebase-details` (N8).
- **Location:** Lines 1118-1123, 1222, 1371-1376.
- **Problem:**
  - FAIRNESS points to "D5 and the contract" for size classes, but the contract does not list them. They appear only at line 514.
  - 64 KiB is not a multiple of 192 or 320.
  - Free lists for entry records are not mentioned.
  - The order of slot release and eviction is unstated. It decides whether an evicting PUT can grow a chunk.
- **Fix:** Add to the contract:
  - four size classes, 128, 192, 256 and 320 bytes; a value goes to the smallest class that fits;
  - slots per chunk = floor(65536 / class), with the remainder unused;
  - free lists for both slots and entry records;
  - on a class change or eviction, the old slot is freed before a new one is taken, in both languages.

### egm-style-r2-3: The allocation-count notation is unclear
- **Reviewer:** `egm-style`.
- **Location:** Lines 1163-1165.
- **Problem:**
  - "1 / 0" is never explained.
  - "Evicting PUT" could mean a new key or a replace.
  - "Once slabs are warm" is attached to the first case only.
- **Fix:**
  - Use a table with columns case / pointer layout / slab layout, and include the GET rows from egm-implementation-readiness-r2-1.
  - Define an evicting PUT as "a new key whose insertion evicts one or more entries".
  - State that every slab count assumes no chunk growth during the measured call.

### egm-style-r2-4: Three session scripts share one undivided list of behaviours
- **Reviewer:** `egm-style`.
- **Location:** Lines 1575-1581.
- **Problem:**
  - It is unclear which script refuses to start, which reconciles state, and which sets the TTL.
  - "Reconcile Terraform state" gives no method.
- **Fix:**
  - Give each script its own sentence:
    - `session-up.sh`: preflight refusals, apply, Helm, the node-type check, the TTL tag and starting the Job;
    - `session-down.sh`: destroy;
    - `extend-ttl.sh`: rewrites the tag.
  - Name the reconcile step, for example `terraform state rm` of the cluster when the provider reports it missing.

### egm-style-r2-5: `derived.json` has two meanings
- **Reviewer:** `egm-style`.
- **Location:** Lines 1633, 1647, 1661.
- **Problem:**
  - `derived.json` is a per-run file.
  - But `analyze.py` also computes paired differences across runs.
  - And `charts.py` reads "`derived.json` only".
- **Fix:**
  - Define a per-run `derived.json` and a campaign-level `results/cache/<env>/summary.json`.
  - Say which one the charts read.

### egm-style-r2-6: Repeats of invalid blocks have no limit
- **Reviewer:** `egm-style`. Also raised by: `egm-implementation-readiness` (N9).
- **Location:** Lines 1286-1288.
- **Problem:** "A block with a variant still invalid is repeated" can loop forever and gives no final outcome.
- **Fix:** Repeat once. If the variant is still invalid, drop the block, flag the configuration as failed in the campaign, and continue.

---

## Nit

### egm-implementation-readiness-r2-11: Bounds are loosely defined
- **Location:** Lines 1117-1118, 1333-1338.
- **Fix:**
  - Compute the shard bound with floor division.
  - Reject `CACHE_MAX_BYTES < 64 × 380` at startup.
  - State that the `large` pass uses `CACHE_MAX_BYTES = 100,000 × 280`.
  - List the entry counts that SPEC.md gives targets for.

### egm-implementation-readiness-r2-12: HTTP edge cases are missing
- **Location:** Lines 1104-1113. Also raised by: `egm-codebase-details` (N7).
- **Fix:** Define:
  - 404 for unknown paths and 405 for wrong methods;
  - 400 for a missing or chunked `Content-Length` and for bad JSON;
  - 409 for a second `/admin/prefill`;
  - how entry requests are handled during prefill;
  - the exact response headers;
  - server timeouts (none, in both stacks);
  - whether `/admin/histograms` includes the current partial second.

### egm-implementation-readiness-r2-13: Scope of reset and stats is loose
- **Location:** Lines 1109, 1111.
- **Fix:**
  - List the counters `/admin/reset` clears. It must not touch `entries`, `charged_bytes` or the growth baseline.
  - Say whether prefill inserts count as evictions or misses.
  - Give the string values of `hash_path`.

### egm-implementation-readiness-r2-14: No owner for derived settings
- **Location:** Lines 1484-1488.
- **Fix:** Name the component (`campaign.py` or `run.sh`) that turns the `GOMEMLIMIT` fraction into bytes (floor of fraction × limit) and sets `CACHE_MAX_BYTES` from the entry count.

### egm-implementation-readiness-r2-15: The campaign image lacks needed tools
- **Location:** Lines 1515-1519.
- **Fix:** Add a template renderer (`envsubst` or gettext), `curl` and `jq`, which `run.sh` needs.

### egm-implementation-readiness-r2-16: Gaps in the test plan
- **Location:** Lines 1453-1458, 1674-1678.
- **Fix:** Add:
  - a Rust counterpart to Go's `server_test.go` (routing, status codes, stamping);
  - unit tests for histogram rotation and encoding in both languages;
  - calibration step tests in `test_campaign.py`.

### egm-codebase-details-r2-14: Long sweeps span several sessions and nodes
- **Location:** Line 1509, Budget, Goal 2 (line 108).
- **Problem:** The roughly 21-hour sweeps exceed the 12-hour session cap. This contradicts "same session on the same node".
- **Fix:** Make the block the pairing unit, record the node for each block, and reword the claim.

### egm-codebase-details-r2-15: The Prometheus PVC escapes the reaper
- **Location:** Lines 1198-1203, 1536-1537.
- **Fix:** Set `server.persistentVolume.enabled: false` in `values.yaml`. The DigitalOcean volume created by the chart's default PVC would not carry the reaper's tag.

### egm-codebase-details-r2-16: The Miri toolchain is not pinned
- **Location:** Lines 1419, 1456, 1674.
- **Fix:** Use `cargo +nightly-YYYY-MM-DD miri test`, and give the test-name filter that selects only the pointer-layout cases.

### egm-codebase-details-r2-17: `.cargo/config.toml` sections are not named
- **Location:** Lines 1416-1418 (spec line 440).
- **Fix:**
  - Put `rustflags` under `[build]` or `[target.x86_64-unknown-linux-gnu]`, and `CFLAGS` under `[env]`.
  - State that the `RUSTFLAGS` environment variable is not used, because it would override the config file.

### egm-codebase-details-r2-18: `stats_alloc` counts are process-wide
- **Location:** Line 1455.
- **Fix:** Put the allocation-count tests in their own test binary, or run them with `--test-threads=1`.

### egm-codebase-details-r2-19: Python dependencies are missing
- **Location:** Lines 1515-1518, 1676.
- **Fix:** Add `pytest` to `harness/campaign/pyproject.toml`, and add a pyproject for `experiments/cache/sim`.

### egm-codebase-details-r2-20: Small mismatches
- **Location:** Lines 1295, 1326, 1682, 1684.
- **Fix:**
  - Reword the NFR to "prediction columns are never edited". Line 1045 says the file is "never edited", but line 1295 appends an Outcome column.
  - Replace `LEVEL=50` with a level that D8 kept (25 or 70).
  - Add a `conformance-stub` target, because `VARIANT=stub` has no variant file.
  - Have `hit_ratio.py` also print the replay hit count and the `large`-pass prefill count.

### egm-style-r2-7: Misleading entry label
- **Location:** Line 1093 and every file entry.
- **Fix:** All files are new, so rename the *Changes:* label to *Contents:*.

### egm-style-r2-8: The Makefile target list is hard to scan
- **Location:** Lines 1175-1181.
- **Fix:** Put one target per line. `build-profiling` and `status` are currently buried in a sentence.

### egm-style-r2-9: Redundant wording
- **Location:** Line 1131.
- **Fix:** Shorten "inserts keys `0..N-1` and stops at `N`" to "inserts keys `0..N-1`".

### egm-style-r2-10: Undefined notation
- **Location:** Line 1140.
- **Fix:** Define `value(i)` as "the generator's value for key index `i`".

### egm-style-r2-11: Unclear referent
- **Location:** Line 1141.
- **Fix:** Change "it is tagged `fill`" to "the PUT is tagged `fill`".

### egm-style-r2-12: Vague term
- **Location:** Line 1191.
- **Fix:** Replace "finalised with the repository home" with "replaced with the final GitHub owner before C3".

### egm-style-r2-13: Unexplained constant
- **Location:** Lines 1226-1227.
- **Fix:** Explain "÷ 180 bytes" by adding "(minimum charge: 16 + 100 + 64)".

### egm-style-r2-14: Factors without operands
- **Location:** Lines 1236-1237.
- **Fix:** Spell out each factor, for example "pod limit = peak working set × 1.3".

### egm-style-r2-15: No unit in the VU formula
- **Location:** Line 1240.
- **Fix:** Give the latency target's unit (seconds).

### egm-style-r2-16: Undefined term
- **Location:** Line 1274.
- **Fix:** Define "baseline variant" as the baseline named for each comparison in the fixed comparison list.

### egm-style-r2-17: Warning threshold differs from the validity limit
- **Location:** Line 1593.
- **Fix:**
  - Explain that the 60% k6 CPU warning is an early warning below the 70% validity limit at line 637.
  - Say that the next session's load node is added by hand.

### egm-style-r2-18: Wrong object
- **Location:** Line 1600.
- **Fix:** Change "Delete the pod" to "delete the Deployment and Service". Deleting only the pod lets the Deployment recreate it.

### egm-style-r2-19: Sentence fragment
- **Location:** Line 1606.
- **Fix:** Replace "The only owner of folding." with "No other script folds profiles."

### egm-style-r2-20: Inconsistent name
- **Location:** Line 1619.
- **Fix:** Replace "worker threads" with `TOKIO_WORKER_THREADS`.

### egm-style-r2-21: Undefined placeholder
- **Location:** Line 1659.
- **Fix:** List the values of `<env>`, for example `local` and `do`.

### egm-style-r2-22: Awkward wording
- **Location:** Line 1681.
- **Fix:** Replace "passes both passes" with "passes `small` and `large`".

### egm-style-r2-23: Informal wording
- **Location:** Line 1690.
- **Fix:** Replace "whatever they show" with "regardless of result".

### egm-style-r2-24: Missing location
- **Location:** Line 1699.
- **Fix:** Name where the result is "recorded and published", for example `TUNING.md` or the methodology post.
