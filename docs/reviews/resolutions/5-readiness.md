# Goldfish Annotated Review Report (Stage 9, Implementation Readiness)

Status: resolved (2026-11-24). Applied: all 57 PROPOSE FIX items (author: auto accept), by rewriting the Detailed Implementation Plan around a new Cross-language contract subsection, plus spec-side fixes (D12 cost, k6 remote write, density rate at 10M, memory.peak). Contested: 0. Escalated: 0.

**Target:** `docs/designs/big_cache.md` (lines 1088-1543, DetailedRange; two spec-side items)
**Source findings:** `implementation_readiness_report.md`
**Totals:** 4 Blocker, 31 Major, 25 Nit. Suggestions: 57 PROPOSE FIX, 0 CONTEST, 0 ESCALATE, 3 DECLINE.

Approach: every decision an implementer would otherwise have to make is decided here, in one new **Cross-language contract** subsection at the top of the implementation plan, so the file entries can point to it instead of repeating it. Byte-level layouts and full JSON schemas still live in `SPEC.md` (written at C3), but every choice that affects fairness or correctness is fixed now.

## Blockers — PROPOSE FIX

| Id | Proposed edit |
| --- | --- |
| implementation-readiness-r1-1 | New contract subsection: key = 32 lowercase hex chars in the URL path (400 otherwise); PUT body = raw octets, `application/octet-stream`, 100–300 bytes (400 otherwise); status codes 200 / 204 / 404 / 400; `POST /admin/prefill` takes `{"count": N, "seed": S}` and returns 202 (asynchronous); field lists for `/admin/stats` and `/admin/histograms`; `/metrics` names listed in METRICS.md; `PORT` default 8080. Histograms: HdrHistogram V2 compressed + base64, ns, 1 ns–60 s, 3 significant digits. Recording: one mutex-protected histogram per one-second slot in both languages, swapped by a ticker (in FAIRNESS). Libraries: `github.com/HdrHistogram/hdrhistogram-go`, Rust `hdrhistogram`, Python `hdrh`. |
| implementation-readiness-r1-2 | Generator: splitmix64 seeded with `PREFILL_SEED ^ i`; key = first 16 output bytes; value length = 100 + (next output mod 201); value bytes = continued output. Zipf rank r → key index r (rejection-inversion sampler, no table). Every PUT sends `value(i)`. Mix: 90% GETs and 10% independent PUTs; each GET miss also issues a cache-aside PUT, not counted in the 10% and excluded from latency percentiles (tagged). Prefill inserts keys 0..N−1 and stops at N; the expected entry count after prefill comes from `hit_ratio.py`. New file `experiments/cache/testdata/golden.json` checked by Go tests, Rust tests, `hit_ratio.py` and a k6 unit check. |
| implementation-readiness-r1-3 | Add `harness/images/campaign/Dockerfile` (pinned Python, kubectl, aws CLI for Spaces, `inferno`), `harness/campaign/pyproject.toml`, `harness/images/sampler/Dockerfile` (perf from the node's distribution; fallback: host perf through `nsenter`, decided at C1). `make build` produces four images. |
| codebase-details-r1-1 | Add `experiments/cache/rust/src/lib.rs` declaring all modules; `main.rs` becomes a thin binary holding the mimalloc `#[global_allocator]`, so tests and Miri do not load C mimalloc. |

## Major — PROPOSE FIX

| Id | Proposed edit |
| --- | --- |
| implementation-readiness-r1-4 | One stamping state machine (in contract + FAIRNESS): start = first read returning bytes after accept or after the previous response's last byte was written; for a pipelined request already buffered, start = previous response's last write; end = last response byte accepted by the wrapped connection's write. Go: wrapped `net.Conn` passed through `http.Server.ConnContext`; Rust: wrapper implementing `AsyncRead` and `AsyncWrite` under `hyper_util::rt::TokioIo`, sharing an `Arc<AtomicU64>`. Conformance checks timing values. |
| implementation-readiness-r1-5 | Calibration step control moves to `campaign.py` (one short k6 run per step, judged from k6's per-second output). Rate = iterations per second; request rate = iterations × (1 + miss ratio × 0.9). First step 500 it/s. Pass = worst 10-second p99 within 4× the low-load value and no dropped iterations. `campaign.py` runs the per-variant checks and writes `calibration.json` to the bucket; campaign files express levels as percentages and manifests record absolute rates. |
| codebase-details-r1-2 | `steady.js`: rejection-inversion Zipf; request tags (`get`, `put`, `fill`); percentiles over `get` and `put` only; threshold on `http_req_duration{scenario:measure}`; `summaryTrendStats` with p(99), p(99.9); `handleSummary` output; `experimental-prometheus-rw` with `K6_PROMETHEUS_RW_PUSH_INTERVAL=1s`; Prometheus `--web.enable-remote-write-receiver`; "1.25 × entry count". |
| codebase-details-r1-3 | One k6 run per measurement: warm-up 3 minutes, then a one-iteration scenario that POSTs `/admin/reset`, then the measurement scenario; `run.sh` starts perf at the same offset. |
| codebase-details-r1-4 | Harness triggers prefill on the pod IP, asynchronously, and polls `/readyz`. Service, Namespace and Secret objects added to the templates. `campaign.py` calls `run.sh` per run. |
| implementation-readiness-r1-6 | Add `harness/schema/campaign.schema.json` and `variant.schema.json`; variant files hold GOMEMLIMIT as a fraction; each campaign file has a seed; the schedule is generated once and stored in the bucket; manifests gain `campaign_id`, `session_id`, `run_key`. Invalid runs: rerun once at the end of the block; a block with a variant still invalid is repeated; OOM rule as in the spec. |
| codebase-details-r1-5 | Per-run checks in `validate.sh`; block-level hit-ratio check in `campaign.py` after each block, written as `block_verdict.json`. Every run folder is uploaded after its verdict, valid or not. |
| implementation-readiness-r1-7 | polars; add `scipy`, `pyyaml`; name `prometheus/client_golang`; Helm charts `prometheus-community/prometheus` and `datadog/datadog`, pinned; crate majors pinned (hyper 1.x, tokio 1.x). |
| codebase-details-r1-6 | Add `experiments/cache/rust/.cargo/config.toml` (rustflags `-C force-frame-pointers=yes -C target-feature=+aes`, `CFLAGS=-fno-omit-frame-pointer`); Cargo profile `lto = "fat"`, `codegen-units = 1`, `opt-level = 3`, `debug = "line-tables-only"`, `panic = "abort"`; no PGO in either language; "frame pointers and debug info"; VU formula on its own bullet. |
| codebase-details-r1-7 | Pin `rust:<version>-bookworm` builder with `gcr.io/distroless/cc-debian12`, by digest. |
| implementation-readiness-r1-8 | State that `build`, `test`, `conformance`, `local-run`, `persistent-up`, `analyze` and `charts` run commands inline in the Makefile; `status` prints the active cluster, its TTL and the campaign's progress from the bucket. |
| implementation-readiness-r1-9 | Add `experiments/cache/conformance/stub/` (an in-memory reference implementation), `experiments/cache/go/cmd/smoke/main.go` and `experiments/cache/rust/src/bin/smoke.rs` (allocation, contended-mutex, drop-heavy loops), page faults in `collect-run.sh`; profile checks move from C6 (local, macOS) to the cloud smoke test. |
| implementation-readiness-r1-10 | `run.sh` starts and stops perf with `kubectl exec` into the sampler pod; the sampler runs with `hostPID` and finds the cache PID by cgroup path. Commands: Go `perf record -F 99 -g`; Rust `perf record -F 99 --call-graph dwarf`. Folding owned by `collect-run.sh` only. THP set by a privileged init container on the sampler DaemonSet. A `node-facts` DaemonSet on `sut` and `load` records `chronyc tracking` and host facts. RBAC verbs listed. |
| implementation-readiness-r1-11 | Map growth: Go reports entries ever exceeding the pre-size hint; Rust compares `HashMap::capacity()` before and after. `GOMAXPROCS=2` and `TOKIO_WORKER_THREADS=2` in every variant file. Hash-path log line in both `main`s; Go uses `golang.org/x/sys/cpu`, added to the allowed dependencies. |
| implementation-readiness-r1-12 | PUT evicts LRU entries from its shard until charged bytes ≤ shard bound (`CACHE_MAX_BYTES / CACHE_SHARDS`); a replace moves the key to the front and the charge delta is applied before evicting; a size-class change frees the old slot and takes a new one; slab chunk = 64 KiB per class; the per-entry timestamp is removed; no free list for pointer-layout nodes. |
| implementation-readiness-r1-13 | Reaper writes `reaper/last_run` to the bucket; `session-up` refuses if older than 24 h. `.env.example` gains `REMAINING_CREDIT_USD`, `SPACES_ACCESS_KEY_ID`, `SPACES_SECRET_ACCESS_KEY`. Clusters tagged `bigcache-session` and `ttl-hours:<n>`, Terraform `ignore_changes` on tags. Local Terraform state per stack (git-ignored). `raw/` prefix public-read. The Job writes `campaign-done` to the bucket and the reaper also destroys clusters whose campaign is done. |
| implementation-readiness-r1-14 | Terraform variables with defaults: `load` = one 4-vCPU dedicated node, `support` = one 2-vCPU node; `campaign.py` warns if k6 CPU exceeds 60% during calibration, so the next session adds a load node. |
| implementation-readiness-r1-15 | Profiling campaign uses separate profiling images: Go built with `-tags ddprof` (dd-trace-go profiler, only in that image); Rust run under Datadog's native `ddprof` wrapper **(needs verification at C1)**. Both images added to the file list; measurement images stay free of Datadog code. |
| implementation-readiness-r1-16 | Named tests: `harness/campaign/tests/test_schedule.py` (shuffle determinism, resume, invalid and OOM rules), `harness/scripts/tests/validate.bats`, golden-vector tests for `hit_ratio.py` and k6, `server_test.go` (routing and status codes), `tests/test_gc_align.py`. |
| codebase-details-r1-8 | `make conformance` runs two passes: small (`CACHE_MAX_BYTES` = 64 KiB) for eviction, byte-bound and same-shard cases; large (prefill 100,000 entries) for prefill and replay. Replay runs serially from one client. Counts: 1,000 same-shard keys, 100 sampled values, a pipelining client sending 3 requests per write with 50 ms delays. |
| codebase-details-r1-9 | Name the campaign files (`tuning-go-tuned.yaml`, `tuning-go-pointerfree.yaml`, `calibration.yaml`, `smoke.yaml`, `load-level.yaml`, `entry-count.yaml`, `sampler-overhead.yaml`, `cross-node.yaml`, `datadog-profiling.yaml`) and test files; prompts as `docs/prompts/<lang>-<component>.md`; add `go.sum`, `.terraform.lock.hcl`, `versions.tf`. |
| codebase-details-r1-10 | Root `go.work` covering both Go modules; module path placeholder `github.com/<owner>/bigcache/...`, finalised with the repository home before C3. |
| codebase-details-r1-11 | Policy scoped: `unsafe` allowed only in `lru_pointer.rs` (including `unsafe impl Send`); allocation counting uses the `stats_alloc` crate; `metrics.rs` gets one named FFI exemption for mimalloc process info. |
| codebase-details-r1-12 | (a) and (b): "Cross-node results (3 blocks) get no interval; confirmed only if the mean has the same sign as the primary node and is within a factor of 2." (c) Density per-replica rate becomes 50% of go-default's capacity at 10M entries (already calibrated). |
| codebase-details-r1-13 | Manifest list gains: VUs, `GOMAXPROCS`, worker threads, OOM resize history, campaign and session IDs, latency target, CFS throttled periods, overload outcome, MIMALLOC settings, page faults, schedule seed and block size, missed scrapes, hash-path log line. |
| style-r1-1 | "rules version" everywhere, defined once at item 7. |
| style-r1-2 | GOMEMLIMIT = 80% or 90% of each tuned variant's own pod limit; fix-iteration cap 3; "at least 10 minutes (tuning sweeps: 6 minutes)"; minimum 10^5 requests and 50,000 CPU samples per run. |
| style-r1-3 | "The frozen frame list maps each frame to exactly one category (memory management, lock wait, other)." |
| style-r1-4 | Campaign shape restated with all four parts. |
| style-r1-5 | Split into `collect-run.sh` (in-cluster) and `collect.sh` (laptop). |
| style-r1-6 | "the names listed in METRICS.md"; per-shard lock acquisitions added to Rust `metrics.rs`. |

## Nit

| Id | Suggestion | Edit or reason |
| --- | --- | --- |
| style-r1-7 | PROPOSE FIX | Heading "Target Files to Create". |
| style-r1-8 | PROPOSE FIX | Split the tuning bullet. |
| style-r1-9 | PROPOSE FIX | "calculated pods per node" only. |
| style-r1-10 | PROPOSE FIX | "Prediction columns are never edited; only an Outcome column is appended after C7"; size buckets <10%, 10–50%, >50%. |
| style-r1-11 | PROPOSE FIX | New "Why" wording. |
| style-r1-12 | PROPOSE FIX | List the provider checks. |
| style-r1-13 | PROPOSE FIX | `CACHE_SHARDS` must be 64; FNV-1a hand-written (a few lines) in both languages. |
| style-r1-14 | PROPOSE FIX | `.golangci.yml` at `experiments/cache/go/`; linters `govet`, `staticcheck`, `errcheck`, `ineffassign`, `unused`. |
| style-r1-15 | PROPOSE FIX | Join the line. |
| style-r1-16 | PROPOSE FIX | Precise GC wording. |
| style-r1-17 | PROPOSE FIX | Allocation cases listed with counts; `testing.AllocsPerRun`; smaller sizes under `cfg(miri)`; Miri nightly pinned. |
| style-r1-18 | PROPOSE FIX | "Same multi-stage distroless pattern as Rust." |
| style-r1-19 | PROPOSE FIX | Dynamic dispatch in both (Go interface, Rust `dyn` trait); add `internal/lru/cache.go`. |
| style-r1-20 | PROPOSE FIX | Reword with the existing "needs verification" marker. |
| style-r1-21 | PROPOSE FIX | Clarify the variant file description. |
| style-r1-22 | PROPOSE FIX | Compose keys `cpus`, `mem_limit`; `GODEBUG=gctrace=1`. |
| style-r1-23 | PROPOSE FIX | UTC `YYYYMMDDTHHMMSSZ`; move HOTPATH_NOTES to C4; smoke run 5 minutes. |
| implementation-readiness-r1-17 | PROPOSE FIX | cAdvisor root cgroup (`id="/"`) for node CPU. |
| implementation-readiness-r1-18 | PROPOSE FIX | Spec sentence: k6 remote-writes to Prometheus. |
| codebase-details-r1-14 | PROPOSE FIX | Sizing pre-run reads cgroup v2 `memory.peak`; cAdvisor interval checked at C1. |
| codebase-details-r1-15 | PROPOSE FIX | `analyze.py` calls `gc_align.py`; `cpu_share.py` outputs truncated-stack share and page faults. |
| codebase-details-r1-16 | PROPOSE FIX | List the 1.2 factors in FAIRNESS. |
| codebase-details-r1-17 | PROPOSE FIX | D12 cost corrected to about 4 cluster-hours. |
| (none) | DECLINE | — |

Declined (3): none of the Nits above; the three declines are recorded for traceability only where a finding duplicated a merged one (implementation-readiness N1, N3, N4 are covered by codebase-details-r1-10, style-r1-13 and codebase-details-r1-9).
