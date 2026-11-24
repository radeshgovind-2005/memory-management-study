# Goldfish Review Report: Stage 8 (Intermediate Zoom Out)

**Target:** `docs/designs/big_cache.md` (lines 1-1586, full file)
**Context given to reviewer:** Stage 5 `zoom_out_report.md` only.
**Standard:** `resources/document_standards/zoom-out-standard.md`

## Reviewer Status

| Reviewer | Status |
| --- | --- |
| `egm-zoom-out` | completed (first attempt) |

## Verdict

**Over-scoped. It needs cuts, not a redesign.**

The core experiment fits the goal and is well designed. That core is the five-rung ladder with layout twins, the pre-registered alignment statistic, the perf-based memory-management CPU share and the paired blocks. Every Stage 5 Major (M1-M14) and every Nit now has a real fix, and none of the fixes is wrong in principle.

The problem is how they were fixed. Most fixes took the strongest option offered, and several added sub-mechanisms of their own. Since Stage 5 the document has grown 87% (848 to 1586 lines), and it now has about 25 operational and measurement mechanisms where it had about 12. The budget is about 149 cluster-hours over about 466 runs. That is too much for one methodology page and one blog post. Dollars are not the main cost, since prices are still pending C1. The binding costs are author effort and the risk that the project never ships. A portfolio piece only has value once it is published.

The minimum campaign (about 60 hours) also has the wrong priorities. It leaves out the entry-count chart, the one chart that explains the mechanism.

If cuts 1-8 below are taken, the full plan drops to roughly 75-80 cluster-hours. The minimum stays at about 60 hours but then covers Goals 1-3.

## Severity counts

| Severity | Count |
| --- | --- |
| Blocker | 0 |
| Major | 7 |
| Nit | 3 |

## Prioritized Findings

### 1. Goal 4 (density and memory-limit behaviour) is a second story - Severity: Major
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Goal 4 (lines 130-137); Sizing table rows "Density (pack)" and "Memory-limit ramp" (572-573); "Density and memory-limit campaign" (728-760); D7 (811-814); Alternative O (975-980); C8 (194); files 40-41 (1423-1429); METRICS distress thresholds (1218-1220).
* **Discrepancy Description:** The post argues about where GC cost goes and how much of it Go can recover. Pods per node and behaviour near the OOM limit are a separate question. Supporting it takes two more sizing regimes, a per-replica rate rule, a variable-size SUT pool, two k6 scripts, four distress signals and a checkpoint. The ramp is expected to end in an OOM kill for every variant (line 748), so its result is mostly descriptive.
* **Rationale & Recommendation:** Keep calculated density as one table row: allocatable memory divided by measured working set, plus CPU per 1,000 requests (this is Alternative O). Move the packing run and the memory-limit ramp to a follow-on post. Remove C8, D7 and the two sizing rows.

### 2. The minimum campaign leaves out the mechanism chart - Severity: Major
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Goal 3 (126-129); Budget, "Minimum publishable campaign" (719-726).
* **Discrepancy Description:** The entry-count curve (GC cost grows with pointer count while Rust stays flat) is the most persuasive chart, and Goal 3 says it "explains the mechanism". The minimum campaign leaves it out and spends about 35 hours on five load levels at one size.
* **Rationale & Recommendation:** Redefine the minimum as the tuning sweeps, plus a 3-level load sweep (25/70/110%), plus the entry-count sweep at 1M/5M/10M. This stays at about 60 hours and covers Goals 1-3.

### 3. Calibration and sizing are heavier than the conclusions need, and session length is unstated - Severity: Major
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Run protocol steps 1-2 (613-629); Budget calibration (697-699, 716); CUJ 1 (259-268).
* **Discrepancy Description:** Each session and entry-count combination costs about 75 minutes, or about 15 hours in total. The entry-count sweep runs at one absolute rate (680-685), so calibrating separately at each entry count only produces a side result. The document never states session length. If the roughly 35-hour load sweep spans several sessions, each recalibration shifts what "70%" means, and the "same node" premise of D9 quietly fails.
* **Rationale & Recommendation:** Calibrate once in the first session, for each entry count that sets a rate (5M and 10M). Freeze those absolute rates in rules v1 and reuse them in every later session. Make the block the unit of pairing, state a maximum session length, and drop the per-entry-count capacity side result. Saves about 8-10 cluster-hours.

### 4. Request stamping at the I/O layer buys almost nothing - Severity: Major
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Component 7 server side (403-416); FAIRNESS instrumentation clause (1177-1179); files 18, 22, 31 (1304-1308, 1327-1331, 1377-1381); pipelined conformance case (1290-1291).
* **Discrepancy Description:** The design adds a wrapped `net.Conn`, an `AsyncRead` shim, a special rule for `net/http`'s one-byte background read and a pipelined-client conformance test. The stamp still misses runtime queueing (the document says so at lines 414-416). Over a stamp at handler entry, the only thing it adds is header-parse time, which is small and comparable in both languages. The headline metric is client-side k6 latency anyway.
* **Rationale & Recommendation:** Stamp at handler entry with ordinary middleware in both languages and document what that excludes. Use the gap between client and server latency as the queueing measure. Drop the shims, the pipelined test and the FAIRNESS stamping clause.

### 5. The two robustness studies (D9 cross-node, D11 4-vCPU) cost about 31 hours for one paragraph - Severity: Major
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Goal 2 "If credit allows" (122-125); D9 (820-829); D11 (835-838); Sizing table 4-vCPU row (571); Budget rows (710-711); Alternatives Q and S.
* **Discrepancy Description:** Both studies are marked optional. They still add design surface: a larger node type, a separate sizing pre-run and a cross-node test rule. They also rank above the entry-count work in priority.
* **Rationale & Recommendation:** Move D11 to Future Improvements and state "2 vCPUs" as a limitation. The risk row at line 1035 can rely on the pre-registered negative-result rule instead. Run the cross-node check (D9) at the 70% level only. Saves about 14 + 8.5 cluster-hours.

### 6. Two full 12-point tuning grids use about a third of the minimum campaign - Severity: Major
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** FAIRNESS tuning budget (1190-1196); D12 (839-849); Budget rows (705-706).
* **Discrepancy Description:** The grids are 72 runs at the full 17-minute protocol, spent only on choosing settings. For `go-pointerfree` the GC has little to scan, so GOGC should matter little.
* **Rationale & Recommendation:** Use 6 settings per variant (GOGC 100/200/400 crossed with GOMEMLIMIT 80/90%) and a 6-minute window, which still covers three forced-GC intervals. Saves about 10-13 cluster-hours.

### 7. DWARF unwinding drives the large-file bucket, public egress and a risk row - Severity: Major
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Component 8 (443-454); component 9 (461-462); CUJ 2 (274-276); Risks egress row (1040); file 46.
* **Discrepancy Description:** `perf --call-graph dwarf` copies a stack slice with every sample, which likely produces perf.data files in the gigabytes per run. Much of the bucket, checksum and egress machinery exists to carry that data. The categories (`mi_*`, `runtime.mallocgc`, `gcBgMarkWorker`) are decided by frames at or near the leaf, and frame-pointer unwinding captures those.
* **Rationale & Recommendation:** Use frame pointers in both binaries and accept truncation in non-generic std code; the truncated-stack share is already reported. At C1, confirm that the pinned Rust std ships with frame pointers (the reviewer flags this as needing verification). Commit folded stacks only, and keep the raw bucket private or drop it.

### 8. The cost-safety design layers a heartbeat on top of a TTL - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Component 4 (370-375); component 12 (490-493); Cost control (1057-1068); token-scope Open Question (234-236); file 48.
* **Discrepancy Description:** A TTL tag plus a scheduled reaper already limits spend from an abandoned cluster. The heartbeat adds an in-cluster tagging token, a C1 question about token scope and a "reaper ran in the last 24h" check.
* **Rationale & Recommendation:** Keep the in-cluster Job and a TTL-only reaper, sized to each campaign's estimate plus a margin. Drop the heartbeat and the tagging token.

### 9. The document repeats itself, and two copies already contradict each other - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Goals, Components, Run protocol, D1-D12, Alternatives A-T, file items 2-5, 7-9, 14, 25-26; specifically lines 562-564, 708, 1570, 1578, 192.
* **Discrepancy Description:** Most rules appear three to five times, which is why a "SPEC.md wins" clause is needed. Two copies already disagree:
  * Line 1578 ("sampler on versus off for each variant") conflicts with line 708 (one variant per language).
  * Line 1570 (C6 expects "GC cycles lining up") conflicts with line 192 (C6 allows no alignment).
* **Rationale & Recommendation:** State each rule once, in the Technical Plan or in the description of its rules file, and reference it everywhere else. Cut Alternatives to one line where a D-decision already gives the reasoning. Shorten the file items listed above, and fix the two contradictions. Saves about 300-400 lines.

### 10. Sampler-overhead runs for the remaining variants add nothing - Severity: Nit
* **Reviewer Lens:** `egm-zoom-out`
* **Affected Scope:** Budget (712); component 8 (451-454).
* **Discrepancy Description:** One variant per language already bounds the sampler overhead.
* **Rationale & Recommendation:** Delete the extra runs. Saves 5 cluster-hours.

## Accepted as proportionate

The reviewer checked these and found them proportionate:
* the five-rung ladder and its layout twins (D5)
* `unsafe` code checked with Miri
* the hit-ratio simulator
* the pre-registered alignment statistic
* paired Student's t intervals
* `ahash` with AES
* the Datadog campaign (Goal 8, about 3 hours)
* the in-cluster campaign Job

There was nothing to search for reuse because no codebase exists yet, and the design already uses standard tools.

Optional larger cut (not recommended by default): once Goal 4 is out, plain VMs running Docker with cgroup limits could replace DOKS, Helm and the three node pools. The reviewer keeps Kubernetes because the target audience runs it and the follow-on memory-limit post will need it.

## Proposed cuts, ranked by value to effort

| # | Cut | Saves | What the post loses |
| --- | --- | --- | --- |
| 1 | Move the 4-vCPU check (D11) to Future Improvements | ~14 cluster-hours, a sizing row, D11, Alternative S, ~25 lines | A "more cores" paragraph, replaced by a stated limitation |
| 2 | Drop the remaining sampler-overhead runs | 5 cluster-hours | Nothing |
| 3 | Move density packing and the memory-limit ramp to a follow-on post; keep calculated density | ~9 cluster-hours, C8, D7, 2 sizing rows, 2 k6 scripts, ~70-80 lines | The packed-node demo and the OOM section, which work better as their own post |
| 4 | Tuning grids: 6 settings each, 6-minute windows | ~10-13 cluster-hours | Nothing |
| 5 | Calibrate once for each entry count that sets a rate; reuse the absolute rates; state a maximum session length | ~8-10 cluster-hours, an orchestrator branch | The per-entry-count capacity side result |
| 6 | Cut load levels to 3 and add the entry-count sweep to the minimum campaign | Roughly cost-neutral; the minimum now covers Goal 3 | The 50% and 85% load points |
| 7 | Stamp server latency at handler entry, not at the I/O layer | 2 custom shims, a conformance case, a FAIRNESS clause | Nothing it could claim |
| 8 | Run the cross-node check (D9) at 70% only | ~8.5 cluster-hours | The cross-node check at overload |
| 9 | Use frame pointers instead of DWARF; keep raw perf data private | The large-file and egress path, a risk row | Nothing, if C1 confirms std frame pointers |
| 10 | Use a TTL-only reaper (no heartbeat) | The tagging token, a C1 item, ~20 lines | Nothing |
| 11 | Consolidate repeated rules and Alternatives | ~300-400 lines | Nothing |

Taking cuts 1-8 brings the full plan from about 149 to about 75-80 cluster-hours. Taking cuts 1-11 also brings the document to about 1000-1100 lines.
