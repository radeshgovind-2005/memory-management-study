# Goldfish Annotated Review Report (Stage 9, round 2)

Status: resolved (2026-11-24). Applied: all 60 PROPOSE FIX items (author: auto accept), by revising the Detailed Implementation Plan and correcting the spec (no record timestamps, block as pairing unit, rates in iterations per second, prediction-column wording). Contested: 0. Escalated: 0. Declined: 0.

**Target:** `docs/designs/big_cache.md` (lines 1091-1705, plus spec-side mismatches)
**Source findings:** `implementation_readiness_report_round2.md`
**Totals:** 1 Blocker, 28 Major, 31 Nit. Suggestions: 60 PROPOSE FIX, 0 CONTEST, 0 ESCALATE, 0 DECLINE.

Every finding is valid. Where a reviewer offered options, the choice taken is stated. Unless noted, the fix is the one given in the source report.

## Choices taken where the reviewers left options

| Finding | Choice |
| --- | --- |
| r2-1 GET/PUT semantics | `Get(key, dst) (n, ok)` copies into a caller-owned, per-connection buffer while holding the shard lock, in all four layouts (Go pointer layout copies too, so every layout does the same work); `Put(key, val)` copies `val`. GET hit and miss = 0 allocations in every layout. |
| r2-2 histograms | Two preallocated live histograms swapped each second; closed slot encoded (V2 compressed, base64) then reset; at most 1,200 encoded slots kept; maximum trackable value lowered to 10 s. |
| r2-3 response end | Go: wrapped `net.Listener`; handler sets `Content-Length`, writes once, calls `http.NewResponseController(w).Flush()`, end stamp after `Flush` returns. Rust: `http1::Builder::pipeline_flush(false)`; per-connection FIFO of expected response sizes; end stamped in `poll_write` at each boundary. Sample goes in the slot of its end second. Only `/v1/entries/*` is recorded. |
| r2-4 workload randomness | splitmix64 everywhere; `u = (x >> 11) × 2^-53`; one draw for the op (`u < 0.9` → GET), one for the Zipf rank; ranks 1..K → index r−1, K = floor(1.25 × E); exponent 1.1 restated in the contract; k6 uses BigInt and seeds each VU with `WORKLOAD_SEED XOR (vu << 32) XOR iteration`; a C5 check measures k6 CPU per iteration. Replay: seed 7, 200,000 iterations, in the `large` pass; expected hit count from `hit_ratio.py`. |
| r2-5 bodies | `Content-Length` required (400 otherwise); Go `io.ReadFull` into a pooled `[300]byte`; Rust collects then copies into a fixed `[u8; 300]`; one write with explicit `Content-Length`; both in FAIRNESS. |
| r2-6 calibration p99 | k6 `--out csv`; `campaign.py` computes exact 10-second p99s from raw samples; one fresh pod per calibration ramp with a 60-second warm-up before the first step. |
| r2-10 THP | `madvise` for `enabled` and `defrag`. |
| codebase-r2-1 seed/entry count | The prefill body's seed is the only source (`PREFILL_SEED` only supplies its default). Configured entry count E sets `CACHE_MAX_BYTES = E × 280` and prefill N = E; resident entry count is what `hit_ratio.py` reports. |
| codebase-r2-2 timestamps | No timestamp in any record; the spec text is corrected. Record fields listed once in the contract. |
| codebase-r2-6 fills | Tag key `op`; thresholds on `{scenario:measure,op:get}` and `{scenario:measure,op:put}` at the latency target, recorded but not aborting; fills send `X-Op: fill`, which the server excludes from histograms. |
| codebase-r2-13 perf samples | The sample count is reported, not a validity rule. |
| style-r2-1 rates | Every rate, capacity and level is in iterations per second. `calibration.json` is written to the bucket and the calibrated rates are then committed into the campaign files by `make collect`. |
| codebase-r2-14 long sweeps | The block is the pairing unit; each block records its node; Goal 2 is reworded from "same session" to "same node within a block". |
| style-r2-6 repeated blocks | Repeat once; if still invalid, drop the block, flag the configuration, continue. |

## All findings

| Id | Suggestion |
| --- | --- |
| egm-implementation-readiness-r2-1 (Blocker) | PROPOSE FIX (choice above) |
| egm-implementation-readiness-r2-2 … r2-10 | PROPOSE FIX (choices above) |
| egm-codebase-details-r2-1 … r2-13 | PROPOSE FIX (choices above where listed; otherwise the source report's fix) |
| egm-style-r2-1 … r2-6 | PROPOSE FIX |
| All 31 Nits (implementation-readiness r2-11 … r2-16, codebase-details r2-14 … r2-20, style r2-7 … r2-24) | PROPOSE FIX, as in the source report |
