# Goldfish Review Report: Stage 6, Comprehension Test

**Target:** `docs/designs/big_cache.md` (FeatureRange 1-845, DetailedRange 846-1272)
**Reviewer:** `egm-comprehension` (fresh context, no other files read). **Mode:** REVIEW. No document standard.
**Result:** 0 Blockers, 7 Major, 11 Nits. The reviewer could paraphrase every section.

## What this proposes

The project builds a benchmark to test the claim that Rust beats Go for large in-memory caches because it has no garbage collector. The claim comes from Discord's "Read States" migration. One HTTP LRU-cache contract is written five times as a "ladder":
- Go rungs: default Go, Go tuned with GOGC and GOMEMLIMIT, and Go with a pointer-free slab layout.
- Rust rungs: one with the same pointer-linked layout as Go, and one with the same slab layout.

Each layout has a twin in the other language, so a difference can be blamed on the language rather than the layout. The variants run one at a time on a single dedicated-CPU Kubernetes node. Load comes from k6 in open-model mode at four load levels plus an overload level, and the cache is tested at 1M, 2M, 5M and 10M entries. Runs go in shuffled blocks, and each result is the paired difference within a block, with bootstrap confidence intervals. Latency is measured on both the client and server side, GC is tracked through gctrace, and a perf-based CPU sampler splits CPU into memory-management and lock-wait shares. Each run goes into a committed folder with a manifest. A second campaign measures pods per node and behaviour as memory approaches the pod limit. Rules are frozen in FAIRNESS.md and METRICS.md before any results. Datadog is used only to explain results, never as the source of a published number. The output is a methodology page and a blog post.

## How the existing system works, per this document

Nothing exists yet: the repository is not created, and every file in the plan is new. The "existing system" is the outside world the design responds to. Discord's Go service showed periodic latency spikes that were blamed on GC (still to be verified). Modern Go's stop-the-world pauses are short; on a pointer-heavy heap the real cost is CPU spent on concurrent marking and mark assists that slow request goroutines. Go has since added a new GC pacer, GOMEMLIMIT and container-aware GOMAXPROCS, and Go practitioners answer large caches with pointer-free layouts. Go's runtime also forces a GC roughly every 2 minutes (to be verified for the pinned version).

---

## Findings

### Major

1. **Checkpoints C1–C11 are never listed** (lines 131, 153-167, 575, 810-812, 849, 884-1234, 1256-1266). The reader must piece them together; C8 is never mentioned; C7 is described both as a starting gate and as a finished state. *Fix:* add a checkpoint table (number, name, exit criterion, files or campaigns produced), including C8.
2. **The latency target is undefined** (lines 494-496, 549, 557, 929). "p99 under the target" has no number or rule; "degraded" and "overloaded" are not told apart; density pass/fail and the ramp's "first sign of distress" depend on it. *Fix:* define the target (for example a multiple of low-load p99, or fixed milliseconds per entry count), define each overload category in one line, and say which rule file holds it.
3. **The pod CPU limit for the main sweeps is missing** (sizing table, lines 437-449; D3, line 586; component 5, line 292). Thread counts are tied to it, but only memory is sized. *Fix:* add a CPU column to the sizing table and relate it to the node size.
4. **go-tuned's settings are circular and the selection rule is missing** (line 441; lines 910-913; line 519). The pre-run that sets the pod limit needs a GOMEMLIMIT, which needs the limit; the tuning sweep's own pod limit is unstated; no rule picks the winning GOGC/GOMEMLIMIT pair. *Fix:* give the order of operations and the selection rule, fixed in FAIRNESS before results.
5. **Global versus per-shard byte bound** (lines 408-415 compared with 430-431 and 982-983). The contract reads as one global bound; the simulation uses per-shard bounds. They give different hit ratios and need different implementations. *Fix:* state that each shard's bound is `CACHE_MAX_BYTES / 64` (or how a global bound is enforced) and use the same wording in all three places.
6. **What freezes when is unclear** (Goal 6, lines 112-116; NFR, lines 810-816; Open Questions, lines 160-162). It is unclear whether METRICS.md freezes at C2 or is amendable until C7, and which version number covers it. *Fix:* one sentence listing PREDICTIONS, METRICS, FAIRNESS and RUN_PROTOCOL with their freeze points and version numbers.
7. **Ramp mechanics are loose** (lines 443, 448-449, 551-560, 1116-1118). "Until the pod is killed" conflicts with "50% to 110%"; "GOMEMLIMIT can stop growth" is wrong (it is a soft GC target); the CPU-share threshold is unnamed. *Fix:* keep writing until the kill with a timeout that marks "no kill"; reword the GOMEMLIMIT sentence; name the threshold.

### Nits

8. **Many names for each layout** (lines 21-31, 389-403, 997-999, 1047-1048, 1099). *Fix:* a table of rung, variant name, language, `CACHE_LAYOUT` value and layout name; one layout name in prose.
9. **Go's forced GC is not explained** (lines 165, 482). *Fix:* one sentence on what it is and why the window must cover it.
10. **Calibration scope is unclear** (lines 471-475). *Fix:* state which variants are calibrated and when.
11. **Go server clock with keep-alive** (lines 310-313, 1004-1008, 1029-1031). *Fix:* say the stamp resets on the first read after each response.
12. **Process jargon** (EGM, Stage 5, Goldfish; lines 5, 820-823, 954, 1272). *Fix:* a one-line gloss, or move to the footer.
13. **"SUT" not expanded at first use** (line 140). *Fix:* "system under test (SUT)".
14. **Pingora and Linkerd mentioned as if known** (line 129; "reviewers" at line 735 unexplained). *Fix:* half a sentence or a forward reference to Alternative M.
15. **Allowed values of `<env>` not listed** (lines 246, 347, 1228). *Fix:* list them.
16. **The Discord Read States service is not described** (lines 14, 825). *Fix:* one sentence on what it does and how the workload maps to it, or a note that the mapping is confirmed at C1.
17. **"inferno" not glossed** (line 324). *Fix:* "inferno (a Rust port of the flamegraph stack-folding scripts)".
18. **Local `LEVEL=50` has no reference capacity** (line 1253). *Fix:* state the local rule.
