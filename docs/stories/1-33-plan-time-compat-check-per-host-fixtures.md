---
story: 1.33
epic: 1
title: "Story 1.33: Plan-time compatibility check (compat_skipped, quota rule) and host-aware entry fixtures"
status: published
depends_on: ["1.31", "1.32"]
issue: 186
umbrella: 173
---

# Story 1.33: Plan-time compatibility check (compat_skipped, quota rule) and host-aware entry fixtures

Umbrella: https://github.com/tim-nish/tanuki/issues/173 (spec:
specs/spec-tanuki-scenario-lifecycle/SPEC.md "Host bindings & declared-input
configuration") — on conflict, the issue and spec win, issue first.

As a tanuki operator driving against a non-default host,
I want the compatibility check to run where selection happens —
`tanuki-scheduler plan` (and loop init) filters scenarios whose declared
requirements are not established on the effective host, persisting the skips
with reasons — and the `entry_fixtures` registry to select per-host path
sets or mark a recipe invalid for hosts lacking the committed artifacts,
So that an incompatible host produces a surfaced, accounted-for plan instead
of a confusing partial run, and can never manufacture loop convergence.

## Context / decision

Decomposed from #173 (spec lane, ratified 2026-07-19). The accounting rules
are contract, not implementation detail: a compat-skipped verify replay
accrues no driven-absence (driven-absence already requires an actual drive —
stated so no implementation weakens it), and if the reserved exploration
slot's scenario is compat-skipped with no compatible unexplored fill,
`quota_met` is false — the run cannot count toward the quiet streak.
`tanuki-drive` keeps a last-line fail-closed check (the missing-declared-
fixture posture) for anything that slips past planning.

## Acceptance Criteria

- Given a plan against a host missing a scenario's declared requirements,
  when `plan` runs, then the scenario is filtered and the persisted plan
  record carries `compat_skipped: [{scenario, unmet}]` — surfaced, never
  silent.
- Given a compat-skipped scenario in the verify set, when compaction later
  evaluates the affected finding, then no driven-absence accrued from that
  run.
- Given the exploration slot's scenario compat-skipped with no compatible
  unexplored replacement, when the plan record is written, then `quota_met`
  is false and the loop's quiet-streak gating reflects it.
- Given an `entry_fixtures` recipe with per-host path sets, when planned
  under a host, then the matching set is selected and recorded in the
  manifest with the existing `{name, paths, source_commit,
  producing_stage_version}` provenance; a recipe invalid for the host is
  reported at plan time, never discovered mid-drive.
- Given a fully compatible host, when `plan` runs, then output is unchanged
  from today (`compat_skipped` empty or absent).

## Out of scope

- The coupling report itself (1.32).
- Cross-host comparison methodology (#163's probe axis governs that,
  stories 1.24–1.26).
