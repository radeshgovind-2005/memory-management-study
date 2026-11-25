# Goldfish Review Report (Stage 7, review loop round 2)

**Target:** `docs/designs/big_cache.md` (lines 1-1072, everything before "## Detailed Implementation Plan")
**Mode:** REVIEW. GoldfishContext: none. Reviewers saw only the target, their persona and (for design and structural) `feature-spec-standard.md`.

## Reviewer Status

| Reviewer | Status | Raw findings |
| --- | --- | --- |
| `egm-design` | completed | 0 Blocker, 10 Major, 5 Nit |
| `egm-structural` | completed | 0 Blocker, 3 Major, 6 Nit |
| `egm-style` | completed | 0 Blocker, 9 Major, 22 Nit |

After deduplication: **0 Blocker, 20 Major, 28 Nit** (48 findings). Seven findings were raised by two reviewers and are merged below. The merged ids are listed under "Also raised as".

Settled findings: no reviewer re-raised any of the five settled points, so none are recorded as "settled".
"Needs verification" items: none of the findings below is raised only because a fact is unverified. Some findings ask that an *unmarked* claim be given the marker, or that a marked claim be stated precisely enough to check at C1.

---

## Blocker

None.

---

## Major

### design-r2-1. Rules can still change after a cloud result, which breaks Goal 6
* **Reviewer:** egm-design (also raised as style-r2-25)
* **Location:** lines 143-145 (Goal 6), 192, 631-632, 790-791 (D10), 1032-1036
* **Problem:** Goal 6 says the rule documents are frozen "before any cloud result". But the cloud smoke test is a cloud run, and the latency target "may be amended once ... at the cloud smoke test". If the smoke test shows per-variant latency, the author could move the pass/fail line after seeing which variants pass. D10 also says the threshold is checked "before any cloud result exists", which contradicts line 632.
* **Fix:** Pick one of these. (a) Freeze the latency target at C6 and let smoke-test amendments cover only operational items (timeouts, scrape config, sizing factors). (b) Define the smoke test so it gives no comparison: one variant only, or per-variant percentiles hidden until v1 is frozen. Then reword Goal 6 and D10 to match.

### design-r2-2. The C6 exit gate requires a positive result
* **Reviewer:** egm-design
* **Location:** line 191 (C6 exit), compared with lines 102-104 (Goal 1) and 989 (risk row)
* **Problem:** C6 ends only when "GC cycles line up with latency in go-default". Goal 1 and the risk table say the alignment measure is reported whatever it shows. As written, a local null result blocks all cloud work, which pushes the author to tune the workload until alignment appears.
* **Fix:** Change the C6 exit to "alignment measure computed and reported for local go-default runs; if there is no alignment, the cause is documented and a go/no-go decision is recorded."

### design-r2-3. The go-tuned tie test cannot work with 3 repetitions
* **Reviewer:** egm-design
* **Location:** lines 450-453 (component 10), 671, 796-801 (D12), 962-968 (Alternative T)
* **Problem:** Component 10 publishes confidence intervals only with at least 5 blocks, but D12 builds a 95% interval from 3 unpaired repetitions. With 2 degrees of freedom the intervals are very wide, so almost every setting "contains the best mean". The rule then reduces to "lowest memory-management CPU", which is the single-criterion rule Alternative T rejects. The tuning sweep also has no block or ordering structure, so drift is not controlled.
* **Fix:** Run the tuning sweep in shuffled blocks of at least 5. Define a tie as a paired difference from the best setting whose interval contains 0, or as a fixed margin (for example within 5% of the best p99.9). Make D12 and component 10 agree.

### design-r2-4. go-pointerfree's GC settings are unstated, and they affect the headline
* **Reviewer:** egm-design
* **Location:** lines 29-31, 48, 547 (`GOMEMLIMIT` column "go-tuned only"), 459-460
* **Problem:** By implication go-pointerfree runs with default `GOGC=100` and no `GOMEMLIMIT`. With a pointer-free heap of several GB, the heap goal is still about 2x live, so working set, bytes per entry and density can carry up to a live-heap's worth of garbage. The headline "gap left after Go's best layout" would then partly measure an untuned `GOGC`, not the layout or the language.
* **Fix:** State go-pointerfree's GC settings. Then either tune it (reuse D12's rule) or keep the default, label the headline "default-GC pointer-free Go", and note that its memory metrics include `GOGC` headroom.

### design-r2-5. With 5 blocks the exact sign test can never reach p < 0.05
* **Reviewer:** egm-design
* **Location:** lines 450-453, 624
* **Problem:** With 5 pairs, the smallest two-sided exact sign-test p-value is 2/32 = 0.0625. Reported next to the t interval, it shows every comparison as non-significant, even when all 5 blocks agree. The document has no rule for reading the sign test.
* **Fix:** Make the sign test descriptive ("k of 5 blocks same sign") and state its minimum possible p. Or raise headline comparisons to at least 6 blocks (minimum p = 0.031). Put the rule in METRICS.md.

### design-r2-6. Goal 1's pre-registered alignment measure is not fully defined
* **Reviewer:** egm-design (also raised as style-r2-2)
* **Location:** lines 99-104, 343-346, 383-384, 390-393
* **Problem:** The measure counts "seconds with a p99.9 spike", but "spike" has no threshold, so the measure cannot be tested. The text also leaves out which series defines it (client-side k6 or server-side HDR), the minimum requests per second (a per-second p99.9 is close to the per-second maximum at low rates), and how many random shifts are used and whether they are circular. Also, the claim that k6 remote-writes per-second summaries to Prometheus is not marked "needs verification". The reviewer believes k6's output pushes on an interval that defaults to about 5 s.
* **Fix:** Define the measure in METRICS.md at C2: the series (preferably server-side per-second HDR), the spike threshold (for example per-second p99.9 above k times the run's median per-second p99.9), the minimum samples per second, the number and kind of shifts, and the comparison statistic. Mark the k6 per-second claim "(needs verification)" and set the push interval explicitly.

### design-r2-7. Dropped-iteration outcomes depend on a k6 setting that is not specified
* **Reviewer:** egm-design
* **Location:** lines 347-353, 598-599, 617, 637-641
* **Problem:** In k6's `constant-arrival-rate` executor, an iteration is dropped only when every VU is busy, so the count depends on `preAllocatedVUs` and `maxVUs`. Run validity, the calibration stop and the held/degraded/overloaded classes all depend on dropped iterations. VU count also sets the number of keep-alive connections, which changes server memory and scheduling load.
* **Fix:** Fix a formula for `preAllocatedVUs` and `maxVUs` in RUN_PROTOCOL.md (for example rate x latency target x safety factor, capped). Use it for every variant at a given level, and record the values in the manifest.

### design-r2-8. Memory limits are sized at 70% load but used at 85% and 110%, and an OOM has no defined outcome
* **Reviewer:** egm-design
* **Location:** lines 549, 554-557, 604-606, 617-619, 626
* **Problem:** Limits are 1.3x the peak working set from a 3-minute pre-run at 70%. That covers about one forced-GC interval and may miss the peak of a full cycle. At 85% and 110% there are more in-flight requests, buffers and goroutines, so peak memory can be higher. An OOM kill makes the run invalid ("no restarts"), and the protocol does not say what happens next.
* **Fix:** Size from a pre-run at the highest level used, or one lasting at least 2 forced-GC intervals, or take the maximum over levels. State that an OOM at 110% is recorded as an outcome ("killed"), and say what an OOM at 25-85% triggers (rerun, resize or drop).

### design-r2-9. Rust stack unwinding is under-specified, so Rust lock-wait and drop categories may be wrong
* **Reviewer:** egm-design
* **Location:** lines 207-210, 412-436
* **Problem:** Only mimalloc gets frame pointers (through `CFLAGS`). The Rust crate needs `-C force-frame-pointers=yes`, and the precompiled standard library, where the futex `lock_contended` and thread-parking frames are, ships without them. Stacks through those frames and through out-of-line `drop_in_place`/`dealloc` may not unwind. The only smoke check, the allocation loop, would not catch this. The frame list also gives no precedence when a sample matches two categories (for example `runtime.lock2` under `mallocgc`).
* **Fix:** Specify `RUSTFLAGS=-C force-frame-pointers=yes`, plus either `-Zbuild-std` or DWARF unwinding (`perf --call-graph dwarf`). Add smoke checks for a contended-mutex loop and a drop-heavy loop in each language. Report the share of truncated stacks per run. Define a category precedence order in METRICS.md.

### design-r2-10. The C library behind the Rust image is an unlisted fairness confound
* **Reviewer:** egm-design (also raised as style-r2-28)
* **Location:** lines 336-337 (`gcr.io/distroless/static`), 89-92, 905-906 (Alternative L)
* **Problem:** A static distroless image implies a static Rust binary, usually built for musl. mimalloc replaces musl's malloc, but musl's `memcpy`/`memmove`/`memcmp` are generally slower than glibc's and are on the hot path (copying values into response buffers). Go uses its own routines. This is a non-language handicap missing from the list of differences at lines 89-92. Alternative L's statement about musl and glibc is also an unmarked comparison and does not say which libc the build uses.
* **Fix:** State the Rust target triple. Either build against glibc on `gcr.io/distroless/cc`, or keep musl and list it in FAIRNESS.md as a known confound, checked by a local musl-vs-glibc A/B run of `rust-index`. Mark the Alternative L claim "(needs verification)".

### structural-r2-1. Alternatives skip a letter and one is out of order
* **Reviewer:** egm-structural (also raised as style-r2-29, rated Nit there)
* **Location:** lines 901-977
* **Problem:** The letters go A to L, then jump to N (line 910); there is no M. Alternative U ("One Rust layout only (D5)", line 970) comes last, although every other D-tagged alternative follows decision order (I=D1 ... T=D12).
* **Fix:** Move the Alternative U block to just after Alternative L and rename it "Alternative M: One Rust layout only (D5)". The reviewer found no text that refers to alternatives by letter, so nothing else needs to change.

### structural-r2-2. The "at least 5 blocks" rule is stated without the exceptions the plan uses
* **Reviewer:** egm-structural (also raised as style-r2-1)
* **Location:** lines 107-108 (Goal 2), 624; contradicted by 452-453, 653-655, 671, 678
* **Problem:** Goal 2 and line 624 say every configuration gets at least 5 blocks. The cross-node check uses 3 blocks per node, the tuning sweep 3 repetitions and the memory-limit ramp 3 runs. Component 10 already treats 3-block results as a special case.
* **Fix:** Line 624 and Goal 2: "Every main-sweep and 4-vCPU configuration gets at least 5 blocks; the cross-node check uses 3 blocks per node (direction and size only, see Analysis), and the tuning sweep and memory-limit ramp use 3 repetitions." If design-r2-3 is accepted, update the tuning-sweep part.

### structural-r2-3. C1's exit criterion covers Open Questions that are not C1 items
* **Reviewer:** egm-structural (also raised as style-r2-18)
* **Location:** line 186 compared with lines 236-246
* **Problem:** C1 requires "each Open Question confirmed or refuted with a source in `claims.yaml`". But "Proxy framing" is answered before proxy work (after C11), "Repository home" is a decision that no source can confirm, and "Cloud provider" is the only fact-type entry with no "Answered at ..." line.
* **Fix:** Either change C1 to "each Open Question marked 'Answered at C1'", or add "Answered at C1" to Cloud provider and mark the other two as outside C1 (for example "Answered before proxy work (not a C1 item)" and "Decided by the author before C5").

### style-r2-3. "Keep their sign and rough size" has no test for "rough size"
* **Reviewer:** egm-style
* **Location:** lines 123-124, 465-466, 784-785
* **Problem:** The phrase appears three times, and no reader can tell whether a result passed.
* **Fix:** State the test once in D9 and refer to it elsewhere, for example "same sign, and each node's mean paired difference within a factor of 2 of the main-sweep value". Or drop "rough size" and write "same sign, with sizes reported".

### style-r2-4. "So no other process shares them" contradicts the hyperthread note
* **Reviewer:** egm-style
* **Location:** lines 241-243 compared with 559-562
* **Problem:** Lines 559-562 say the cache pod may still share physical cores, memory bandwidth and last-level cache through hyperthreads. The phrase "whole-number CPU exclusive cores" is also garbled.
* **Fix:** "a static CPU manager, which gives a Guaranteed pod with a whole-number CPU request exclusive vCPUs (sibling hyperthreads may still be shared; see Sizing table notes)".

### style-r2-5. "Request-path instrumentation is identical in all five variants" overstates
* **Reviewer:** egm-style
* **Location:** lines 410-411, compared with 396-398
* **Problem:** Go uses a wrapped `net.Conn` and Rust an `AsyncRead` shim. The two mechanisms are not identical.
* **Fix:** "Request-path instrumentation is identical within each language and follows the same stamping rule in both (FAIRNESS)."

### style-r2-6. A client-server percentile gap is called evidence of queueing, but it includes more than that
* **Reviewer:** egm-style
* **Location:** line 405
* **Problem:** The gap also includes network time, kernel accept and socket buffering, and k6 client overhead.
* **Fix:** "A gap between client and server percentiles measures time outside the server's stamped span: kernel and runtime queueing, network and client overhead."

### style-r2-7. "Both are AES-based on amd64" is an unmarked claim that depends on the build
* **Reviewer:** egm-style
* **Location:** lines 530-532
* **Problem:** ahash uses its AES path only when the binary is compiled with the AES target feature; otherwise it falls back to a non-AES hash. Go's map hash depends on runtime CPU detection.
* **Fix:** Mark it "(needs verification)" and name the build requirement, for example "the Rust build enables `target-feature=+aes`; verified at C4 by a check in the allocation tests or a startup log".

### style-r2-8. The D4 (mimalloc) rationale is vague
* **Reviewer:** egm-style
* **Location:** lines 750-752
* **Problem:** "A working set that tracks live data more closely (cleaner memory-limit behaviour)" does not say more closely than what, and "cleaner" is a judgement. The "needs verification" marker covers only maintenance.
* **Fix:** "Chosen because it is expected to return freed memory to the OS sooner than the system allocator and jemalloc (needs verification), so working set tracks live bytes during the memory-limit ramp, and because it is actively maintained (needs verification)."

### style-r2-9. Layout names drift after the document fixes them
* **Reviewer:** egm-style
* **Location:** lines 38-42, compared with 33, 35, 98, 127-128, 841-842
* **Problem:** Line 40 promises that the rest of the document uses **pointer layout** and **slab layout**, but line 39 then says "Go pointer ... Rust pointer-linked ... Go pointer-free ... Rust index-based". Lines 98 and 128 and Alternative D use other names. A new reader has to work out that five terms map to two layouts.
* **Fix:** Introduce the two names before the rung list, or say "rungs 1-2 and 4 share the pointer layout; rungs 3 and 5 share the slab layout". After that, use only those two names, apart from variant ids. "Pointer-heavy" can stay where it describes the heap rather than naming a layout.

---

## Nit

### design-r2-11. The calibration and sizing time budget looks low
* **Reviewer:** egm-design
* **Location:** lines 590-606, 664-665, 681
* **Problem:** The reviewer's estimate per session and entry count is about 70-80 minutes (coarse pass ~20, fine pass ~12, other variants ~16-20, sizing pre-runs ~20-25), not about 40. That is nearer 15 than 8 cluster-hours, and the 52-hour minimum campaign and the `session-up` credit gate depend on it.
* **Fix:** Recompute the figure from the protocol's step counts with an assumed capacity range, or state the assumed capacity behind 40 minutes.

### design-r2-12. "Final capacity" for pre-sized maps and record arrays is undefined
* **Reviewer:** egm-design
* **Location:** lines 498-516
* **Problem:** The bound is in logical bytes per shard, so the entry count depends on the value-size mix (180-byte minimum charge against about 280 mean). Sizing by the mean lets a shard of short values resize mid-measurement. Sizing by the maximum oversizes slab record arrays about 1.55x, which affects bytes per entry.
* **Fix:** In SPEC.md, define the pre-size (bound / minimum charge, or mean plus a stated margin). Add an `/admin/stats` assertion that no map grew after prefill.

### design-r2-13. The memory-limit ramp is not specified enough to compare variants
* **Reviewer:** egm-design
* **Location:** lines 552, 707-719
* **Problem:** It does not say how the 50% starting working set is reached, give a unit for the write rate (entries/s and bytes/s give different kill times), give the read rate, or give the window for the 25% memory-management CPU signal.
* **Fix:** Fix all four in RUN_PROTOCOL.md.

### design-r2-14. The reaper is a single cost safety net with known scheduling weaknesses
* **Reviewer:** egm-design
* **Location:** lines 296-298, 472-475, 1013-1016, 1030-1031
* **Problem:** GitHub Actions cron can be delayed, and scheduled workflows in public repositories are disabled after 60 days without activity. The cron interval, the token scope and leftover resources (volumes, load balancers) are not discussed.
* **Fix:** State the interval and worst-case delay. Make `session-up` refuse to start if the reaper has not run in the last N hours. Scope the token. Delete associated resources.

### design-r2-15. Smaller validity gaps
* **Reviewer:** egm-design
* **Location:** lines 387-393, 416-428, 653-655
* **Problem:** (a) Kernel-side memory-management work (page faults, `madvise` from mimalloc purging and the Go scavenger) is outside both CPU categories. (b) The cross-node check guarantees a different VM, not a different physical host. (c) k6's end-of-run percentiles keep every sample, which can reach hundreds of MB per trend metric at high rates.
* **Fix:** (a) Count kernel frames under a user-space memory-management ancestor, or report page-fault counts. (b) Say "different VM" and record any host id the provider exposes. (c) Size the load-pool memory for it.

### structural-r2-4. Some cross-references point to things that are not headings
* **Reviewer:** egm-structural (also raised as style-r2-30)
* **Location:** lines 474, 984, 986, 987, 988, 994
* **Problem:** "Cost control" and "Secrets" are bold bullet labels under Non-Functional Requirements, not headings. "Sizing text" (line 988) names no section.
* **Fix:** Follow the line 32 pattern: "(see Cost control under Non-Functional Requirements)". In the Risks table use "NFR: Cost control" and "NFR: Secrets". Replace "sizing text" with "Sizing table notes".

### structural-r2-5. The Budget's run order leaves out the Datadog profiling row
* **Reviewer:** egm-structural
* **Location:** lines 680, 684-690
* **Problem:** "Datadog profiling" is in neither the minimum campaign nor the follow-on list, so its place in the credit order is undefined.
* **Fix:** Add "and Datadog profiling (C9)" to the end of the follow-on list, or say it is budgeted separately.

### structural-r2-6. D2's title does not name the choice
* **Reviewer:** egm-structural
* **Location:** line 730
* **Problem:** Every other decision title reads "decided, <choice>". D2 reads only "Cache implementation source: decided".
* **Fix:** "D2. Cache implementation source: decided, hand-written sharded LRU from one shared spec (author)."

### structural-r2-7. Diagram box borders look misaligned in places
* **Reviewer:** egm-structural
* **Location:** lines 289-290, 309, 313-315
* **Problem:** These are hand counts. Line 314's right border seems one column further right than lines 313 and 315. The laptop box's top border (289) seems one column narrower than 290 and 295. Line 309 has no space before the closing border ("99 Hz)│").
* **Fix:** Check in a monospace render. Line up each box's right edges, and widen the line 309 box by one column.

### structural-r2-8. The overview table repeats a column
* **Reviewer:** egm-structural
* **Location:** lines 44-50
* **Problem:** The "Layout" column only restates `CACHE_LAYOUT`.
* **Fix:** Drop the "Layout" column, or merge the two under "`CACHE_LAYOUT` (layout name)".

### structural-r2-9. A divider splits the feature-level sections
* **Reviewer:** egm-structural
* **Location:** line 996
* **Problem:** The rule between Risks and Mitigations and Non-Functional Requirements makes the NFRs, Additional Resources and Future Improvements look like a separate part, although Goal 6 points into them.
* **Fix:** Remove the rule at line 996. Keep the one at line 1071, which separates the feature spec from the Detailed Implementation Plan.

### style-r2-10. "Almost nothing to scan" is vague
* **Reviewer:** egm-style
* **Location:** lines 30-31
* **Fix:** "records and slabs contain no pointers, so the garbage collector does not scan them".

### style-r2-11. "Typically short" has no number for C1 to check
* **Reviewer:** egm-style
* **Location:** lines 66-67
* **Problem:** The claim is marked, but C1 cannot confirm or refute it without a figure.
* **Fix:** Give the figure to be checked, for example "typically under 1 ms (value to be confirmed at C1)".

### style-r2-12. "If that remaining gap is small" depends on an undefined "small"
* **Reviewer:** egm-style
* **Location:** lines 92-93
* **Fix:** "The remaining gap is published whatever its size."

### style-r2-13. "Spends measurable CPU on GC marking" is filler
* **Reviewer:** egm-style
* **Location:** lines 98-99
* **Fix:** "reports the CPU share spent on GC marking".

### style-r2-14. "Confirmed by packing" assumes the outcome
* **Reviewer:** egm-style
* **Location:** lines 131-132, 771-773
* **Fix:** Use "checked by packing", since line 706 already reports the two numbers separately.

### style-r2-15. Two goal headings read awkwardly
* **Reviewer:** egm-style
* **Location:** lines 146, 150
* **Fix:** "The author can explain each hot path" and "Demonstrate Datadog for diagnosis and presentation".

### style-r2-16. "Supported well enough to use" has no criterion
* **Reviewer:** egm-style
* **Location:** lines 204-205
* **Fix:** "Does it produce symbolized CPU profiles for the Rust binary?"

### style-r2-17. The Pingora sentence stacks two hedges on the marker
* **Reviewer:** egm-style
* **Location:** lines 237-239
* **Fix:** "Pingora reportedly replaced NGINX (written in C), and linkerd2-proxy a Scala/JVM proxy (needs verification)."

### style-r2-19. The reason for using Python is weak
* **Reviewer:** egm-style
* **Location:** lines 357-358
* **Fix:** "a Python program (`campaign.py`), rather than a shell script, because shuffling, calibration and resume logic need structured state".

### style-r2-20. One parenthetical holds five facts
* **Reviewer:** egm-style
* **Location:** lines 371-375
* **Fix:** Split node count, node sizes, the C1 check, the same-node reason and QoS into separate sentences or one sub-bullet per pool.

### style-r2-21. The scope of "only" for Prometheus is unclear and the stated uses disagree
* **Reviewer:** egm-style
* **Location:** lines 378, 382-384, 391-393
* **Fix:** "For latency, Prometheus carries only k6's per-second series, used for live viewing and GC alignment." Make line 391 match.

### style-r2-22. The queueing sentence names a cause as if certain
* **Reviewer:** egm-style
* **Location:** lines 389-390
* **Fix:** "They include queueing time, such as waiting while GC workers hold the CPU."

### style-r2-23. "Variant minus reference" reuses a defined term
* **Reviewer:** egm-style
* **Location:** lines 448-449 (term defined at line 121)
* **Fix:** "variant minus baseline variant".

### style-r2-24. The restart rule is worded two ways
* **Reviewer:** egm-style
* **Location:** lines 618, 1005
* **Fix:** Use the line 618 wording ("no restarts unless the run is a memory-limit run") in both places.

### style-r2-26. Filler, and the slowdown factor is defined three times
* **Reviewer:** egm-style
* **Location:** lines 114, 633-635, 787-789
* **Fix:** Write "it is reported and decides nothing". Define the slowdown factor once, in the Latency target section, and refer to it elsewhere.

### style-r2-27. The D11 rationale states an expectation as fact
* **Reviewer:** egm-style
* **Location:** lines 793-794
* **Fix:** "Fewer vCPUs are expected to make the collector's CPU share more visible".

### style-r2-31. The rerun rule leaves the test to judgement
* **Reviewer:** egm-style
* **Location:** lines 1037-1039
* **Problem:** "If a change clearly affects only some variants" does not say what counts or who decides.
* **Fix:** "If a change alters code or settings used by only some variants, the author reruns those variants".

---

## Deduplication map

| Kept id | Merged duplicate | Note |
| --- | --- | --- |
| design-r2-1 | style-r2-25 | Latency target can be amended after a cloud result |
| design-r2-6 | style-r2-2 | Undefined "spike" in the alignment measure |
| design-r2-10 | style-r2-28 | musl/glibc |
| structural-r2-1 | style-r2-29 | Alternatives lettering (style rated it Nit; kept at Major) |
| structural-r2-2 | style-r2-1 | "At least 5 blocks" exceptions |
| structural-r2-3 | style-r2-18 | C1 and Open Questions |
| structural-r2-4 | style-r2-30 | "Sizing text" and non-heading cross-references |
