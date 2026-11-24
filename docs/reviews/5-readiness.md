# Goldfish Review Report: Implementation Readiness (Stage 9, round 1)

**Target:** `docs/designs/big_cache.md` (lines 1088-1543, "## Detailed Implementation Plan" to end). Lines 1-1087 were read as the spec.

**Test:** Could an implementer build this from the plan alone, without making decisions that matter? Items marked "needs verification" (gated at C1) were not counted as findings.

## Reviewer Status

| Reviewer | Status |
| --- | --- |
| `egm-implementation-readiness` | completed |
| `egm-codebase-details` | completed |
| `egm-style` | completed |

## Summary

- **Verdict: not ready.**
- After deduplication: **4 Blockers, 31 Major, 25 Nits.**
- The numbers agree with the spec: budget, VU formula, size classes, 180-byte minimum charge, shard count and tuning grid.
- The plan falls short in three areas:
  - **The shared contract between languages.** Wire format, generator and histograms are not defined.
  - **How the harness runs.** Images, file schemas, resume and sequencing are not defined.
  - **Measurement mechanics.** Some of them cannot be built with the chosen tools as written.

Where several reviewers raised the same point, the finding is listed once under the first reviewer and the others are named under "Also raised by".

---

## Blockers

### implementation-readiness-r1-1: The HTTP wire contract and data formats are only headings
- **Reviewer:** egm-implementation-readiness. Also raised by egm-codebase-details.
- **Location:** lines 1226, 1239-1241, 1247-1256, 1290-1302, 1340-1350.
- **Problem:** Five variants, k6, the conformance CLI and the Python analysis must agree byte for byte on several things the plan leaves out:
  - How the 16-byte key is encoded in the URL.
  - The PUT body format and Content-Type.
  - Status codes for success and for a bad key or value length.
  - How `POST /admin/prefill` receives N.
  - The JSON field names and types for `/admin/stats` and `/admin/histograms`.
  - The `/metrics` names and the default `PORT`.
  - The HDR histogram serialization: unit, lowest and highest trackable values, significant digits and wire format.
  - How per-second histograms rotate while workers record into them concurrently. `hdrhistogram-go` has no concurrent recorder, while Rust has `hdrhistogram::sync`. Different locking means different hot-path cost, which breaks the fairness rule.
  - Which HDR library is used. Item 24 says only "an HDR histogram library".
- **Fix:** Put the contract in the plan, or in a SPEC.md skeleton the plan quotes:
  - Key: 32 lowercase hex characters, 400 for anything else.
  - PUT body: raw octets of 100-300 bytes; 413 or 400 outside that range.
  - Exact JSON schemas for both admin endpoints.
  - Histograms: HdrHistogram V2 compressed and base64-encoded, values in ns, 1 ns to 60 s, 3 significant digits.
  - One recording scheme for both languages, for example a mutex-protected histogram per one-second slot, swapped by a ticker. Add this scheme to FAIRNESS.
  - Libraries: `github.com/HdrHistogram/hdrhistogram-go`, the Rust `hdrhistogram` crate and Python `hdrh`.
  - The list of `/metrics` names.

### implementation-readiness-r1-2: The deterministic generator and the workload mix are undefined
- **Reviewer:** egm-implementation-readiness. Also raised by egm-codebase-details (M7) and egm-style (M11).
- **Location:** lines 1232-1237, 1243-1246, 1295-1296, 1345-1346, 1379-1381; 90/10 mix at lines 1235 and 1380.
- **Problem:** "A seeded generator producing key `i` and a value derived from `i` and the seed" must be reproduced exactly in Go, Rust, k6 and `hit_ratio.py`. The plan leaves all of these open:
  - The PRNG. The standard PRNGs differ between languages.
  - How key and value bytes are derived from `i`.
  - The value-length distribution. Its mean sets `CACHE_MAX_BYTES`.
  - How a Zipf rank maps to a key index.
  - Which value k6 sends on a PUT.
  - Whether the cache-aside PUTs count inside the 10%.
  - When prefill stops: at N entries or at the byte bound. Shards fill unevenly, so the expected prefill entry count in conformance is undefined.
- **Fix:**
  - Name the algorithm, for example splitmix64 seeded with `PREFILL_SEED ^ i`.
  - Key = 16 output bytes. Value length = 100 + (r mod 201). Value bytes = the continued output.
  - Zipf rank r maps to key index r. Every PUT sends `value(i)`.
  - State the mix exactly: "90% GETs, 10% independent PUTs; each GET miss also issues a cache-aside PUT, which is not counted in the 10%".
  - State the prefill end condition and the expected entry count.
  - Add a golden-vector file (for example `experiments/cache/testdata/prefill_golden.json`) to the file list. Have the Go tests, Rust tests, `hit_ratio.py` and `steady.js` all check against it.

### implementation-readiness-r1-3: No image exists for the campaign Job or the sampler DaemonSet
- **Reviewer:** egm-implementation-readiness. Also raised by egm-codebase-details (B1).
- **Location:** lines 1103-1108, 1413-1424, 1439-1454, 1512-1513.
- **Problem:** `make build` produces two images, Go and Rust.
  - The campaign Job runs `campaign.py`, `run.sh`, `collect.sh` and `validate.sh`. It needs Python, kubectl, an S3 client for Spaces and `inferno`, but no Dockerfile or Python dependency file exists for it.
  - The sampler "with `perf` installed" has no image, and perf must match the node kernel.
  - Without these images CUJ 1 cannot run.
- **Fix:**
  - Add `harness/images/campaign/Dockerfile`, with a pinned base, kubectl, the aws CLI pointed at Spaces, and `inferno`.
  - Add `harness/campaign/pyproject.toml` (or `requirements.txt`) with pinned dependencies.
  - Add `harness/images/sampler/Dockerfile`, with a base matched to the DOKS kernel, or document running the host's perf through `nsenter`.
  - Make `make build` produce four images and update line 1512.

### codebase-details-r1-1: The Rust integration tests cannot compile against a binary-only crate
- **Reviewer:** egm-codebase-details. Also raised by egm-implementation-readiness (M4), rated Major there.
- **Location:** lines 1313-1355.
- **Problem:** `tests/lru.rs` can import only from a library target. The plan lists `src/main.rs` and modules but no `src/lib.rs`.
- **Fix:**
  - Add `experiments/cache/rust/src/lib.rs` (Create). It declares `lru`, `lru_pointer`, `lru_slab`, `server`, `prefill` and `metrics`.
  - Make `main.rs` a thin binary that keeps `#[global_allocator]` mimalloc. The tests and Miri then do not load C mimalloc.

---

## Major

### implementation-readiness-r1-4: Server-side latency start and end points are undefined
- **Reviewer:** egm-implementation-readiness. Also raised by egm-codebase-details (M6) and egm-style (M12).
- **Location:** lines 1145, 1267-1270, 1290-1294, 1340-1344.
- **Problem:** Go stamps "when request bytes first arrive"; Rust stamps "the first read after each response". Neither covers these cases:
  - Pipelined requests already in the read buffer, where no new read occurs.
  - The first request on a connection.
  - The end point. Go flushes after the handler returns, and hyper writes after the service future resolves.
  - How the stamp reaches the handler.

  An `AsyncRead`-only shim cannot know when a response was written. FAIRNESS requires identical instrumentation, so these gaps matter.
- **Fix:** Define one state machine in SPEC/METRICS:
  - Start = first read that returns bytes after accept or after the last response write.
  - For a pipelined request already in the buffer, start = the moment the previous response finished writing.
  - End = the last response byte accepted by the wrapped `net.Conn.Write` (Go) or `poll_write` (Rust).

  Then name the plumbing:
  - Go: `http.Server.ConnContext`.
  - Rust: a per-connection `Arc<AtomicU64>` under `hyper_util::rt::TokioIo`, wrapping both AsyncRead and AsyncWrite.

  Use the same term in items 18, 22 and 31. Have conformance check the timing values, not just the sample count.

### implementation-readiness-r1-5: `calibrate.js` cannot do what the plan says
- **Reviewer:** egm-implementation-readiness. Also raised by egm-codebase-details (M8) and egm-style (M15, M16).
- **Location:** lines 1373-1377, 1390, 1440.
- **Problem:**
  - A k6 script cannot read its own per-step "worst 10-second p99" at runtime and stop the ramp. `abortOnFail` thresholds are cumulative.
  - "Rate" could count iterations or requests. A GET miss plus its PUT is one iteration but two requests.
  - The first step's rate and the pass threshold are not given.
  - The 150-second per-variant cross-check has no owner.
  - It is unclear whether calibrated rates are written into the campaign files beforehand or computed during the run. `campaign.py` runs in the cluster and cannot commit to git.
- **Fix:**
  - Move step control to `campaign.py`: one short k6 run per step at a given rate, judged from Prometheus or the per-second k6 output.
  - Define the rate as iterations per second and give the formula for the request rate.
  - Give the first-step rate. A step passes if its worst 10-second p99 is within the D10 target and k6 drops no iterations.
  - Have `campaign.py` run the per-variant check.
  - Have calibration write `calibration.json` to the bucket. Campaign files express levels as a percentage of capacity, and the absolute rates are recorded in each manifest.

### codebase-details-r1-2: The `steady.js` load script leaves out details that decide the headline result
- **Reviewer:** egm-codebase-details. Also raised by egm-implementation-readiness (M2).
- **Location:** lines 1378-1385, 1427.
- **Problem:**
  - No Zipf algorithm is named. A per-VU cumulative table over 12.5M keys is about 100 MB per VU.
  - It is unclear whether cache-aside PUTs count in the p99 and p99.9.
  - Measurement-scenario percentiles appear in the k6 summary only if a threshold is defined on the sub-metric.
  - The k6 output and Prometheus receiver flags are not named.
  - "1.25 × capacity" is ambiguous, because "capacity" also means the 100% rate.
- **Fix:**
  - Use rejection-inversion Zipf (Hörmann-Derflinger), with no table.
  - Tag requests by type and state which tags count in the percentiles.
  - Define a threshold on `http_req_duration{scenario:measure}`, with `summaryTrendStats` including p(99) and p(99.9), and output through `handleSummary`.
  - Name `experimental-prometheus-rw` with `K6_PROMETHEUS_RW_PUSH_INTERVAL=1s`, and add `--web.enable-remote-write-receiver` to Prometheus.
  - Write "1.25 × entry count".

### codebase-details-r1-3: Warm-up, reset and measurement sequencing contradict each other
- **Reviewer:** egm-codebase-details.
- **Location:** lines 1381, 1445-1448.
- **Problem:**
  - `steady.js` runs warm-up and measurement as two scenarios of one k6 run.
  - `run.sh` runs warm-up, reset and measure as separate steps. Nothing calls `/admin/reset` or starts perf at the boundary.
  - The warm-up length is never given, but the budget depends on it.
- **Fix:**
  - Use one k6 run, with a one-iteration scenario at `startTime = warmup` that POSTs `/admin/reset`.
  - Have `run.sh` start perf at the same offset.
  - Fix the warm-up length in RUN_PROTOCOL, for example 3 minutes.

### codebase-details-r1-4: Prefill and readiness conflict
- **Reviewer:** egm-codebase-details. Also raised by egm-style (M19) and egm-implementation-readiness (N7).
- **Location:** lines 1255, 1408-1411, 1445-1446.
- **Problem:**
  - `/readyz` fails until prefill finishes. If `run.sh` calls `POST /admin/prefill` through a Service, the not-ready pod gets no traffic, so prefill never starts.
  - It is undefined whether the service prefills itself or the harness triggers it.
  - It is undefined whether the call is synchronous. A synchronous call for 10M entries risks timeouts.
  - No Service, Namespace or Secret objects are listed.
  - Nothing says who calls `run.sh`.
- **Fix:**
  - Choose one owner for prefill.
  - If the harness triggers it, call it on the pod IP or set `publishNotReadyAddresses`, make it asynchronous, and poll `/readyz`.
  - Define the request body in SPEC.md.
  - Add the Service and Secret to `sut.yaml.tmpl` and `campaign-job.yaml.tmpl`.
  - State that `campaign.py` calls `run.sh` for each run.

### implementation-readiness-r1-6: Campaign and variant file formats, resume and invalid-run rules are undefined
- **Reviewer:** egm-implementation-readiness. Also raised by egm-codebase-details (M14).
- **Location:** lines 1363-1368, 1386-1392, 1439-1444, 1458-1465.
- **Problem:**
  - There is no schema for `variants/*.yaml` or `campaigns/*.yaml`.
  - Tuning sweeps need GOGC and GOMEMLIMIT overrides per configuration. GOMEMLIMIT must be stored as a fraction, because the pod limit is known only after sizing.
  - Resume needs a recorded seed and a fixed schedule; otherwise a resumed session reshuffles.
  - The manifest lacks a campaign ID, a session ID and a run key.
  - No rule covers invalid runs: rerun or not, retry cap, how the "at least 5 blocks" requirement is kept.
  - The OOM rule (pod limit × 1.2, one rerun) and the provisional 1.5 × C6 calibration limit are missing from item 50.
- **Fix:**
  - Add `harness/schema/campaign.schema.json` and `variant.schema.json`.
  - Add a seed to each campaign file. Generate the schedule once and store it in the bucket. Resume against that schedule.
  - Add `campaign_id`, `session_id` and `run_key` to the manifest.
  - State the invalid-run and OOM policies in item 50.

### codebase-details-r1-5: The block-level hit-ratio check cannot run per run
- **Reviewer:** egm-codebase-details. Also raised by egm-style (M18).
- **Location:** lines 1423-1424, 1453-1457.
- **Problem:**
  - The spec (line 636) requires each run's hit ratio to be within 2 points of the other variants in the same block. `validate.sh` runs before the rest of the block exists.
  - "Uploads each validated run folder" reads as if invalid runs are dropped, but item 53 keeps them.
- **Fix:**
  - Split validation into per-run checks and a block-level check that `campaign.py` runs after each block.
  - Define how a later invalid mark is recorded, for example an overlay file.
  - Write "uploads every run folder after validate.sh writes its verdict, valid or not".

### implementation-readiness-r1-7: Dependency choices are left open or incomplete
- **Reviewer:** egm-implementation-readiness. Also raised by egm-codebase-details (M3, M4) and egm-style (M21).
- **Location:** lines 1264-1266, 1316, 1425-1431, 1469-1476.
- **Problem:**
  - "pandas or polars" leaves the choice open.
  - `scipy` and a YAML parser are used but not listed.
  - The HDR library, the Go Prometheus client and the Go dependencies are not named.
  - The Prometheus and Datadog Helm charts are not named or pinned.
  - Crate major versions are not pinned.
- **Fix:**
  - Pick one dataframe library.
  - Add `scipy` and `pyyaml`, pinned.
  - Name `github.com/prometheus/client_golang` and the HDR library.
  - Name and pin `prometheus-community/prometheus` (or `kube-prometheus-stack`) and the Datadog chart.
  - Pin crate majors, for example hyper 1.x.

### codebase-details-r1-6: The build and toolchain settings cannot live where the plan puts them, and some have no value
- **Reviewer:** egm-codebase-details. Also raised by egm-implementation-readiness (M7) and egm-style (M2, M5).
- **Location:** lines 1134-1137, 1165-1166, 1313-1320, 1356-1358.
- **Problem:**
  - `Cargo.toml` cannot set `CFLAGS` or rustflags.
  - Line 1165 writes `RUSTFLAGS=-C force-frame-pointers=yes`, `target-feature=+aes`. This lacks a `-C` and is broken across a line.
  - "Frame pointers or debug info" leaves a choice open, but Rust DWARF unwinding needs both.
  - No values are given for PGO, LTO, `codegen-units`, `opt-level` or `panic`.
  - "The k6 VU formula" is stuck inside the Rust bullet.
- **Fix:**
  - Add `experiments/cache/rust/.cargo/config.toml` (Create) containing `[build] rustflags = ["-C","force-frame-pointers=yes","-C","target-feature=+aes"]` and `[env] CFLAGS = "-fno-omit-frame-pointer"`.
  - In Cargo.toml set `lto = "fat"`, `codegen-units = 1`, `opt-level = 3`, `debug = "line-tables-only"`, and say no PGO.
  - Change "or" to "and" at line 1135.
  - Move the VU formula to its own bullet.

### codebase-details-r1-7: The Rust binary may not run on the runtime image
- **Reviewer:** egm-codebase-details.
- **Location:** lines 1356-1359.
- **Problem:** If the builder's glibc is newer than the glibc in `distroless/cc`, the binary fails at startup.
- **Fix:** Pin a matching pair by digest, for example `rust:<pinned>-bookworm` with `gcr.io/distroless/cc-debian12`.

### implementation-readiness-r1-8: Makefile targets call scripts that are not in the file list
- **Reviewer:** egm-implementation-readiness. Also raised by egm-codebase-details (M18).
- **Location:** lines 1103-1109 and 1432-1454.
- **Problem:** No scripts are listed for `build`, `test`, `conformance`, `local-run`, `persistent-up`, `status`, `analyze` or `charts`. What `status` prints is not defined.
- **Fix:** List each script as Create, or say which targets run commands inline. Define the output of `status`.

### implementation-readiness-r1-9: Files the spec and checkpoints need are missing
- **Reviewer:** egm-implementation-readiness. Also raised by egm-codebase-details (M22) and egm-style (M21).
- **Location:** lines 1396-1398, 1500-1503, 1527-1531; spec lines 185 and 441-445.
- **Problem:**
  - C3 runs conformance against a stub, but no stub is listed.
  - The spec requires allocation, contended-mutex and drop-heavy smoke loops in each language. The plan lists no code for them and verifies only the allocation loop.
  - No file produces page-fault counts.
  - The local C6 profile check runs on macOS with no sampler in `compose.yaml`.
- **Fix:**
  - Add `experiments/cache/conformance/stub/`.
  - Add `experiments/cache/go/cmd/smoke/main.go` and `experiments/cache/rust/src/bin/smoke.rs`, each with the three loops, and check all three in the Mechanism bullet.
  - Have `collect.sh` record page faults.
  - Either add a privileged sampler service to compose, with a C1 check, or move the profile checks to the cloud smoke test.

### implementation-readiness-r1-10: Node-level work and sampler control have no implementing detail
- **Reviewer:** egm-implementation-readiness. Also raised by egm-codebase-details (M11, N10) and egm-style (M17).
- **Location:** lines 1151, 1413-1424, 1452-1463.
- **Problem:**
  - Nothing says how the Job starts and stops perf, or how perf finds the cache PID (`hostPID`, container ID lookup).
  - The per-language perf flags are missing: 99 Hz, frame pointers for Go, `--call-graph dwarf` for Rust.
  - Folding is assigned to both the DaemonSet (line 1416) and `collect.sh` (line 1453).
  - No file pins transparent huge pages.
  - Nothing collects NTP offsets or host facts on the `sut` and `load` pools.
  - RBAC lists only the Deployment and the Jobs. It is missing `pods/exec`, `pods/log`, and read access to events and nodes.
- **Fix:**
  - Specify the trigger (for example `kubectl exec`) and the PID lookup by cgroup path.
  - Add the per-language perf command lines.
  - Give folding a single owner.
  - Add a THP init container that writes the chosen mode.
  - Add a node-facts DaemonSet on both pools, with `chronyc tracking`.
  - List the exact RBAC verbs.

### implementation-readiness-r1-11: Map-growth reporting, thread counts and the hash-path log are not specified
- **Reviewer:** egm-implementation-readiness. Also raised by egm-codebase-details (M16) and egm-style (N16).
- **Location:** lines 1149-1150, 1239, 1262-1263, 1297-1300, 1321-1324.
- **Problem:**
  - Go maps do not expose growth, so "map growth since prefill (must be zero)" cannot be built as written.
  - `GOMAXPROCS` and the tokio worker count have no stated source.
  - Spec line 545 requires a startup log of the hash path, which neither `main` mentions.
  - Logging the hash path in Go needs `golang.org/x/sys/cpu`, which breaks the item 17 dependency list.
- **Fix:**
  - Define growth per language. Go: entries ever exceeded the pre-size hint. Rust: `HashMap::capacity()` before and after.
  - Put `GOMAXPROCS=2` and `TOKIO_WORKER_THREADS=2` in every variant file and the manifest.
  - Add the hash-path log line to both `main`s. Either allow `x/sys/cpu` or drop the Go log line.

### implementation-readiness-r1-12: LRU and slab edge cases, the shard bound and the chunk size are undefined
- **Reviewer:** egm-implementation-readiness. Also raised by egm-style (M10, M14, N9).
- **Location:** lines 1227-1231, 1238, 1245, 1278-1283, 1335-1338.
- **Problem:**
  - "PUT may evict until `CACHE_MAX_BYTES` holds" is vague, and "shard bound" is never defined.
  - Does a replacing PUT move the key to the front?
  - Is the charge computed as new minus old?
  - What happens when a replace changes the size class?
  - The slab chunk size and the initial slot count are not given.
  - The node timestamp has no reader and no unit.
  - Node reuse in the pointer layout is not ruled out.
- **Fix:**
  - "PUT evicts LRU entries from the target shard until its charged bytes are at most the shard bound (`CACHE_MAX_BYTES` / `CACHE_SHARDS`)."
  - A replace moves the key to the front, and the charge delta is computed before evicting.
  - On a class change, free the old slot and take a new one.
  - Chunk = 64 KiB per class (or another value you state).
  - Drop the timestamp, or define it.
  - No free list for pointer-layout nodes.

### implementation-readiness-r1-13: Cost control, the reaper and secrets plumbing are incomplete
- **Reviewer:** egm-implementation-readiness. Also raised by egm-codebase-details (M12, M13) and egm-style (N2).
- **Location:** lines 1110-1114, 1121-1125, 1420-1438.
- **Problem:**
  - Nothing says where the reaper records its last run, or how `session-up` checks it (spec: within the last 24 hours).
  - `.env.example` lacks the remaining-credit variable, the Spaces key pair and a GitHub token.
  - How a session cluster, or a leftover volume or load balancer, is recognised is not defined.
  - The `ttl-hours` tag format is not given. `extend-ttl` fights Terraform over the tag.
  - Terraform state location and public-read on `raw/` are not set.
  - The cluster is not destroyed at campaign end (CUJ 1). It idles until its TTL.
- **Fix:**
  - The reaper writes `reaper/last_run` to the bucket, and `session-up` refuses to start if it is older than 24 h.
  - Add `REMAINING_CREDIT_USD`, `SPACES_ACCESS_KEY_ID`, `SPACES_SECRET_ACCESS_KEY` and `GITHUB_TOKEN` (if used).
  - Use the tag `bigcache-session` and the format `ttl-hours:<n>`, with `lifecycle ignore_changes` on tags.
  - Use local Terraform state per stack, and a public-read `raw/` prefix.
  - The Job writes a `campaign-done` marker, and the reaper destroys clusters that have the marker. Or budget the idle hours.

### implementation-readiness-r1-14: Load and support node pools have no size or count
- **Reviewer:** egm-implementation-readiness. Also raised by egm-codebase-details (N8) and egm-style (N21).
- **Location:** lines 1399-1407.
- **Problem:** The budget assumes one node per pool, but no size is given. "Sized from expected capacity" leaves the decision to the implementer.
- **Fix:**
  - Give default droplet sizes and counts as Terraform variables, for example one 4-vCPU dedicated load node and one 2-vCPU support node.
  - State the rule for adjusting them after calibration.

### implementation-readiness-r1-15: The Datadog profiling mechanism conflicts with the dependency rules
- **Reviewer:** egm-implementation-readiness.
- **Location:** lines 1264-1266, 1428-1431.
- **Problem:** The Go Continuous Profiler runs inside the process (dd-trace-go), and item 17 forbids that dependency. Rust has no in-process equivalent. "Node agent with the profiler on" therefore produces no Go profiles.
- **Fix:** State the approach. Either:
  - use Datadog's host or eBPF profiler for both languages, or
  - add a `-tags ddprof` build and a separate image used only in the profiling campaign, listed in the file list.

### implementation-readiness-r1-16: The test plan does not cover the harness or the cross-language pieces
- **Reviewer:** egm-implementation-readiness.
- **Location:** lines 1509-1537.
- **Problem:** No tests are specified for:
  - `campaign.py`: shuffle determinism, resume, and the invalid and OOM rules.
  - `validate.sh` checks.
  - `hit_ratio.py` and the k6 generation against the golden vectors.
  - Go `server.go` routing and status codes.
  - `gc_align.py`.
- **Fix:** Add named test cases, test file paths and expected outcomes for each.

### codebase-details-r1-8: The conformance setup is undefined
- **Reviewer:** egm-codebase-details. Also raised by egm-style (N10).
- **Location:** lines 1247-1256, 1519-1521.
- **Problem:**
  - The CLI takes only a base URL. Eviction, byte-bound and same-shard cases need a small `CACHE_MAX_BYTES`; prefill cases need a large one.
  - "Many keys", "sampled values" and "a pipelined, delayed keep-alive client" have no counts.
- **Fix:**
  - Specify the `CACHE_MAX_BYTES` value(s), and the restarts, that `make conformance` uses.
  - Require the hit-count replay to run serially from one client.
  - Give the counts and the pipelining pattern.

### codebase-details-r1-9: Some paths are globs or bare folders, and lock files are missing
- **Reviewer:** egm-codebase-details. Also raised by egm-implementation-readiness (N4, N9).
- **Location:** lines 1209, 1363, 1386-1392, 1488.
- **Problem:**
  - `campaigns/*.yaml` stands in for seven known files. It also omits the smoke-test and calibration campaigns.
  - `harness/analysis/tests/` names no files.
  - `docs/prompts/` has no naming pattern.
  - `go.sum`, `.terraform.lock.hcl` and `versions.tf` are not listed.
- **Fix:**
  - Name each campaign file and each test file.
  - Give a prompt naming pattern, for example `docs/prompts/<lang>-<component>.md`.
  - Mark `posts/img/` as authored output.
  - Add the lock files and `versions.tf` as Create.

### codebase-details-r1-10: There are two Go modules, but the build commands assume one
- **Reviewer:** egm-codebase-details. Also raised by egm-implementation-readiness (N1).
- **Location:** lines 1247, 1264, 1511.
- **Problem:**
  - `go build ./...` fails at a root that has no `go.mod`.
  - The module path depends on the repo home, which the spec defers to "before C5", but the modules are created at C3 and C4.
- **Fix:**
  - Add a root `go.work`, or give the commands per module.
  - Decide the module path, or a placeholder, before C3.

### codebase-details-r1-11: The plan breaks its own `unsafe` policy
- **Reviewer:** egm-codebase-details.
- **Location:** lines 1147-1148, 1347-1355.
- **Problem:** FAIRNESS says `unsafe` appears only in `lru_pointer.rs`. Three things need it elsewhere:
  - A counting `GlobalAlloc` in the tests.
  - mimalloc process info through `libmimalloc-sys` in `metrics.rs`.
  - `unsafe impl Send` for `NonNull<Node>`.
- **Fix:**
  - Allow `unsafe impl Send` in `lru_pointer.rs`.
  - Use `stats_alloc` for counting allocations.
  - Scope the policy to the cache hot path, with a named exemption for the `metrics.rs` FFI, or drop the mimalloc process info.

### codebase-details-r1-12: The plan does not match the spec in three places
- **Reviewer:** egm-codebase-details. Also raised by egm-style (M7).
- **Location:** lines 1192-1195, 1484-1486, 1170-1196.
- **Problem:**
  - (a) Cross-node results "must keep the same sign", but the plan omits the D9 factor-of-2 test (spec lines 786-788). It also does not say "same sign as what".
  - (b) Cross-node runs have 3 blocks (spec line 465), which conflicts with "at least 5 blocks".
  - (c) Calculated density needs go-default's capacity at 1M entries (spec line 715). Item 50 calibrates only at 5M and 10M.
- **Fix:**
  - Write: "Cross-node results (3 blocks) get no interval; confirmed only if the mean has the same sign as the primary node and is within a factor of 2."
  - Either add a 1M calibration and budget it, or change the density rate to one that is measured.

### codebase-details-r1-13: Required manifest fields are missing
- **Reviewer:** egm-codebase-details. Also raised by egm-implementation-readiness (N6).
- **Location:** lines 1458-1465.
- **Problem:** The spec requires these fields, and the manifest list omits them:
  - k6 `preAllocatedVUs` and `maxVUs`.
  - Worker thread count and `GOMAXPROCS`.
  - OOM resize history.
  - Campaign and session ID.
  - Latency target.
  - CFS throttled periods.
  - Overload outcome.
  - MIMALLOC settings.
  - Page faults.
  - Schedule seed and block size.
  - Missed-scrape count.
  - The hash-path log line.
- **Fix:** Add each field to the manifest list.

### style-r1-1: The rules version has three names
- **Reviewer:** egm-style.
- **Location:** lines 1132, 1167, 1533.
- **Problem:** "A single version number after", "the shared rules version" and "Rules v1 freezes here" describe one thing. The first does not say what it comes after or what changes the number.
- **Fix:** At 1132 write "holds the shared rules version: v0 at C2, frozen as v1 at the C7 smoke test, bumped on every later change". Use "rules version" everywhere.

### style-r1-2: FAIRNESS and RUN_PROTOCOL wording leaves values open
- **Reviewer:** egm-style.
- **Location:** lines 1152-1159, 1199-1200.
- **Problem:**
  - "GOMEMLIMIT at 80/90% of their pod limit" is unclear: it could mean each variant's own limit or go-default's.
  - "A cap on fix iterations" has no number.
  - The tuning sweep uses 6-minute windows, but RUN_PROTOCOL sets "at least 10 minutes".
  - "Minimum samples" has no unit or value.
- **Fix:**
  - "go-tuned and go-pointerfree use go-default's pod memory limit; GOMEMLIMIT = 80% or 90% of that limit."
  - State N for fix iterations.
  - "Measurement window at least 10 minutes (tuning sweeps: 6 minutes)."
  - Give the sample rule, for example "at least N CPU samples and N requests per run".

### style-r1-3: "Reason" and "category" name the same thing
- **Reviewer:** egm-style.
- **Location:** lines 1176, 1186.
- **Problem:** "One reason per frame" and "CPU-category precedence rule" seem to describe the same thing. The categories are not listed.
- **Fix:** Write "the frozen frame list maps each frame to exactly one category (memory management, lock wait, other)". Use "category" throughout.

### style-r1-4: The campaign-shape sentence is hard to parse and incomplete
- **Reviewer:** egm-style. Also raised by egm-codebase-details (N6).
- **Location:** lines 1197-1203.
- **Problem:** "Two crossing sweeps at a fixed rate for the entry-count sweep" is unclear. The tuning sweeps are omitted.
- **Fix:** Write "a load-level sweep at <X> entries and an entry-count sweep at one fixed absolute rate, plus the tuning sweeps and the cross-node check".

### style-r1-5: `collect.sh` is described as running in two places
- **Reviewer:** egm-style.
- **Location:** lines 1449-1454; item 2.
- **Problem:** One script is said to run both inside the cluster and on the laptop, "from the bucket".
- **Fix:** Split it into `collect-run.sh` (in-cluster, per run) and `collect.sh` (laptop, pulls from the bucket).

### style-r1-6: Go and Rust metrics are not at parity
- **Reviewer:** egm-style.
- **Location:** lines 1144, 1299, 1347-1349.
- **Problem:** Go exports "selected `runtime/metrics` values" (not listed) and per-shard lock acquisitions. Rust lists neither. This contradicts "identical request-path instrumentation".
- **Fix:**
  - Replace "selected" with "the names listed in METRICS.md".
  - Add per-shard lock acquisitions to item 33, or state that the counter is Go-only and why.

---

## Nit

### style-r1-7: The heading says Create/Modify, but every item is Create
- **Reviewer:** egm-style.
- **Location:** lines 1090, 1095.
- **Problem:** The heading says "Create/Modify", but every item is Create.
- **Fix:** Rename the heading to "Target Files to Create".

### style-r1-8: The tuning-budget bullet is too long
- **Reviewer:** egm-style.
- **Location:** lines 1157-1164.
- **Problem:** The bullet is one 70-word sentence.
- **Fix:** Split it into four sentences: grid, window and blocks, selection rule, Rust asymmetry.

### style-r1-9: The density metric has two names
- **Reviewer:** egm-style.
- **Location:** lines 1180, 1486.
- **Problem:** "Calculated pods per node" and "calculated density" name the same metric.
- **Fix:** Use one term.

### style-r1-10: The PREDICTIONS.md edit rule contradicts itself
- **Reviewer:** egm-style. Also raised by egm-codebase-details (N5).
- **Location:** lines 1204-1207.
- **Problem:** "Never edited" sits beside "an outcomes column is added later". "Rough size" is vague.
- **Fix:**
  - Either "prediction columns are never edited; only an Outcome column is appended after C7", or keep outcomes in a separate file.
  - Define size buckets, for example <10%, 10-50%, >50%.

### style-r1-11: PREDICTIONS.md's "Why" line does not explain its purpose
- **Reviewer:** egm-style.
- **Location:** lines 1207-1208.
- **Problem:** "Interview material independent of who typed the code" does not say what the file is for.
- **Fix:** "Records the expected mechanisms before results, so they can be checked against outcomes."

### style-r1-12: "Provider capabilities" is vague
- **Reviewer:** egm-style.
- **Location:** line 1220.
- **Problem:** The phrase does not say what is checked.
- **Fix:** List the checks: dedicated-CPU nodes, THP control, bucket-scoped keys.

### style-r1-13: The shard count is both a variable and a constant
- **Reviewer:** egm-style. Also raised by egm-implementation-readiness (N3) and egm-codebase-details (N2).
- **Location:** lines 1231, 1240-1241, 1274, 1336.
- **Problem:** The text hard-codes "modulo 64", but `CACHE_SHARDS` is a variable. The hash input is not named. No `fnv` crate is listed.
- **Fix:**
  - "FNV-1a 64-bit of the 16-byte key, modulo `CACHE_SHARDS` (must be 64; any other value fails at startup)."
  - Say FNV is hand-written, or add the crate.

### style-r1-14: Go linters and `.golangci.yml` location are not given
- **Reviewer:** egm-style. Also raised by egm-codebase-details (N7).
- **Location:** lines 1265-1266, 1303-1305.
- **Problem:** The "enabled linters" are not named. The location of `.golangci.yml` is not given.
- **Fix:** Name the linters and give the file location.

### style-r1-15: A "Why" line is split across lines
- **Reviewer:** egm-style.
- **Location:** lines 1271-1272.
- **Problem:** "*Why:*" and "entry point." are split across lines.
- **Fix:** Join them.

### style-r1-16: The GC claim is not precise
- **Reviewer:** egm-style.
- **Location:** line 1283.
- **Problem:** "Almost nothing for the GC to scan" is uncalibrated.
- **Fix:** "Entry records, map and slabs hold no pointers, so GC marking does not visit entries."

### style-r1-17: The allocation-test cases are not defined
- **Reviewer:** egm-style. Also raised by egm-codebase-details (N3).
- **Location:** lines 1284-1288.
- **Problem:** "Steady state" is undefined, and the PUT cases tested are not listed.
- **Fix:**
  - List each case and its count: new key, same-length replace, different-length replace, evicting PUT. For example: 0 allocations for slab, 2 for pointer.
  - Name `testing.AllocsPerRun`.
  - Shrink the test sizes under `cfg(miri)`, and pin the Miri nightly.

### style-r1-18: "Same image pattern as Rust" is loose
- **Reviewer:** egm-style.
- **Location:** line 1306.
- **Problem:** Go uses distroless/static and Rust uses distroless/cc, so the patterns are not the same.
- **Fix:** "Same multi-stage distroless pattern as Rust."

### style-r1-19: "A small trait" is vague and dispatch is unspecified
- **Reviewer:** egm-style. Also raised by egm-implementation-readiness (M4).
- **Location:** line 1325.
- **Problem:** "A small trait" does not say whether dispatch is static or dynamic, and dispatch affects hot-path parity with Go.
- **Fix:**
  - State dynamic dispatch, to match the Go interface.
  - Add `internal/lru/cache.go` (the Go interface) to the file list.

### style-r1-20: The musl `memcpy` claim is unsupported
- **Reviewer:** egm-style.
- **Location:** lines 1358-1359.
- **Problem:** "musl's slower `memcpy` family" is an unsupported performance claim.
- **Fix:** Cite a source, or reword to "glibc, to avoid musl allocator and `memcpy` differences".

### style-r1-21: The sizing data file description is unclear
- **Reviewer:** egm-style.
- **Location:** lines 1366-1368.
- **Problem:** It is unclear what the data file holds. "The manifest" could mean the run manifest or the Kubernetes manifest.
- **Fix:** Write "the sizing-table row the requests and limits come from" and "the run manifest embeds this file".

### style-r1-22: Compose keys and the gctrace setting are wrong or unnamed
- **Reviewer:** egm-style.
- **Location:** lines 1396-1397, 1411.
- **Problem:** `--cpus` and `--memory` are `docker run` flags, not compose keys. "gctrace to stderr" names no setting.
- **Fix:** Use the compose keys `cpus` and `mem_limit`. Write `GODEBUG=gctrace=1`.

### style-r1-23: Small gaps in the output, results and verification items
- **Reviewer:** egm-style.
- **Location:** lines 1497, 1504, 1531-1533.
- **Problem:**
  - The `<timestamp>` format is not given.
  - HOTPATH_NOTES (C4) is filed under "Results and posts (C6-C11)".
  - The smoke-run length is not given.
- **Fix:**
  - Use UTC `YYYYMMDDTHHMMSSZ`.
  - Move HOTPATH_NOTES to the C4 group.
  - Give the smoke-run duration.

### implementation-readiness-r1-17: The node CPU metric for the k6 70% check is not named
- **Reviewer:** egm-implementation-readiness.
- **Location:** lines 1188-1189.
- **Problem:** The check does not say which metric measures node CPU.
- **Fix:** Use the cAdvisor root cgroup (`id="/"`), or add node-exporter to the Prometheus values.

### implementation-readiness-r1-18: A spec sentence disagrees with the plan (spec-side fix)
- **Reviewer:** egm-implementation-readiness.
- **Location:** spec line 383 against plan line 1427.
- **Problem:** The spec says Prometheus scrapes k6. The plan uses remote-write, which matches spec component 3.
- **Fix:** Fix the spec sentence.

### codebase-details-r1-14: The cAdvisor refresh is slower than the scrape
- **Reviewer:** egm-codebase-details.
- **Location:** line 1427.
- **Problem:** cAdvisor refreshes about every 10 s, so a 2-second scrape repeats values and can miss the peak working set used for sizing.
- **Fix:**
  - Verify the refresh interval at C1.
  - Consider reading cgroup v2 `memory.peak` for the sizing pre-run.

### codebase-details-r1-15: GC alignment has two owners, and attribution outputs are missing
- **Reviewer:** egm-codebase-details. Also raised by egm-style (M21).
- **Location:** lines 1476-1483.
- **Problem:**
  - Both `analyze.py` and `gc_align.py` compute alignment.
  - `cpu_share.py` omits the truncated-stack share and page faults (spec lines 444-445).
- **Fix:**
  - State that `analyze.py` calls `gc_align.py`.
  - Consider reusing `kubectl logs --timestamps` for gctrace times.
  - Add the missing outputs to `cpu_share.py`.

### codebase-details-r1-16: FAIRNESS omits two factors
- **Reviewer:** egm-codebase-details.
- **Location:** lines 1132-1168.
- **Problem:** The 1.2 CPU density factor and the 1.2 OOM resize factor are not listed.
- **Fix:** List both in FAIRNESS.

### codebase-details-r1-17: The spec contradicts itself on cost (spec-only, outside the plan)
- **Reviewer:** egm-codebase-details.
- **Location:** spec line 812 against line 692.
- **Problem:** D12 says go-pointerfree's tuning sweep costs about 10 cluster-hours, but the Budget table says 4. Item 40 promises campaign headers that match the Budget.
- **Fix:** Resolve the spec before writing campaign files.
