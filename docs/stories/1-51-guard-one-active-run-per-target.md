---
story: 1.51
epic: 1
title: "Story 1.51: Guard one active run per target — init fails closed on a live run, current-resolution refuses when ambiguous"
status: published
issue: 258
---

Canonical discussion record: https://github.com/tim-nish/tanuki/issues/258

**As** an operator (or a headless run) driving `tanuki-loop` on a target whose
`~/.tanuki/<target>/` state is shared machine-wide,
**I want** the loop to treat a target's run as exclusive — refusing a second
`init` while a run is live, and refusing an ambiguous bare subcommand —
**so that** two runs can never silently corrupt each other's
`state.json`/`audit.md` or each other's convergence accounting through the
shared `~/.tanuki/<target>/loop/current` pointer.

Governing invariant (added by this issue's spec decision): **"One active run
per target — guarded at `init`"** in `specs/spec-tanuki-loop/SPEC.md`
(Non-negotiables). This story implements the guard the spec now promises,
following the established base-freshness fail-closed idiom. On any conflict
between this story and the issue, the issue wins.

## Acceptance criteria

- **Given** a target with an unfinished run whose worktree still exists,
  **when** a second `tanuki-loop --target <t> init …` runs, **then** it fails
  closed (the `{"breaker": …}` JSON convention, exit 3), names the live run id
  and its worktree path, writes **no** new run state, and does **not** repoint
  `current`.
- **Given** the same situation, **when** `init` is passed `--allow-concurrent`,
  **then** it proceeds and records the deliberate override in the audit trail.
- **Given** more than one unfinished run exists for a target, **when** a
  state-mutating subcommand (`iter-start`, `iter-verify`, `attempt`, `test`,
  `log`, `record-cycle`) is invoked **without** `--run`, **then** it refuses
  with an `ambiguous — pass --run <id>` message listing the candidate run ids
  and mutates nothing.
- **Given** exactly one unfinished run for a target, **when** a subcommand
  omits `--run`, **then** it resolves via `current` exactly as today — the
  common single-run path is untaxed and still needs only `--target`.
- **Given** a live unfinished run for a target, **when** `tanuki-loop doctor`
  runs for that target, **then** its report includes a live-run check that
  surfaces the conflict before a headless run starts (consistent with doctor's
  other fail-closed readiness checks).
- **Given** the guard lands, **when** the `tools/tests/test-*` suite runs,
  **then** a fixture asserts all three behaviors — second-`init` refused,
  `--allow-concurrent` proceeds and is audited, and an ambiguous bare
  subcommand is refused — so the invariant cannot silently regress.
