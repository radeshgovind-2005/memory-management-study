# Goldfish Annotated Review Report (Stage 7, round 2)

Status: resolved (2026-11-24). Applied: 41 PROPOSE FIX items (author: auto accept). Declined: 7 Nits. Contested: 0. Escalated: 0.

**Target:** `docs/designs/big_cache.md` (lines 1-1072, FeatureRange)
**Source findings:** `critic_report_round2.md`
**Totals:** 20 Major, 28 Nit. Suggestions: 20 Major PROPOSE FIX; 21 Nit PROPOSE FIX; 7 Nit DECLINE; 0 CONTEST; 0 ESCALATE.
**Author decision taken during resolution:** go-pointerfree gets its own tuning sweep (design-r2-4, option A).

Approach: fix every validity and contradiction finding; fix one-line wording nits where the fix does not add length; decline nits that would add text or only restyle.

## Major — all PROPOSE FIX

| Id | Proposed edit |
| --- | --- |
| design-r2-1 | Latency target frozen at C6 from local runs only. Smoke-test amendments limited to operational items (timeouts, scrape config, sizing factors). The smoke test runs one variant per language and reports no per-variant comparison. Goal 6, D10 and the Latency target section reworded to match. |
| design-r2-2 | C6 exit: "alignment measure computed and reported for local `go-default` runs; if there is no alignment, the cause is documented and a go/no-go decision recorded". |
| design-r2-3 | Tuning sweeps run in shuffled blocks (3 blocks, all 12 settings per block). A tie is any setting whose mean p99.9 is within 5% of the best; among ties, lowest memory-management CPU share wins. D12 and component 10 aligned. |
| design-r2-4 | **Author decision A:** go-pointerfree gets its own 12-setting tuning sweep with the D12 rule. Variant table, sizing table and Budget updated (+36 runs, ~10 cluster-hours, in the minimum campaign). |
| design-r2-5 | Sign test becomes descriptive ("k of n blocks agree in sign"); its minimum possible p is stated; the t interval is the decision rule. |
| design-r2-6 | Alignment measure defined: server-side per-second HDR series; spike = per-second p99.9 above 3 × the run's median per-second p99.9; seconds with fewer than 1,000 requests excluded; 100 circular random shifts; statistic = observed overlap share minus mean shifted share. k6 per-second claim marked "needs verification" with the push interval set explicitly. |
| design-r2-7 | k6 VUs fixed by formula: `preAllocatedVUs = ceil(rate × latency target × 4)`, `maxVUs = 2 ×` that, the same for every variant at a level; values recorded in the manifest. |
| design-r2-8 | Sizing pre-run lasts 5 minutes (at least two forced-GC intervals) at the highest level the session uses (85%). OOM at 110% recorded as outcome "killed"; OOM at 25–85% invalidates the run and triggers one resize (limit × 1.2) and rerun, recorded in the manifest. |
| design-r2-9 | `RUSTFLAGS=-C force-frame-pointers=yes`; `perf --call-graph dwarf` for Rust (standard library has no frame pointers). Smoke checks added for a contended-mutex loop and a drop-heavy loop per language. Truncated-stack share reported per run. Category precedence: lock wait > memory management > other. |
| design-r2-10 | Rust built for glibc (`x86_64-unknown-linux-gnu`) on `gcr.io/distroless/cc`, removing musl's slower `memcpy` as a confound. Alternative L musl/glibc claim marked "needs verification". |
| structural-r2-1 | Alternative U moved after L and renamed Alternative M. |
| structural-r2-2 | Block rule states its exceptions (cross-node 3 blocks, tuning 3 blocks, ramp 3 runs). |
| structural-r2-3 | C1 exit covers Open Questions marked "Answered at C1"; Cloud provider marked so; Proxy framing and Repository home marked as outside C1. |
| style-r2-3 | D9 defines the test once: same sign, and each node's mean paired difference within a factor of 2 of the main-sweep value. Other mentions refer to D9. |
| style-r2-4 | Static CPU manager wording fixed; hyperthread note cross-referenced. |
| style-r2-5 | "identical within each language and following the same stamping rule in both". |
| style-r2-6 | Client–server gap described as time outside the server's stamped span. |
| style-r2-7 | Rust build enables `target-feature=+aes`, checked at C4 by a startup log; claim marked "needs verification". |
| style-r2-8 | D4 rationale reworded with markers. |
| style-r2-9 | Layout names introduced before the rung list; only "pointer layout" / "slab layout" used after. |

## Nit

| Id | Suggestion | Edit or reason |
| --- | --- | --- |
| design-r2-11 | PROPOSE FIX | Calibration and sizing recomputed at about 75 minutes per session and entry count (~15 cluster-hours). |
| design-r2-12 | PROPOSE FIX | Pre-size = shard bound ÷ minimum charge (180 bytes); `/admin/stats` reports map growth after prefill (must be zero). |
| design-r2-13 | PROPOSE FIX | Ramp: starts from prefill to 50% of the pod limit; writes at a fixed bytes/s rate in RUN_PROTOCOL; reads at 50% of capacity; CPU-share signal over 10-second windows. |
| design-r2-14 | PROPOSE FIX | Reaper every 15 minutes; `session-up` refuses to start if the reaper has not run in the last 24 hours; reaper also deletes leftover volumes and load balancers. |
| design-r2-15 | PROPOSE FIX | Page-fault counts reported per run; "different VM" wording; load-pool memory sized for k6 trend storage. |
| structural-r2-4 | PROPOSE FIX | Cross-references name "NFR: Cost control", "NFR: Secrets", "Sizing table notes". |
| structural-r2-5 | PROPOSE FIX | Datadog profiling added to the end of the follow-on order. |
| structural-r2-6 | PROPOSE FIX | D2 title names its choice. |
| structural-r2-7 | DECLINE | Diagram alignment is cosmetic; it renders in a code block either way. |
| structural-r2-8 | DECLINE | The Layout column was added at the comprehension reviewer's request so readers do not have to map values to names. |
| structural-r2-9 | PROPOSE FIX | Remove the divider before Non-Functional Requirements. |
| style-r2-10 | PROPOSE FIX | "contain no pointers, so the garbage collector does not scan them". |
| style-r2-11 | PROPOSE FIX | "typically under 1 ms (to be confirmed at C1)". |
| style-r2-12 | PROPOSE FIX | "The remaining gap is published whatever its size." |
| style-r2-13 | PROPOSE FIX | "reports the CPU share spent on GC marking". |
| style-r2-14 | PROPOSE FIX | "checked by packing". |
| style-r2-15 | DECLINE | Heading wording preference; no change in meaning. |
| style-r2-16 | PROPOSE FIX | "Does it produce symbolized CPU profiles for the Rust binary?" |
| style-r2-17 | PROPOSE FIX | Single hedge with the marker. |
| style-r2-19 | DECLINE | Rationale already adequate; would only lengthen. |
| style-r2-20 | DECLINE | Splitting adds lines without changing content. |
| style-r2-21 | PROPOSE FIX | "For latency, Prometheus carries only k6's per-second series, used for live viewing and GC alignment." |
| style-r2-22 | PROPOSE FIX | "They include queueing time, such as waiting while GC workers hold the CPU." |
| style-r2-23 | PROPOSE FIX | "variant minus baseline variant". |
| style-r2-24 | PROPOSE FIX | One restart-rule wording. |
| style-r2-26 | DECLINE | Slowdown factor repetition is short; removing it would force readers to jump between sections. |
| style-r2-27 | PROPOSE FIX | "are expected to make". |
| style-r2-31 | PROPOSE FIX | "If a change alters code or settings used by only some variants, the author reruns those variants." |
