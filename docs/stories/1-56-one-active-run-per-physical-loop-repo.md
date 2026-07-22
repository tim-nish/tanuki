---
story: 1.56
epic: 1
title: "Story 1.56: One active run per physical --loop-repo, not just per target"
status: published
issue: 286
---

Canonical discussion record: https://github.com/tim-nish/tanuki/issues/286
On any conflict between this story and the issue, the issue wins.

Governing invariant (added by the 2026-07-22 spec decision, triage of #286):
specs/spec-tanuki-loop/SPEC.md's "Non-negotiables" section, new bullet "One
active run per physical `--loop-repo`, not just per target" — extending the
existing per-target guard (SPEC.md:143-160, triage of #258) to also cover two
*different* targets sharing one physical repo checkout.

**As** an operator running loops across multiple targets,
**I want** `init` to refuse a second run against a physical `--loop-repo`
checkout that another target's unfinished run already holds,
**so that** distinct target slugs alone are never mistaken for safety when
they share one underlying git repo.

## Acceptance criteria

- **Given** target A has an unfinished run (worktree still present) against
  `--loop-repo` path P, **when** `init` is run for a *different* target B
  also naming `--loop-repo` path P (after resolving both to the same real
  path), **then** `init` refuses with a breaker (exit 3), mirroring the
  existing per-target refusal at `tools/tanuki-loop:2318` (`cmd_init`) and
  the fail-closed idiom at `specs/spec-tanuki-loop/SPEC.md:154-158`.
- **Given** the same setup, **when** `--allow-concurrent` is passed to
  target B's `init`, **then** it proceeds and the concurrency is recorded in
  the audit — the same convention already used for the per-target case
  (`specs/spec-tanuki-loop/SPEC.md:157`).
- **Given** no other target holds a live, worktree-present run against the
  resolved `--loop-repo` path, **when** `init` runs, **then** behavior is
  unchanged from today — no new refusal, no new prompt, for the common
  one-target-per-repo case.
- **Given** the cross-target detection needs a record of which target holds
  which resolved repo path, **then** that record lives under `~/.tanuki/`
  (never inside the target repo), consistent with the existing per-target
  convention at `tools/tanuki-loop:237` (`unfinished_runs`), which already
  scopes its check to a target's own `~/.tanuki/<target>/loop/` directory and
  would need a sibling, cross-target index to extend the check by repo path.

### Story question (unverified — mechanism not located)

No existing cross-target index structure was found to cite as precedent —
`unfinished_runs()` (`tools/tanuki-loop:237`) only scans one target's own
directory. The exact registry shape (a single shared JSON keyed by resolved
repo path vs. a per-repo lock file vs. scanning every target's `state.json`)
is left to the implementer's judgment; the acceptance criteria above bind
only to the observable behavior (refusal / `--allow-concurrent` override /
no-op in the common case), not to a specific storage mechanism.
