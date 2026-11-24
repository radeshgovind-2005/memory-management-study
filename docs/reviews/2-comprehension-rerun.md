# Goldfish Review Report: Stage 6, Comprehension Test (re-audit)

**Target:** `docs/designs/big_cache.md` (FeatureRange 1-968, DetailedRange 969-1407)
**Reviewer:** `egm-comprehension` (fresh context, no other files read). **Mode:** REVIEW.
**Verdict:** passes the self-containment test. 0 Blockers, 3 Major, 13 Nits. Almost every project-specific term is defined in the document.

## What this proposes

The project rebuilds the workload behind Discord's Read States move from Go to Rust: an in-memory LRU cache with millions of small entries, mostly reads, and Zipf-distributed (skewed, a few hot) keys. It then measures what actually changes. The same HTTP cache contract is built five times as a "ladder":
- Go default.
- Go with GOGC and GOMEMLIMIT tuned by a fixed sweep.
- Go with a pointer-free "slab" layout (fixed records plus size-class byte slabs, linked by integer indexes).
- A Rust twin of the pointer layout.
- A Rust twin of the slab layout.

Because each layout has a twin in the other language, layout effects are kept apart from language effects. All variants run one at a time on one dedicated-CPU Kubernetes node, under open-model k6 load, at five load levels (25/50/70/85/110% of the weakest variant's capacity) and four entry counts (1M to 10M). Each run lands in a committed, self-describing run folder. Rules and predictions are frozen before the results exist, and analysis uses paired within-block differences with bootstrap confidence intervals. A second campaign on the same images measures pod density and how each variant approaches an OOM kill. Datadog is used only to explain and illustrate. The output is a methodology page and one blog post, used as a portfolio piece.

## How the existing system works, per this document

There is no existing codebase; the repository is created from scratch. The "existing system" is Discord's Go Read States service, which per the document stored per-user, per-channel read markers and showed periodic latency spikes blamed on garbage collection (facts still to be verified at C1). The document argues that in current Go the cost on a large, pointer-heavy heap is mostly concurrent mark CPU and mark assists that slow request goroutines, not stop-the-world pauses. A forced GC roughly every 2 minutes may drive periodic spikes on a heap that rarely changes. Modern Go also offers a new pacer, GOMEMLIMIT and pointer-free layouts as fixes, so a fair comparison has to include them.

---

## Findings

### Major

1. **The "reference" in the paired differences is never named** (component 10, lines 406-413; METRICS.md item, lines 1063-1066; D9). A reader cannot tell which differences the headline charts report. *Fix:* list the comparisons that get confidence intervals (for example each Go rung vs go-default; rust-pointer vs go-tuned; rust-index vs go-pointerfree) and mark the headline.
2. **The source of client-side latency numbers is ambiguous** (components 3, 6, 7; items 54, 58). Prometheus carries k6 per-second statistics but is "not the source for latency percentiles"; run-level p99.9 cannot be built from per-second percentiles. *Fix:* name the artefact that produces run-level client-side p99/p99.9, and say what the per-second series is for.
3. **Calibration's "low-load value" is undefined** (Run protocol step 1, lines 531-537; item 38; local LEVEL note). Every load level derives from capacity. *Fix:* define the low-load reference (starting rate, duration, client or server side) and the ramp's step size and hold time.

### Nits

4. **"Headline configuration" used before it is defined** (Goal 2 line 104, D9, D11; defined at line 582). *Fix:* define inline at first use.
5. **"Block" used before it is defined** (CUJ 1 line 233, Goal 2; defined at line 539). *Fix:* parenthetical at first use.
6. **Which C1 items exist is not stated** (checkpoint table, line 183). *Fix:* say C1 items are every "(needs verification)" marker plus every Open Question.
7. **Sampler-overhead scope and checkpoint disagree** (line 395 and Verification Plan vs Budget; C7 vs C9). *Fix:* one scope and one checkpoint everywhere.
8. **Campaign file list misses the 4-core check** (item 42). *Fix:* add it.
9. **"FAIRNESS version" vs "rules version"** (CUJ 1, items 56, 58 vs Goal 6, NFR). *Fix:* "rules version" everywhere.
10. **CUJ 1's precondition cannot hold for the first session** (line 227). *Fix:* say CUJ 1 covers sessions after the smoke test.
11. **Latency target "written into rules v0" at C6, but v0 is committed at C2** (lines 560-561). *Fix:* "added as a dated v0 amendment at C6".
12. **Applicability matrix labels undefined** (line 1061). *Fix:* define Primary and Secondary.
13. **The Rust tuning alternative is vague** (lines 1039-1040). *Fix:* say whether it is run, with which settings, and where reported, or drop it.
14. **Thread count for fractional CPU in density** (sizing table, D3). *Fix:* state the rounding rule.
15. **"DOKS" and "static CPU manager" unexplained** (lines 1271, 217). *Fix:* expand DOKS; explain the static CPU manager (exclusive cores for Guaranteed pods with whole-number CPU).
16. **Rung 4 description** (line 34) says memory is freed "when values go out of scope". *Fix:* "freed explicitly at eviction, with no garbage collector".
