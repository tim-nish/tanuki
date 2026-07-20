---
story: 1.45
epic: 1
title: "Story 1.45: init warns on a non-default/untracked base and records a resilient base ref"
status: published
issue: 239
---

# Story 1.45: init base resolution resilient to a transient checkout

Issue: https://github.com/tim-nish/tanuki/issues/239 — the canonical
discussion record; on conflict, the issue wins.

As a tanuki-loop operator,
I want init to warn when the base it derived is not the repo's default or
upstream-tracked branch, and to record a base ref that survives to gate-pr,
So that a concurrent session's transient checkout at init time does not later
break `gate-pr` with "Base ref must be a branch".

## Context / decision

`init`'s default base is whatever branch is checked out at init time. A
transient branch recorded as `state.base` may no longer resolve by the time
`gate-pr` runs. Relates to the `--base origin/<branch>` gate-pr rejection
(F128). This story hardens the default; it does not change the meaning of an
explicit `--base`.

## Acceptance criteria

- **Given** init runs while a non-default, non-upstream-tracked branch is
  checked out, **when** the base is derived, **then** init emits a warning
  naming the derived base and that it is neither the default branch nor
  upstream-tracked.
- **Given** a derived base, **when** init records `state.base`, **then** the
  recorded ref is resilient enough that a later `gate-pr` can resolve it (or
  the recorded base is surfaced prominently so the operator can correct it
  before delivery).
- **Given** an explicit `--base`, **when** init runs, **then** behavior is
  unchanged (the operator's named ref wins, no warning about their choice).
- Tests cover the transient-checkout warning path and the explicit-`--base`
  no-warning path.

## Out of scope

- The gate-pr `--base origin/<branch>` handling itself (tracked separately).
