---
story: 5.5
epic: 5
title: "Story 5.5: gate-check's gate_progress reports steps the recorded state cannot prove"
status: published
issue: 301
umbrella: 301
---

# Story 5.5: The issues step declares its applicability

Canonical discussion record: [#301](https://github.com/tim-nish/tanuki/issues/301).
On conflict, the issue wins.

> **Scope note.** #301 named two defects. The first — `last_completed_step`
> reporting `cleanup` before any cleanup — **was already fixed** on `main` in
> commit `ec4d516`, verified by running `_gate_progress` against a
> finished-but-unlanded state (it returns `final-tests`). This story covers only
> the second: `next_step` reporting `issues` on a target the step does not apply
> to.

## Story

As **an operator picking the delivery sequence back up hours later**,
I want **`gate_progress` to tell me the next step that actually applies to my
target**,
so that **I am never nudged toward a step my target skips, and never left
guessing which of two signals to believe.**

## Context

`_gate_progress` (`tools/tanuki-loop:3771`) derives the operator's position in
the delivery sequence from recorded state. It returns `next_step: "issues"` on
every target that has landed, and attaches a note saying the step may not apply
— because, as the code states at `tools/tanuki-loop:3814`,
`# F234: this derivation cannot know whether issue tooling exists for the
target`.

That is an honest refusal, and the fix is not to guess better. Both candidate
signals were checked at triage and both are unsound: `gate` mode describes
delivery rather than tracking, and hostlessness does not imply tracklessness
(this repo is hostless and has a tracker). `specs/spec-tanuki-loop/SPEC.md` was
amended in the same sitting to make applicability **declared, never inferred**,
with undeclared resolving to cannot-determine.

## Acceptance criteria

**AC1 — the target can declare it.**
Given a scenarios file's `loop` block,
When it declares whether an issue tracker is configured,
Then `tanuki-loop` reads that declaration into run state at `init`, alongside
the other `loop` settings.

**AC2 — a declared-false target skips the step.**
Given a target that declares no issue tracker,
When `gate_progress` derives the position after `landed`,
Then `next_step` is `cleanup`, and no `next_step_note` about applicability is
emitted — there is nothing undetermined to report.

**AC3 — a declared-true target names the step plainly.**
Given a target that declares an issue tracker,
When `gate_progress` derives the position after `landed`,
Then `next_step` is `issues` with no applicability note.

**AC4 — undeclared stays cannot-determine, and says so.**
Given a target that declares nothing (every target today),
When `gate_progress` derives the position after `landed`,
Then the undetermined state is carried as a first-class value rather than an
`issues` position that reads as applicable, and the output names the
declaration that would resolve it. Today's behaviour — `next_step: "issues"`
plus `next_step_note` (`tools/tanuki-loop:3814`) — is the starting point this
criterion revises, not preserves.

**AC5 — `doctor` surfaces the undeclared state once.**
Given an operator runs `tanuki-loop doctor`,
When the target has not declared tracker availability,
Then the report names it as undeclared and shows the key to set — so the
operator can close it once rather than meeting the undetermined state at every
gate.

**AC6 — no inference is introduced.**
Given the amended spec forbids standing in `gate` mode or host presence for the
declaration,
When the derivation runs,
Then it reads only the declaration, and a test pins that a `branch`-gate target
which declares a tracker still gets `next_step: issues`.

## Story questions (unverified — not acceptance criteria)

- What is the key's name and shape in the `loop` block? Not decided at the
  gate. `issue_tracker: true|false` is the obvious form, but whether it should
  instead name the tracker (allowing a non-GitHub value later) is open.
- Should `gate-check`'s JSON carry the undetermined state as a distinct field
  rather than a note string? AC4 requires it be first-class but does not fix the
  shape; the three-valued absence work
  (`specs/spec-tanuki-scenario-lifecycle/SPEC.md`) is the nearest precedent and
  was **not** compared in detail this sitting (cannot-determine — not read).
- Does anything else consume `next_step` such that a new value breaks it?
  Unverified — the consumers were not enumerated.

## Out of scope

- The `last_completed_step` contiguous-prefix behaviour: already fixed
  (`ec4d516`) and verified.
