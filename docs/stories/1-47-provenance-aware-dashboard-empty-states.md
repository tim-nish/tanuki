---
story: 1.47
epic: 1
title: "Story 1.47: provenance-aware empty-states — manual/ad-hoc run gaps read as expected, not defect"
status: published
issue: 236
umbrella: 236
---

# Story 1.47: provenance-aware dashboard/view empty-states

Issue: https://github.com/tim-nish/tanuki/issues/236 — the canonical
discussion record; on conflict, the issue wins. Umbrella spec decision: #236
(provenance-aware `expected`, ratified 2026-07-20 —
`specs/spec-tanuki-view/SPEC.md` D3 and `specs/spec-tanuki-loop/SPEC.md`
fixed-skeleton point 2).

As a cold reader of the loop dashboard / views,
I want an empty `latest drive` / `scheduler decisions` section to read as
**expected** when the run deliberately drove outside the scheduler, and as a
**gap** only when a scheduled run should have driven and did not,
So that a manual/toy/ad-hoc run's normal emptiness is not misread as a defect.

## Context / decision

Decomposed from #236 (spec lane, ratified 2026-07-20). The empty-state
`expected: true|false` signal must be derived from **run provenance** — the
presence/absence of a persisted plan record for the run — never guessed. A
run with no persisted plan (manual/toy/ad-hoc) resolves its empty
drive/scheduler sections to a distinct enumerated member with `expected:
true` and a reason naming the provenance; a scheduled run that should have
driven and did not stays `expected: false`.

## Acceptance criteria

- **Given** a run with no persisted plan record, **when** the dashboard/view
  renders its empty `latest drive` / `scheduler decisions` sections, **then**
  each resolves to an enumerated member with `expected: true` and a reason
  naming the provenance (e.g. "this run drove outside the scheduler — no plan
  was persisted").
- **Given** a scheduled run that should have driven and did not, **when** the
  same sections render empty, **then** they resolve to `expected: false` (a
  real GAP) as today.
- Provenance is derived only from persisted artifacts (plan-record presence),
  never inferred from model memory.
- One test fixture per new enumerated member; a regression fails visibly.
- Both surfaces (spec-tanuki-view D3 and the spec-tanuki-loop fixed skeleton)
  share the one rule.

## Out of scope

- Applying the empty-state convention to `status` settlement fields (story
  1.46 / #240, which depends on this).
- Refactoring `tools/tanuki-scheduler` (spec-tanuki-view D4: out of scope).
