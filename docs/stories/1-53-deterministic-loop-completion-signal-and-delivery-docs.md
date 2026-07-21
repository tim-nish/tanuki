---
story: 1.53
epic: 1
title: "Story 1.53: Deterministic loop completion signal + PR-as-default delivery docs"
status: ready
umbrella: 263
depends_on:
  - 1.52
---

Canonical discussion record: https://github.com/tim-nish/tanuki/issues/262
(deterministic completion signal), under the #263 umbrella. On any conflict
between this story and the issue, the issue wins.

Governing invariant (added by the 2026-07-21 spec decision, triage of
#262/#263): **"Two-outcomes-only delivery"** in `specs/spec-tanuki-loop/SPEC.md`
— "The completion signal is deterministic". This story implements the
completion-signal half of #262 and the documentation of the new delivery
contract. It **depends on 1.52** (the two-outcomes delivery model must exist
first).

**As** an operator (or an automation) reading a loop run's output,
**I want** a normal completion to always leave a fixed-shape terminal artifact
and any other exit to be clearly marked,
**so that** a completed run is unambiguously distinguishable from one that
crashed or stopped early — the absence of the expected artifact means "did not
complete normally", never a silent state identical to success.

## Acceptance criteria

- **Given** a run that completes normally (`stop_reason` `cap|converged`),
  **when** it ends, **then** it has **always** produced the fixed-shape
  terminal artifact for its gate — a PR opened (`gate: "pr"`) or a recorded
  branch-only terminal fact (`gate: "branch"`) — never a best-effort/
  inconsistent result.
- **Given** a run that aborts, errors, or stops early, **when** its output is
  read, **then** it is distinguishable from a normal completion from the output
  alone (stop reason ≠ `cap|converged`, or no recorded terminal delivery) — it
  is never left as a silent "no PR" state identical to a clean completion.
- **Given** the completion signal, **when** any consumer (dashboard, `status`,
  an automation) inspects a run, **then** presence/absence of the terminal
  artifact is a reliable status signal — inconsistent PR creation across
  repeated runs is treated as a defect, not tolerated.
- **Given** the docs, **when** an operator reads `commands/tanuki-loop.md`,
  the README, and the CHANGELOG, **then** they state PR-as-default, the
  two-outcomes-only delivery contract, and the completion-signal contract
  (absence of the terminal artifact ⇒ not a normal completion).
- **Given** the change lands, **when** the `tools/tests/test-*` suite runs,
  **then** a fixture asserts a normally-completed run and an aborted/errored
  run are distinguishable from their recorded output alone.
