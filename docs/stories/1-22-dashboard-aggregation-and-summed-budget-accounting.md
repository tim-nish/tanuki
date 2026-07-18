---
story: 1.22
epic: 1
title: "Story 1.22: dashboard latest-drive aggregation and budget accounting summed at the mining barrier"
status: ready
umbrella: 138
depends_on: [1.21]
---

# Story 1.22: dashboard latest-drive aggregation and budget accounting summed at the mining barrier

Umbrella: https://github.com/tim-nish/tanuki/issues/138 (spec: specs/spec-tanuki-loop/SPEC.md "Concurrent drives within one iteration") — on conflict, the amended spec wins.

As a tanuki operator watching an unattended run,
I want the dashboard's latest-drive section to aggregate per-scenario progress entries (running scenarios with stages, done/total) and the iteration's token/cost accounting to be the sum across concurrent drives folded in at the mining barrier,
So that liveness stays honest and the token-budget breaker sees the true total before the next iter-start.

**Acceptance Criteria:**

- Given `drive_concurrency: N>1` mid-drive, when the dashboard renders, then latest-drive lists each running scenario with its stage and a done/total across the planned set — never a single fabricated "current" scenario.
- Given `drive_concurrency: 1`, when the dashboard renders, then output is unchanged from today.
- Given concurrent drives complete, when the mining barrier folds accounting, then the iteration's recorded token/cost totals are the sum across all drives, and the next iter-start's token-budget breaker evaluates against that sum.
- Given the loop skill's background-liveness rule, when relaying progress lines during a concurrent drive, then the compact line reflects the aggregate (e.g. `[3/6] two running: s2 (80s), s5 (40s)`), sourced from the progress substrate, never re-derived.
