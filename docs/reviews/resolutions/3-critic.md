# Goldfish Annotated Review Report

Status: resolved (2026-11-24). Applied: 47 PROPOSE FIX items (author accepted all). Declined: 7 Nits. Contested: 0. Escalated: 0.

**Target:** `docs/designs/big_cache.md` (lines 1-993, FeatureRange)
**Source findings:** `critic_report.md` (Stage 7, round 1: egm-design, egm-structural, egm-style)
**Totals:** 26 Major, 28 Nit. Suggestions: 47 PROPOSE FIX, 0 CONTEST, 0 ESCALATE, 7 DECLINE.

Approach: correctness and validity problems get real fixes. Style and structure findings are fixed by tightening or removing text where possible, not by adding more.

## Prioritized Findings & AI Suggestions

### Major

#### design-r1-6 — No Risks and Mitigations section
* **AI Suggestion:** PROPOSE FIX
* **Justification:** Valid. The standard requires it, and several risks (credit exhaustion, privileged perf pod, provider refusing perf, host variation, unsafe Rust) have no stated mitigation.
* **Proposed Edit:** Add `## Risks and Mitigations` after Alternatives Considered: one table (Risk, Type, Mitigation, Where enforced) with about 12 rows, each pointing to an existing D-number, checkpoint, Goal or NFR. No new mechanisms except where a risk had none.

#### design-r1-1 — Sizing needs a capacity not yet measured
* **AI Suggestion:** PROPOSE FIX
* **Justification:** Valid ordering bug.
* **Proposed Edit:** Run protocol becomes: 1 Calibrate (go-default with a generous provisional limit: 1.5 × the C6 local estimate), 2 Size (each variant at 70% of the calibrated rate), 3 Measure. For the entry-count sweep, sizing uses that sweep's fixed rate.

#### design-r1-2 — Calibration steps shorter than the forced-GC cycle
* **AI Suggestion:** PROPOSE FIX
* **Justification:** Valid; this would overstate Go's capacity and bias every load level in Go's favour.
* **Proposed Edit:** Coarse-then-fine ramp: coarse steps of +25% held 60 s to find the region, then fine steps of +5% held 150 s (longer than one forced-GC interval) and judged on their worst 10-second p99. Each variant's cross-check lasts 150 s. Written into rules v0.

#### design-r1-3 — The 5M point is not shared between sweeps
* **AI Suggestion:** PROPOSE FIX
* **Justification:** Valid; the rates differ.
* **Proposed Edit:** The entry-count sweep runs all four counts (including 5M) at its fixed rate. Budget row becomes 4 counts × 5 variants × 5 blocks = 100 runs.

#### design-r1-4 — Bootstrap over 3-5 blocks too narrow; D12 tie rule ambiguous
* **AI Suggestion:** PROPOSE FIX
* **Justification:** Valid statistics problem.
* **Proposed Edit:** Replace the bootstrap with a paired Student's t 95% interval on block differences (n − 1 degrees of freedom), plus an exact sign test reported alongside; minimum 5 blocks before an interval is published (3-block cross-node results are reported as direction and size only). D12 becomes: take the setting with the lowest mean p99.9; among all settings whose interval contains that mean, pick the one with the lowest memory-management CPU share.

#### design-r1-5 — Go and Rust start the latency clock at different points
* **AI Suggestion:** PROPOSE FIX
* **Justification:** Valid fairness gap.
* **Proposed Edit:** Both stamp at the I/O layer: Go via the wrapped `net.Conn`, Rust via an `AsyncRead` shim around `TcpStream`. The stamp is the moment a read returns the first bytes after the previous response was written (Go's background 1-byte read counts as that read). Add a conformance case with a pipelined, delayed keep-alive client.

#### design-r1-7 — Node memory may not fit go-default at 10M
* **AI Suggestion:** PROPOSE FIX
* **Justification:** Valid; node memory is never stated.
* **Proposed Edit:** Name node types: main sweeps on a dedicated-CPU 4-vCPU / 16 GB node; 4-core check on 8-vCPU / 32 GB **(sizes and prices need verification at C1)**. Add a feasibility line: go-default at 10M is estimated at about 3 GB live heap, up to about 7 GB peak, a pod limit near 9 GB, which fits 16 GB minus system reserve. C1 checks it; if it does not fit, the sweep is capped at 5M and the post says so.

#### design-r1-8 — "Other cores do not compete" ignores hyperthreading
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** Use "vCPU" throughout; record `lscpu` topology and the pod's cpuset in every manifest; reword to "so they do not share the cache pod's vCPUs", and state shared physical cores, memory bandwidth and last-level cache as a limitation in the Risks table.

#### design-r1-9 — Ramp pre-sizing aimed above the pod limit
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** In the ramp, maps and slab arrays are pre-sized for the entry count expected at the pod limit (from main-sweep bytes per entry), not from `CACHE_MAX_BYTES`; each variant's working set right after startup is recorded in the manifest.

#### design-r1-10 — `memory.events` `high` probably never fires
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** Distress signal becomes: a nonzero `max` counter in `memory.events`, or memory PSI `some avg10` above 10%, alongside the latency and CPU-share thresholds.

#### design-r1-11 — Token-scoping claims unbacked
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** State that the full-account token never enters the cluster; the Job gets the narrowest token the provider offers, whose actual scope is a C1 check **(needs verification)**; bucket access uses separate storage keys. Large raw files are public-read so CUJ 2 readers can fetch them; egress cost is noted in the Risks table.

#### design-r1-12 — Green Tea GC not mentioned
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** Add to Open Questions (C1): which collector the pinned Go version uses by default; `PREDICTIONS.md` states which collector the predictions assume; if Green Tea is the default, the post says so and the mechanism story is re-checked at C6.

#### structural-r1-2 — D5 has no rejected alternative
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** Add "Alternative U: One Rust layout only (D5)". Reason: without a twin layout, layout effects cannot be separated from language effects.

#### structural-r1-3 — Alternative M is an open question
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** Move "Proxy framing" into Open Questions (answered before any proxy work); delete Alternative M; repoint references.

#### structural-r1-4 — Two paragraphs render inside the list above
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** Insert blank lines and outdent both paragraphs so they end the list.

#### structural-r1-5 — Additional Resources and Future Improvements nested under NFRs
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** Promote both to `##`.

#### style-r1-1 — Remaining gap credited to the memory model alone
* **AI Suggestion:** PROPOSE FIX
* **Justification:** Valid overclaim; the pairs also differ in HTTP stack, mutex, map and scheduler.
* **Proposed Edit:** Problem and D2 reworded: the remaining gap is attributed to the runtime and its memory management, together with the HTTP stack, mutex and map differences listed in FAIRNESS; lock wait is measured separately.

#### style-r1-2 — Contradiction about lock contention
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** Concurrency: "the lock pattern is identical; its cost is measured separately as lock wait". Component 8: "because mutex implementations differ".

#### style-r1-3 — "Headline" has several meanings
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** Headline metric = client-side p99.9; "headline configuration" renamed "reference configuration"; "co-headline" replaced with "reported with equal weight".

#### style-r1-4 — Hit counts "match" overstated
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** "evict the same keys for the same request sequence (checked by the conformance replay); under load, hit ratios agree within the run-protocol tolerance."

#### style-r1-5 — Goals promise checks the checkpoints make optional
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** Mark cross-node, 4-core and sampler-overhead checks "if credit allows (see Budget)"; C7 exit: "tuning and load-level sweeps committed; remaining campaigns run in Budget order until credit runs out; the post lists which ran." Sampler overhead moves into the minimum campaign (one variant per language), because the CPU-share metric depends on it.

#### style-r1-6 — "Not specific to one machine" overclaims
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** "to check whether the headline differences keep their sign and rough size on other nodes".

#### style-r1-7 — Validity thresholds say "about"
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** "k6 node CPU, averaged over 10-second windows, stays at or below 70% for the whole window", used in all three places.

#### style-r1-8 — "With room to spare" unmeasurable
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** "every variant's p99 at the 25% level is at most half the target".

#### style-r1-9 — Goal 2 is one 120-word sentence
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** Split into "Conditions" and "Metrics" bullets; point to the Run protocol for levels and outcomes.

#### style-r1-10 — Alternative P assumes the result
* **AI Suggestion:** PROPOSE FIX
* **Proposed Edit:** "without an overload level, behaviour beyond capacity is never measured."

### Nit

| Id | Suggestion | Justification / edit |
| --- | --- | --- |
| design-r1-13 | PROPOSE FIX | Pre-register the alignment measure: share of p99.9-spike seconds overlapping a GC cycle vs a time-shifted baseline, across all go-default runs. |
| design-r1-14 | PROPOSE FIX | State as a limitation that go-tuned is tuned once (5M, 70%, 2 vCPU). |
| design-r1-15 | PROPOSE FIX | Correct the Budget (ramp duration, calibration overhead with the new ramp, node price note, totals, "3 repetitions"). |
| design-r1-16 | PROPOSE FIX | Load pool sized so k6 stays at or under 70% CPU; density packing adds k6 pods per replica group. |
| design-r1-17 | PROPOSE FIX | Manifest records THP mode, kernel and cgroup version; THP mode pinned in FAIRNESS. |
| design-r1-18 | PROPOSE FIX | The cache-aside PUT counts as a request in CPU per 1,000 requests. |
| structural-r1-6 | DECLINE | egm-design judged the Alternatives Rule met; the deciding trade-off is in each Cons line. Adding clauses would lengthen the document further. |
| structural-r1-7 | DECLINE | The checkpoint table was placed near Goals at the previous review's request so readers see the order of work early. |
| structural-r1-8 | DECLINE | Headings follow the EGM design-doc template ("The Problem", "Technical Plan (Architecture)"). |
| structural-r1-9 | DECLINE | Cosmetic rewrap; does not affect rendering. |
| structural-r1-10 | PROPOSE FIX | Renumbered as part of design-r1-1. |
| structural-r1-11 | PROPOSE FIX | "All four values". |
| structural-r1-12 | DECLINE | Short names are already consistent after first use; file paths appear in the implementation plan. |
| structural-r1-13 | PROPOSE FIX | Cite D2 only. |
| structural-r1-14 | PROPOSE FIX | Delete the finding-code and process sentence. |
| structural-r1-15 | PROPOSE FIX | Make sibling documents relative links. |
| style-r1-11 | PROPOSE FIX | "is incomplete in three ways". |
| style-r1-12 | PROPOSE FIX | "would omit the standard Go fix for large caches". |
| style-r1-13 | PROPOSE FIX | "easy to bias by accident". |
| style-r1-14 | PROPOSE FIX | Code complexity measured as hot-path line count in `HOTPATH_NOTES.md`. |
| style-r1-15 | PROPOSE FIX | "models the workload of a documented production migration". |
| style-r1-16 | PROPOSE FIX | Use only "pointer layout" / "slab layout" or variant IDs after the table. |
| style-r1-22 | PROPOSE FIX | Concrete criteria for the four vague phrases. |
| style-r1-23 | PROPOSE FIX | Mark map-growth and musl/glibc claims "needs verification"; reword line 736 as expectation. |
| style-r1-24 | PROPOSE FIX | "Pingora is reported to have replaced NGINX (written in C)" (moves with Alternative M into Open Questions). |
| style-r1-26 | PROPOSE FIX | Remove the repeated copies (rules versioning, reaper guarantee, start-up footnote); point to one place. |
