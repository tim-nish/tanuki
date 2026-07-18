---
story: 1.18
epic: 1
title: "Story 1.18: doctor warns when a declared entry fixture's provenance lags the live host"
status: ready
umbrella: 136
depends_on: [1.16]
---

# Story 1.18: doctor warns when a declared entry fixture's provenance lags the live host

Umbrella: https://github.com/tim-nish/tanuki/issues/136 (spec: specs/spec-host-snapshot/SPEC.md "Stage-entry fixtures") — on conflict, the amended spec wins.

As a tanuki operator preparing an unattended run,
I want `doctor` to compare each declared entry fixture's `{source_commit, producing_stage_version}` against the live host tip and warn when either has moved,
So that I know before a run that its stage-entry state may no longer match what the current pipeline would produce — without staleness ever blocking the run.

**Acceptance Criteria:**

- Given a declared fixture whose provenance matches the host tip, when `doctor` runs, then the fixture check reports ok with the provenance shown.
- Given a fixture whose source commit or producing-stage version lags the live host, when `doctor` runs, then the report carries a warning naming the fixture, both commits, and what moved — informational, `ready` unaffected (same posture as host drift).
- Given a declared fixture that cannot be materialized at all, when `doctor` runs, then that is a failing check (init would fail closed on it).
- Given a hostless target or a scenarios file with no `entry_fixture` keys, when `doctor` runs, then the check reports its typed skip/empty state, never silently omitted.
