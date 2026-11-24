# Goldfish Review Report: Stage 7 Critic Review, Round 1

**Target:** `docs/designs/big_cache.md` (FeatureRange lines 1-993, everything before "## Detailed Implementation Plan")
**Mode:** REVIEW
**Standard:** `resources/document_standards/feature-spec-standard.md` (egm-design, egm-structural); persona rules only (egm-style)
**GoldfishContext:** none

## Reviewer Status

| Reviewer | Status | Raw findings |
| --- | --- | --- |
| `egm-design` | completed | 0 Blocker, 12 Major, 6 Nit |
| `egm-structural` | completed | 0 Blocker, 5 Major, 10 Nit |
| `egm-style` | completed | 0 Blocker, 10 Major, 15 Nit (plus an optional redundancy note) |

After deduplication: **0 Blocker, 26 Major, 28 Nit.**

How duplicates are handled: when two or more reviewers raised the same issue, the finding keeps one id and lists the others under "Also reported as". No reviewer marked anything a Blocker. Items marked "needs verification" were not reported just for being unverified.

---

## Blocker

None.

---

## Major

### design-r1-6: There is no Risks and Mitigations section
- **Reviewers:** egm-design, egm-structural. Also reported as structural-r1-1.
- **Location:** whole range, lines 1-993.
- **Problem:** The standard requires the spec to name its failure modes, security implications and scalability limits, and to say how each one is mitigated. In this document that material is scattered across the "Accepted cost" lines in D1-D12, the Open Questions, the NFRs, the Secrets bullet and Goal 1. Several risks are never stated with a mitigation:
  - credit running out partway through a campaign
  - the privileged perf DaemonSet on the SUT node
  - the provider refusing perf
  - noisy neighbours, or variation between dedicated-CPU hosts
  - the mechanism not reproducing on 2-core nodes
  - the frame list misattributing CPU
  - bugs in AI-generated code, or unsafe Rust corrupting memory under load
- **Fix:** Add "## Risks and Mitigations" after Alternatives Considered. Use a table with the columns Risk, Type (failure / security / scalability / validity), Likelihood, Impact, Mitigation, and Where enforced (D-number, Goal, NFR or checkpoint). Include the risks in design-r1-7, design-r1-8 and design-r1-11.

### design-r1-1: The sizing pre-run needs a capacity that has not been measured yet
- **Reviewer:** egm-design
- **Location:** Run protocol, lines 547-560; sizing text, lines 519-522.
- **Problem:** Step 0 (Size) runs each variant at "70% load". That percentage is a share of capacity, but capacity is only found in step 1 (Calibrate). Calibration in turn needs the pod memory limit, which comes from sizing. As written, the steps cannot run in this order.
- **Fix:** Calibrate first, against a pod with a generous fixed limit. Then size each variant at 70% of the calibrated rate, and then set the per-variant limits. Also say which absolute rate the entry-count sweep uses for sizing.

### design-r1-2: Calibration steps are shorter than the GC cycle being studied
- **Reviewer:** egm-design
- **Location:** Calibrate, lines 550-560; compare lines 78-81 and 566-568.
- **Problem:** The calibration steps last 60 seconds and the per-variant cross-check lasts 2 minutes. Both can miss a roughly 2-minute forced GC cycle completely, especially for go-pointerfree. That overstates capacity for exactly the variants whose weakness is periodic, and every load level from 25% to 110% inherits the error.
- **Fix:** Hold each step for at least one GC cycle, confirmed from gctrace, or judge each step on its worst 10-second p99. Make the cross-check at least one forced-GC interval long. Write this rule into RUN_PROTOCOL v0.

### design-r1-3: The 5M point is not actually shared between the two sweeps
- **Reviewer:** egm-design
- **Location:** Campaign shape, lines 599-604; Budget, line 622.
- **Problem:** The entry-count sweep uses one fixed absolute rate (70% of go-default's capacity at 10M). The load-level sweep uses percentages of the 5M capacity. These two rates differ, so the claim "5M is shared" is wrong, and the entry-count curve would have one point measured at a different rate.
- **Fix:** Either add 5M runs at the fixed rate (about 25 runs, roughly 7 cluster-hours) and update the budget, or redefine the entry-count rate so that it equals one of the 5M load levels.

### design-r1-4: Bootstrap intervals over 3 to 5 blocks are unreliable, and the D12 tie rule is ambiguous
- **Reviewers:** egm-design, egm-style. Also reported as style-r1-21.
- **Location:** Analysis, lines 416-428; D12, lines 739-744; D9 cross-node check, lines 605-607.
- **Problem:** A percentile bootstrap has only 126 distinct resamples when n=5 and only 10 when n=3. The resulting intervals are too narrow. Separately, the D12 rule ("if two settings' confidence intervals overlap") does not work across 12 candidates, because overlap is not transitive.
- **Fix:** Use a t-interval or an exact sign or permutation test on the paired block differences, or raise the headline comparison to 8-10 blocks, and state the minimum number of blocks needed before an interval is published. Make D12 mechanical: take the setting with the lowest mean p99.9, then among all settings whose interval contains that value, pick the one with the lowest memory-management CPU share.

### design-r1-5: Go and Rust start the server-side latency clock at different points
- **Reviewer:** egm-design
- **Location:** component 7, lines 367-374.
- **Problem:** Go's time stamp is taken before header parsing, but Rust's is taken after hyper has already parsed the request. Any GC or allocator cost during parsing therefore lands only on Go's side of a co-headline metric. Separately, net/http starts a background 1-byte read, so "first read after each response" needs a precise definition.
- **Fix:** Stamp at the I/O layer in both languages, using an AsyncRead shim around the Rust TcpStream. Define the stamp as the moment the read returns. Add a conformance test with a pipelined, delayed keep-alive client.

### design-r1-7: The node's memory may not fit go-default at 10M entries
- **Reviewer:** egm-design
- **Location:** Sizing table, lines 512-517; component 5, line 346.
- **Problem:** A rough estimate is about 3 GB of live heap at 10M entries. That peaks at about 6-7 GB under GOGC=100, which means a pod limit of about 8-9 GB. The spec never states the node's memory, and a typical 4-vCPU CPU-optimized node has 8 GB.
- **Fix:** Name the node type and its memory. Add a feasibility estimate (pod limit plus system reserve) for go-default at 10M. Add a C1 check that it fits, or cap the entry-count sweep.

### design-r1-8: The claim that "the other cores do not compete" ignores hyperthreading
- **Reviewers:** egm-design, egm-style. Also reported as style-r1-19.
- **Location:** Sizing text, lines 522-524.
- **Problem:** Cloud vCPUs are usually hyperthreads, so the cache pod's vCPUs may share physical cores with the kubelet and perf. Memory bandwidth and the last-level cache are shared in any case. The text also mixes "cores" and "vCPUs".
- **Fix:** Use "vCPU" consistently. Record `lscpu` topology and the CPU-manager assignment in every manifest. Where the provider allows it, give the pod whole physical cores. Otherwise state this as a limitation, and reword the claim to "so they do not share the cache pod's vCPUs".

### design-r1-9: In the memory-limit ramp, pre-sizing is aimed above the pod limit
- **Reviewer:** egm-design
- **Location:** lines 465-478; ramp row, line 517; lines 654-664.
- **Problem:** Maps and slab arrays are created at a final capacity derived from CACHE_MAX_BYTES, which is 1.5 times the pod limit. Whether those pages count toward the working set depends on how each runtime or allocator touches them, and Go and mimalloc may differ. That can cause an early OOM, or shift the distress fraction, for reasons unrelated to GC.
- **Fix:** Define the ramp's pre-sizing explicitly. Either size for the expected entry count at the limit, or measure each variant's working set right after startup, record it in the manifest, and subtract it.

### design-r1-10: The `memory.events` `high` distress signal will probably never fire
- **Reviewers:** egm-design, egm-style. Also reported as part of style-r1-22.
- **Location:** Memory-limit ramp, lines 660-662.
- **Problem:** The `high` counter only increases when memory.high is set. Kubernetes sets memory.high only when the MemoryQoS feature gate is on, and managed clusters usually leave it off.
- **Fix:** Use the `max`, `oom` and `oom_kill` counters in memory.events, together with memory.pressure (PSI). Or set memory.high explicitly, to the same value for every variant, and record it. Name the exact counter and threshold, for example "a nonzero `max` counter".

### design-r1-11: The token-scoping security claims are unbacked, and the broad cloud token may end up in the cluster
- **Reviewer:** egm-design
- **Location:** component 4, lines 335-337; Secrets, lines 948-953.
- **Problem:** The spec says the Job's token "can only update tags on its own cluster", but provider tokens appear to be scoped by resource type, not by individual resource, and Spaces uses separate keys. The Secrets section also suggests that the full cloud token is passed into a cluster that runs a privileged DaemonSet.
- **Fix:** Add per-resource scoping to the C1 checklist. State that the full-account token never enters the cluster. Document the smallest scope actually available. Give the results bucket's access policy and its egress cost for CUJ 2 readers.

### design-r1-12: The mark-phase explanation may be out of date for current Go
- **Reviewer:** egm-design
- **Location:** Problem, lines 67-81; Goal 1; Alternatives.
- **Problem:** Go's newer "Green Tea" GC changes marking to work span by span. If the pinned Go version uses it by default, the mechanism story and the predictions change. The document does not mention it anywhere.
- **Fix:** Add a C1 check of which collector the pinned version uses by default. Consider a GOEXPERIMENT on/off rung or a footnote. State in PREDICTIONS.md which collector the predictions assume.

### structural-r1-2: Decision D5 has no rejected alternative
- **Reviewer:** egm-structural
- **Location:** D5, lines 701-707; Alternatives, lines 746-918.
- **Problem:** Every other decision has a matching alternative tagged with its D-number. D5 (both Rust layouts as core rungs) has none, which breaks the Alternatives Rule.
- **Fix:** Add "Alternative U: One Rust layout only (D5)" with Pros, Cons and a Reason for Rejection. The reason is that without the twin layout, the effect of layout cannot be separated from the effect of language.

### structural-r1-3: Alternative M is an open question filed as an alternative
- **Reviewer:** egm-structural
- **Location:** lines 851-858; cross-references at lines 158 and 986.
- **Problem:** Alternative M's "Reason for Rejection" says "Open, deferred", so it is not a rejection. The proxy-framing question it holds is also missing from Open Questions (lines 200-224).
- **Fix:** Move "Proxy framing" into Open Questions, to be answered at C1. Delete Alternative M, or rewrite it as a real rejected alternative. Repoint lines 158 and 986 to the Open Question.

### structural-r1-4: Two closing paragraphs render as part of the list item above them
- **Reviewers:** egm-structural, egm-style, egm-design. Also reported as style-r1-17 and part of design-r1-18.
- **Location:** lines 427-432 (Analysis: "Main-sweep results are stated as holding for one node..."); lines 664-666 (Memory-limit ramp: "A memory-pressure-aware rung ...").
- **Problem:** Neither paragraph has a blank line separating it from the list above, so Markdown joins it to the last bullet. As a result, the D9 cross-node statement reads as a note on the rust-index minus rust-pointer comparison only.
- **Fix:** Put a blank line before each paragraph. If the paragraph belongs to the parent item, indent it to that item's content column; otherwise end the list first.

### structural-r1-5: Additional Resources and Future Improvements are nested under Non-Functional Requirements
- **Reviewer:** egm-structural
- **Location:** lines 963 and 976.
- **Problem:** Both are H3 headings under the NFR section, which makes them read as requirements. Neither one is.
- **Fix:** Promote both to H2, or move them after the Detailed Implementation Plan.

### style-r1-1: The remaining gap is credited to the memory model alone
- **Reviewer:** egm-style
- **Location:** Problem, lines 89-91; D2, lines 676-677.
- **Problem:** The text says "Whatever gap remains... is the part Rust's memory model accounts for", and D2 says the language is "the only difference". The design itself lists other differences between the pairs: the HTTP stack, the mutex, the map and hash, the scheduler and the timing start point (lines 369-374).
- **Fix:** At line 89, attribute the gap to "the runtime and its memory management, together with the HTTP stack, mutex and map differences listed in FAIRNESS". In D2, write "layout and algorithm are held constant; the remaining differences are listed in FAIRNESS".

### style-r1-2: Component 8 contradicts the Concurrency section about lock contention
- **Reviewer:** egm-style
- **Location:** line 398 versus lines 494-495.
- **Problem:** Line 398 says hot-shard contention "behaves differently in each language". Line 495 says it "is the same in both languages". Neither claim has evidence behind it.
- **Fix:** In Concurrency, say the lock pattern is identical in both languages and its cost is measured separately as lock wait. At line 398, give the reason as "because mutex implementations differ".

### style-r1-3: "Headline" has several meanings and no fixed metric
- **Reviewer:** egm-style
- **Location:** lines 104-112, 364-365, 426, 916.
- **Problem:** "Headline" is used for a metric, a "co-headline", a comparison and a configuration. The parenthetical on line 110 is also ambiguous about which item it attaches to.
- **Fix:** Define the term once in Goal 2 ("Headline metric: client-side p99.9"). Rename the configuration to "reference configuration". Keep "headline comparison". Replace "co-headline" with "reported next to the server-side values with equal weight".

### style-r1-4: The claim that hit counts match overstates the guarantee
- **Reviewer:** egm-style
- **Location:** lines 484-486.
- **Problem:** Identical eviction holds only for the single-threaded replay. Under concurrent k6 load the request order differs between runs, which line 574 already accepts with a 2-point tolerance.
- **Fix:** Write "...evict the same keys for the same request sequence (checked by the conformance replay); under load, hit ratios agree within the run-protocol tolerance."

### style-r1-5: The Goals promise checks that the checkpoints make optional
- **Reviewers:** egm-style, egm-design. Also reported as part of design-r1-18.
- **Location:** Goal 2, lines 104-106 and 119-120; component 8, lines 405-406; C7, line 191; Budget, lines 632-637.
- **Problem:** The cross-node check, the 4-core check and the sampler-overhead measurement are stated as unconditional. C7, however, runs them "as credit allows", and the minimum campaign leaves them out. The C7 exit criterion cannot be tested.
- **Fix:** Mark these checks "if credit allows (see Budget)" in Goal 2, component 8 and D9/D11. Rewrite the C7 exit as: "tuning sweep and load-level sweep committed; remaining campaigns run in the Budget order until credit is exhausted; the post lists which ran."

### style-r1-6: "To show the result is not specific to one machine" overclaims
- **Reviewer:** egm-style
- **Location:** Goal 2, line 106; D9, lines 727-728.
- **Problem:** Two extra nodes cannot show that. Lines 429-431 already use the correct, weaker wording.
- **Fix:** Write "to check whether the headline differences keep their sign and rough size on other nodes."

### style-r1-7: The validity thresholds use "about"
- **Reviewer:** egm-style
- **Location:** lines 326, 572, 927.
- **Problem:** "about 70%" is used as a pass/fail rule, and line 572 says only "under the limit".
- **Fix:** Write "k6 node CPU, averaged over 10-second windows, stays at or below 70% for the whole measurement window", and use the same wording in all three places.

### style-r1-8: The latency target's "with room to spare" cannot be measured
- **Reviewer:** egm-style
- **Location:** line 584.
- **Problem:** The phrase is not a number, so no one can check whether a run met it.
- **Fix:** Write "every variant's p99 at the 25% level is at most half the target", or name the margin that FAIRNESS will fix.

### style-r1-9: Goal 2 is one sentence of about 120 words
- **Reviewer:** egm-style
- **Location:** lines 100-117.
- **Problem:** The single sentence mixes the conditions, the definition of a block, the reference configuration, the load levels, the outcome classes and the slowdown factor.
- **Fix:** Split it into "Conditions" and "Metrics". Point to the Run protocol (lines 579-595) for the load levels and outcome definitions.

### style-r1-10: Alternative P assumes the result
- **Reviewer:** egm-style
- **Location:** lines 881-882.
- **Problem:** The phrase "the 'traffic Go cannot handle and Rust can' outcome is never shown" predicts the finding, in a design that says it will publish a small gap if that is what the data show.
- **Fix:** Write "without an overload level, behaviour beyond capacity, where variants may differ in whether they keep up, is never measured."

---

## Nit

### design-r1-13: Goal 1's "at least one committed run" invites cherry-picking
- **Reviewer:** egm-design. **Location:** lines 97-99.
- **Fix:** Pre-register an alignment measure across all go-default runs, for example the share of p99.9-spike seconds that overlap a GC cycle, compared against a time-shifted baseline.

### design-r1-14: go-tuned is tuned at only one point
- **Reviewer:** egm-design. **Location:** D12; tuning sweep in the Budget.
- **Fix:** State in the post that the 5M / 70% / 2-vCPU tuning is reused everywhere, as a limitation. Or retune briefly at 10M and at 4 cores.

### design-r1-15: The Budget leaves out real costs
- **Reviewers:** egm-design, egm-style. Also reported as style-r1-20. **Location:** lines 613-637.
- **Problem:**
  - Calibration overhead is probably above 10%: a 10% ramp from 500 rps takes about 48 steps.
  - Ramp runs last up to 20 minutes plus prefill, not 17.
  - Larger nodes cost more per cluster-hour.
  - The 45-hour minimum leaves out the overhead.
  - 117 rows plus 10% comes to about 129, not 127.
  - "3 blocks" in the tuning row misuses the term "block".
  - "Node-hour figures are cluster-hours" is redundant.
- **Fix:** Correct the figures. Use a coarse-then-fine calibration ramp. Give dollar figures per node type. In the tuning row, write "3 repetitions".

### design-r1-16: The load node is never sized
- **Reviewer:** egm-design. **Location:** component 3; density run.
- **Fix:** Size the load pool from the expected capacity, and say how density packing scales the load, so the k6 node stays under its 70% CPU limit.

### design-r1-17: THP, kernel version and cgroup version are missing from the manifest
- **Reviewer:** egm-design. **Location:** Goal 5, lines 134-137.
- **Fix:** Record all three in the manifest, and pin the THP mode in FAIRNESS. THP affects both Go and mimalloc RSS.

### design-r1-18: "CPU per 1,000 requests" does not say whether the cache-aside PUT counts as a request
- **Reviewer:** egm-design. **Location:** Goal 2.
- **Fix:** Define whether the PUT that follows a miss counts as a request. (The other editorial points in this design finding are merged into structural-r1-4, structural-r1-9 and style-r1-5.)

### structural-r1-6: Many "Reason for Rejection" lines only restate the choice
- **Reviewer:** egm-structural. **Location:** Alternatives D, H, I, K, L, N, O, P, Q, R, S, T.
- **Problem:** These lines say only "Rejected by author in favour of X", and H says only "removed entirely". egm-design judged the Alternatives Rule met on content, because the trade-offs appear in the Cons lines.
- **Fix:** Add one clause to each Reason naming the deciding trade-off. For example, for L: "glibc retains memory and jemalloc returns it gradually, which muddies the memory-limit results".

### structural-r1-7: Checkpoints and Open Questions come before the material they refer to
- **Reviewer:** egm-structural. **Location:** lines 178-224.
- **Fix:** Move both sections after the Technical Plan or after Alternatives.

### structural-r1-8: Two headings differ from the standard's names
- **Reviewer:** egm-structural. **Location:** lines 61 and 262.
- **Fix:** Rename to "Problem Statement" and "Proposed Architecture", or add the standard term in brackets.

### structural-r1-9: Line lengths are inconsistent
- **Reviewers:** egm-structural, egm-design. **Location:** over-long lines 386, 557, 652; short-wrapped lines 221-222, 484, 585.
- **Fix:** Rewrap these lines to about 80 columns.

### structural-r1-10: The Run protocol numbering is inconsistent
- **Reviewer:** egm-structural. **Location:** lines 547-577.
- **Fix:** Number the steps 1-3 (Size, Calibrate, Measure). Turn "At least 5 blocks" into a rule sentence after the list.

### structural-r1-11: "All three values" should say four
- **Reviewers:** egm-structural, egm-style. Also reported as style-r1-18. **Location:** line 519.
- **Fix:** Write "All four values". The table has four setting columns.

### structural-r1-12: Rule files are named inconsistently
- **Reviewer:** egm-structural. **Location:** lines 140, 374, 505, 506, 574, 704, 718, 925.
- **Fix:** Give each file's full repository path in backticks on first use, then use one short name throughout.

### structural-r1-13: The line 160 cross-reference cites D5 loosely
- **Reviewer:** egm-structural. **Location:** line 160.
- **Fix:** Cite D2 only, or say what D5 adds.

### structural-r1-14: Additional Resources describes finding codes the document never uses, plus process jargon
- **Reviewers:** egm-structural, egm-style. Also reported as style-r1-25. **Location:** lines 965-968.
- **Problem:** The codes B1-B8 and M1-M10 never appear in the document, and "M" can be confused with Alternative M. The sentence about the EGM process adds nothing for a reader.
- **Fix:** Delete the sentence, or cite the codes where the findings were applied.

### structural-r1-15: Related documents are not links
- **Reviewer:** egm-structural. **Location:** lines 965-974.
- **Fix:** Make the four sibling documents relative Markdown links. The missing Discord URL is a known open item and is not raised here.

### style-r1-11: The Problem section makes subjective claims about readers
- **Reviewer:** egm-style. **Location:** lines 63-65.
- **Fix:** Write "'Rust has no garbage collector, so it has no GC pauses' is incomplete in three ways:".

### style-r1-12: Hyperbole about Go practitioners
- **Reviewer:** egm-style. **Location:** lines 76-77 and 778.
- **Fix:** Write "would omit the standard Go fix for large caches".

### style-r1-13: "Rig by accident" contradicts itself
- **Reviewer:** egm-style. **Location:** line 82.
- **Fix:** Write "easy to bias by accident".

### style-r1-14: "At what cost in code complexity" is never measured
- **Reviewer:** egm-style. **Location:** lines 88-89.
- **Fix:** Define a measure, such as hot-path line count from HOTPATH_NOTES.md, or drop the phrase.

### style-r1-15: "Rebuilds the conditions" overclaims
- **Reviewer:** egm-style. **Location:** line 11.
- **Fix:** Write "models the workload of a documented production migration".

### style-r1-16: Layout names drift
- **Reviewer:** egm-style. **Location:** lines 38-42, 96, 123, 256, 782-783.
- **Fix:** After line 42, use only "pointer layout" and "slab layout", or the variant IDs.

### style-r1-22: Hedged or vague phrases
- **Reviewer:** egm-style. **Location:** lines 204, 495-498, 499, 695. (The line 662 item is merged into design-r1-10.)
- **Fix:**
  - Line 204: give a concrete criterion, such as "produces symbolized CPU profiles for the Rust binary".
  - Lines 495-498: write "both AES-based".
  - Line 499: give the exact value of s ≈ 1.1, or point to SPEC.md.
  - Line 695: write "a working set that tracks live data more closely".

### style-r1-23: Unsourced claims about existing behaviour
- **Reviewer:** egm-style. **Location:** lines 477-478, 736, 846.
- **Problem:**
  - Lines 477-478 state map-growth behaviour for Go and Rust. Go's map implementation has changed recently, so this needs checking for the pinned version.
  - Line 846 makes unsourced claims about musl and glibc.
  - Line 736 states a hypothesis as fact.
- **Fix:** Mark these claims "needs verification" or cite a source, and reword line 736 as "are expected to make".

### style-r1-24: Weak attribution and an unclear "(C)"
- **Reviewer:** egm-style. **Location:** line 854.
- **Fix:** Write "Pingora is reported to have replaced NGINX (written in C)".

### style-r1-26: Repeated content
- **Reviewer:** egm-style (optional trim). **Location:** rules versioning at lines 138-144 and 954-961; reaper guarantee at lines 441 and 938-939; start-up footnote at lines 132-133 and 668.
- **Fix:** Keep one copy of each and point to it from the other places.

---

## Merged ids (for traceability)

- structural-r1-1 is merged into design-r1-6.
- style-r1-21 is merged into design-r1-4.
- style-r1-19 is merged into design-r1-8.
- The line 662 part of style-r1-22 is merged into design-r1-10.
- style-r1-17 and part of design-r1-18 are merged into structural-r1-4.
- Part of design-r1-18 is merged into style-r1-5.
- style-r1-20 is merged into design-r1-15.
- Part of design-r1-18 (line 386) is merged into structural-r1-9.
- style-r1-18 is merged into structural-r1-11.
- style-r1-25 is merged into structural-r1-14.
