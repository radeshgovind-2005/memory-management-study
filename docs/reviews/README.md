# Design reviews

Before any code was written, the [design](../designs/cache-experiment.md) went
through a staged review. At each stage, reviewers who had no access to the
earlier conversation read the document on its own and reported what was
missing, unclear or wrong. The author then decided, finding by finding, whether
to fix it, and recorded those decisions. This folder keeps both: what the
reviewers found, and how each finding was handled.

The reports are a historical record. They refer to the design by its working
name at the time, `docs/designs/big_cache.md` (now
[`cache-experiment.md`](../designs/cache-experiment.md)), and their line
numbers point to the revision they reviewed, not to the current file.

## Stages

| Stage | What the reviewers checked | Findings | Resolution |
| --- | --- | --- | --- |
| 1. Zoom out | Whether the design's scope and mechanisms fit its goal | [first pass](1-zoom-out.md), [re-check](1-zoom-out-rerun.md) | applied directly to the design |
| 2. Comprehension | Whether a reader with no context can understand the document on its own | [first pass](2-comprehension.md), [re-check](2-comprehension-rerun.md) | applied directly to the design |
| 3. Critic | Architecture, structure against the feature-spec standard, and writing | [round 1](3-critic.md), [round 2](3-critic-round2.md) | [round 1](resolutions/3-critic.md), [round 2](resolutions/3-critic-round2.md) |
| 4. Intermediate zoom out | Whether the fixes so far had pushed the design past its goal | [report](4-intermediate-zoom-out.md) | scope cuts applied; the pre-cut version is [archived](../designs/archive/cache-experiment-before-scope-cuts.md) |
| 5. Implementation readiness | Whether the plan can be built without the implementer making decisions that matter | [round 1](5-readiness.md), [round 2](5-readiness-round2.md) | [round 1](resolutions/5-readiness.md), [round 2](resolutions/5-readiness-round2.md) |

The idea for the project, and the first round of review on it, are in the
[brainstorm brief](../brainstorm/concepts_brief.md).
