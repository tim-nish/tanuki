---
story: 5.4
epic: 5
title: "Story 5.4: absence verification and quiet-accounting require reach"
status: ready
umbrella: 299
depends_on: [5.3]
---

# Story 5.4: Absence and quiet-accounting require reach

Umbrella: [#299](https://github.com/tim-nish/tanuki/issues/299). Second half of
that issue's decomposition; consumes the `unreached` record story 5.3 writes.

## Story

As **an operator trusting a converged verdict**,
I want **convergence and fix-verification to ignore executions that never
reached the code under test**,
so that **a broken drive cannot manufacture a clean result.**

## Context

The amended invariant (`specs/spec-tanuki-scenario-lifecycle/SPEC.md:170`) makes
absence verification and quiet-accounting — and only those two consumers — read
reach. Before the amendment, the blanket v1 exclusion kept coverage out of every
scheduler computation, which was correct for yield but left this door open: an
execution that ran without reaching its subject advanced verification counters
and could accumulate quiet cycles.

`quota_met` is the existing precedent for "a run cannot manufacture
convergence" — a cycle counts toward the streak only if its persisted plan
reported the exploration quota met. This story adds the sibling guard for a run
that *executed* but never *reached*.

## Acceptance criteria

**AC1 — an unreached execution does not advance absence.**
Given an accepted finding awaiting verification by absence,
When a scenario in its scope runs and is recorded `unreached`,
Then the finding's unseen-runs counter does not advance, and the absence verdict
for that finding is `cannot-determine` rather than progress toward the bar.

**AC2 — an unreached verify set makes the cycle non-quiet.**
Given `record-cycle` computes quietness,
When the iteration's verify-set scenarios ran but were `unreached`,
Then the cycle is not quiet, and the reason states reach — mirroring how
`quota_met` already disqualifies a cycle that skipped exploration.

**AC3 — the reason is stated, never inferred.**
Given a cycle is disqualified for reach,
When `record-cycle` reports `quiet_because`,
Then it names reach explicitly, so an operator reading the output learns why
convergence did not advance without reading source.

**AC4 — yield, streaks and demotion still ignore reach.**
Given the amended invariant preserves the original exclusion for those three,
When `record-run` computes yield and updates streaks,
Then an `unreached` scenario is treated exactly as before: findings it produced
count, and it can still re-promote a regression-pool member.

**AC5 — the conditional invariant is documented where both are computed.**
Given the invariant is now conditional (two consumers read reach, three do not),
When a future reader works on either computation,
Then a comment at each site names which side of the split it is on — the
counter-argument recorded at the decision gate was precisely that a conditional
invariant is easier to misapply than a blanket one.

## Story questions (unverified — not acceptance criteria)

- Does an existing accepted finding's counter need retroactive correction for
  runs already recorded before this lands? The ledger holds 139 accepted
  findings and at least one already reads `cannot-determine` for an unrelated
  reason (F178). Whether to recompute historical counters, or only apply the
  rule forward, was **not** decided at the gate.
- Should `unreached` also feed the dashboard's convergence section as a named
  blocker? Story 5.3's AC4 covers the run summary; the convergence section was
  not specified.
