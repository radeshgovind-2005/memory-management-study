# Memory Management Study: Go vs Rust

This repository studies how two approaches to memory management, Go's garbage
collector and Rust's ownership model, behave when they run the same server
workload on the same hardware.

> **Status:** the automated design review is complete and human review is
> pending; there are no results yet. The next step is checkpoint C1, which
> verifies the facts the design depends on. Results will appear in `posts/`
> once they exist.

## Background: how languages free memory

Every long-running program allocates memory as it works, and it has to give
that memory back once it no longer needs it, otherwise it eventually runs out.
Languages take three broad approaches to this. In C and C++ the programmer
frees memory explicitly. In Go, Java, C# and JavaScript a runtime component
called the garbage collector periodically traces which memory the program can
still reach and frees the rest, so the programmer does not have to think about
it. Rust takes a third approach called ownership: the compiler tracks which
part of the program owns each value and inserts the code that frees it when the
owner goes out of scope, so no garbage collector runs alongside the program.

None of these approaches is free, and the costs show up in different places. A
garbage collector spends CPU time finding the memory that is still live, and it
usually needs extra memory headroom so that it does not have to run constantly.
Go's collector does most of its work concurrently with the program, so on a
large heap that contains many pointers, most of the cost comes from the CPU
spent marking live objects. Part of that marking is done by the program's own
request-handling goroutines: while a collection is in progress, a goroutine
that allocates memory may be asked to help with the marking before it can
continue, which is called a mark assist and which slows that request down. Go
offers two settings to tune this behaviour, `GOGC` and `GOMEMLIMIT`, and a
program can also reduce the collector's work by storing its data in a layout
that contains no pointers, because the collector then has nothing to follow.

Rust avoids this collection work entirely, but it still allocates and frees
memory, so its cost moves to the memory allocator and to the code that drops
(frees) values when they go out of scope.

## What this project wants to learn

Because the two languages pay for memory management in different places, a
single number such as "Rust is faster" says little about why. This project
therefore compares Go, in several progressively tuned forms, with Rust on one
workload, and looks at where each language's memory-management costs appear:
in tail latency, in CPU time, and in memory use. Tail latency here means the
response time of the slowest requests, reported as percentiles: p99 is the
time within which 99% of requests complete, and p99.9 is the same for 99.9%.
The aim is to measure how much of the difference between Go and Rust remains
once Go uses the tuning and data layouts available to it, and then to explain
the mechanism behind whatever remains.

## The first experiment: a large in-memory cache

The workload is an in-memory LRU cache holding millions of small entries. An
LRU (least recently used) cache has a fixed size limit, here a number of bytes,
and when it is full it makes room by removing the entries that were used least
recently. A cache of this size suits the question well, because it keeps
a very large number of objects alive at once, which is the situation where a
garbage collector has the most to scan. The workload is modelled on a service
that Discord described when it moved that service from Go to Rust ("Read
States"), and the details of that account are checked at C1.

The same HTTP cache is implemented five times. The variants differ in language,
in garbage collector settings, and in how they lay out the cache in memory:

| Rung | Variant | Language | Layout | Garbage collector settings |
| --- | --- | --- | --- | --- |
| 1 | `go-default` | Go | pointer | defaults |
| 2 | `go-tuned` | Go | pointer | `GOGC` and `GOMEMLIMIT` chosen by a fixed sweep |
| 3 | `go-pointerfree` | Go | slab | chosen by its own sweep |
| 4 | `rust-pointer` | Rust | pointer | none (no garbage collector) |
| 5 | `rust-index` | Rust | slab | none (no garbage collector) |

The two layouts are the key to reading this table. In the pointer layout, each
entry is a separate heap allocation linked to its neighbours by pointers, so in
Go the collector has to scan all of those pointers. In the slab layout, entries
are stored in arrays allocated up front and linked to each other by integer
positions in those arrays rather than by pointers, so there is nothing for a
collector to follow. The Go variants form a ladder: each rung applies one more
of the fixes Go offers, so the results show how much each fix recovers. Both
layouts are also implemented in Rust, which makes it possible to separate a
difference caused by the language from a difference caused by the data layout.
For that reason the main comparison is between `rust-index` and
`go-pointerfree`, the two slab-layout variants.

## How it is measured

The variants run on Kubernetes, and k6, a load-testing tool, sends them
requests at three load levels: 25%, 70% and 110% of measured capacity. Capacity
is one request rate shared by all five variants: it is calibrated against the
weakest variant, as the last rate before its p99 latency exceeds four times its
low-load value or k6 starts dropping requests, so every variant receives the
same absolute load at each level. The 110% level deliberately overloads the
cache to show how each variant behaves past its limit. Each run records the
following:

| Metric | Source |
| --- | --- |
| Tail latency (p99, p99.9) | k6 on the client side, and per-second HDR histograms (a compact record of the full latency distribution) on the server side |
| CPU spent on memory management | `perf` sampling on the measured node, with the same counting rule for both languages |
| Lock wait | `perf` sampling, reported as a separate category so it is not mistaken for a memory-management cost |
| Memory per pod, bytes per entry | cgroup working set |
| CPU per 1,000 requests | cgroup CPU accounting |
| Pods per node | calculated from the measured memory and CPU |

Each run is stored as a folder in this repository, together with a manifest
that records the toolchain versions, image digests, node, settings and rules
version, so that any published number can be traced back to the runs that
produced it. Large raw files, such as `perf` profiles, are kept in a
public-read results bucket instead of git, and the run folder points to them.

## Keeping the comparison fair

A benchmark like this can produce a gap between the languages that has nothing
to do with memory management, for example through different build settings,
uneven hardware, or a load generator that hides slow responses. The method is
built to rule these out.

The rules are fixed before anything is measured. Three rule documents, to be
written at checkpoint C2, define the build settings, the metrics and the
procedure: `docs/FAIRNESS.md`, `docs/METRICS.md` and `docs/RUN_PROTOCOL.md`.
They are frozen before the cloud measurements start. The expected outcomes are
written down in `docs/PREDICTIONS.md` before any code exists, so that they can
be compared with the results afterwards.

All five variants are built from one specification. They share one HTTP
contract, one data generator checked against fixed test vectors, and one
conformance suite, and the suite must report identical hit counts for every
variant, which shows that they all do the same work.

The variants also run under the same conditions. Measurements are planned on
dedicated-CPU cloud nodes (the provider is confirmed at C1), with 2 vCPUs per
pod and equal thread counts, and both tuned Go variants go through the same
tuning procedure. Runs are grouped into blocks, where one block runs all five
variants once, in a freshly shuffled order, on the same node. Results are then
compared as paired differences within each block, with confidence intervals, so
that slow drift in the node or the cloud affects every variant in a block
equally instead of favouring one.

The load itself is open-model, meaning k6 sends requests at a fixed rate
regardless of how quickly earlier responses arrive. A load generator that
waited for each response before sending the next would slow down whenever the
server slowed down, which would hide exactly the slow responses this study is
looking for.

Finally, the code is written with AI assistance, so the author annotates the
performance-critical code line by line in
`experiments/cache/HOTPATH_NOTES.md`, and records errors found in generated
code in `docs/AI_LOG.md`. Both files are written at checkpoint C4.

## Stack

Go (`net/http`) and Rust (`hyper`, mimalloc) · Kubernetes on DigitalOcean
(planned; confirmed at C1) · k6 · Prometheus · `perf` · Terraform · Python for
analysis · Datadog for profiling and dashboards.

## Progress and next steps

Work is ordered by checkpoints rather than dates, and each checkpoint builds on
the one before it.

| # | Checkpoint | Status |
| --- | --- | --- |
| — | Design and review | done |
| C1 | Verify the facts the design depends on | next |
| C2 | Write and freeze the rules and predictions | |
| C3 | Specify the cache, conformance suite and test vectors | |
| C4 | Build the five variants | |
| C5 | Build the measurement harness locally | |
| C6 | Reproduce the mechanism locally | |
| C7 | Cloud measurements | |
| C8 | Calculated density | |
| C9 | Datadog profiling | |
| C10 | Methodology write-up | |
| C11 | Cache experiment write-up | |

After this experiment, the plan is to add density and memory-limit tests on
real nodes, variants built on existing cache libraries, and a proxy
experiment.

## Documentation

- [Design document](docs/designs/cache-experiment.md): the decisions,
  alternatives considered, risks and implementation plan.
- [System diagram](docs/designs/system_design.excalidraw), which can be viewed
  at [excalidraw.com](https://excalidraw.com).
- [Design reviews](docs/reviews/): the findings from each review stage and how
  each one was resolved.

## Repository layout

```text
docs/
  designs/       design document, diagram and archived revisions
  brainstorm/    the brief the design started from
  reviews/       design review findings and their resolutions
experiments/
  cache/         this experiment: specification, Go and Rust code, load scripts
harness/         infrastructure, cluster templates, campaign runner, analysis
results/         committed run folders
posts/           methodology page and write-ups
```

## Running it

Build and run instructions will be added at checkpoint C5, when the
measurement harness is built.
