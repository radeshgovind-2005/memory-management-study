# Design Sitemap

| Design | Date | Author | Components | Description |
| --- | --- | --- | --- | --- |
| [Cache experiment](cache-experiment.md) (In Review) | 2026-11-23 (revised 2026-11-24) | Radesh Govind | cache service (Go, Rust), conformance suite, harness (Terraform, Kubernetes, Prometheus, k6), analysis, Datadog | Compares Go's garbage collector (default, tuned and pointer-free layouts) with Rust's ownership model on a large in-memory LRU cache. |
| [Cache experiment, before scope cuts](archive/cache-experiment-before-scope-cuts.md) (archived) | 2026-11-24 | Radesh Govind | — | Earlier revision holding the density-packing and memory-limit designs for a follow-on write-up. |
