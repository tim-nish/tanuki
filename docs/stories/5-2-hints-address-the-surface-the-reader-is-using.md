---
story: 5.2
epic: 5
title: "Story 5.2: Operator-facing hints instruct the user to run internal tools directly, with no rule fixing which surface a hint may address"
status: published
issue: 311
umbrella: 311
---

# Story 5.2: Hints address the surface the reader is using

Canonical discussion record: [#311](https://github.com/tim-nish/tanuki/issues/311).
On conflict, the issue wins.

## Story

As **an operator following Tanuki's own guidance**,
I want **every hint to name an act on the surface I am actually using**,
so that **following the tool's advice never dead-ends on a command I cannot
find or cannot run as written.**

## Context — the rule already exists

The issue proposes settling a contract. It is already settled, and the code does
not conform:

> Replace it with a derivation over an explicit, closed enumeration of ledger
> states … each mapping to exactly one next command (**a `/tanuki` form where
> one exists, the tool form otherwise**). One test fixture per state in
> `tools/tests/`, so a new state or a regression fails visibly.
> — `specs/spec-short-command-surface/SPEC.md:141` (D2)

The discoverability half is likewise ratified: D3, "help text carries the
contract" (`specs/spec-short-command-surface/SPEC.md:153`).

So this story is **conformance work, not a contract change** — which is why the
lint is the load-bearing part: six recorded findings (F24 rec 8, F36, F76, F84,
F90, F130/F120) oscillated *against* an existing rule, each fix trading one
unreachable reference for another. Nothing mechanical was checking it.

## Acceptance criteria

**AC1 — every enumerated state names its `/tanuki` form where one exists.**
Given a ledger state whose next act is reachable through the command surface,
When `next` or `status` renders its hint,
Then the `/tanuki` form is the named command, not a parenthetical. Two states
violate this today: `tools/tanuki-ledger:1519` and `tools/tanuki-ledger:1542`
both lead with `tanuki-drive` and demote `/tanuki` to a trailing aside.

**AC2 — nothing presented as runnable carries unresolved placeholders.**
Given a hint renders a command line,
When the operator copies it,
Then it runs as written — every argument the tool can derive from state is
derived (the target is known; the scenarios path is canonical). Where a value
genuinely cannot be derived, the hint describes the act instead of presenting a
command that only looks pasteable.
*Known instance, not on `main`:* commit `1d0b680` on branch
`tanuki-loop/tanuki/20260726-204018` prints
`tanuki-drive --target <t> --plugin <repo> --scenarios <file> --run <run> --only …`
per cannot-determine finding. That commit is undelivered, so this criterion is
satisfied either by amending it before delivery or by correcting it after.

**AC3 — a foreign scenario is never suggested as a drive target.**
Given an accepted finding's evidence names a scenario absent from this target's
matrix,
When a hint states how to resolve it,
Then it says the scenario is not in this target's matrix rather than emitting a
`--only` command that cannot succeed. Observed with F178, whose scenarios
(`draft-postmortem-f2`, `draft-survey`) belong to the `writing-assistant`
matrix.

**AC4 — `--help` names the wrapper it assumes.**
Given a hint names a `/tanuki` form,
When the operator reads `tanuki-ledger --help`,
Then the slash-command surface is named there, per D3 — closing F84's original
dead end rather than re-creating it.

**AC5 — a lint enforces AC1–AC3 mechanically.**
Given the repo's tracked tree,
When the lint runs in CI,
Then a hint string that names a bare sibling tool where a `/tanuki` form exists,
or that contains an unresolved `<placeholder>` in a presented command, fails the
check. `tools/check-publication-boundary` is the shipped precedent for a
mechanical rule over the tracked tree.

**AC6 — the per-state fixtures are updated, not worked around.**
Given `tools/tests/test-ledger-next` currently pins the violating prose,
When AC1 lands,
Then those fixtures are updated to pin the conforming text — a fixture that
pins prose this story legitimately improves is a fixture to update.

## Story questions (unverified — not acceptance criteria)

- How many enumerated states name a tool form where a `/tanuki` form exists?
  Two were located by inspection (`:1519`, `:1542`); the full enumeration was
  **not** walked this sitting (cannot-determine — not read). Additional
  tool-form mentions exist at `tools/tanuki-ledger:1348`, `:1356` and `:2330`
  in absence/compaction prose and need judging individually.
- Should the lint cover `tanuki-loop` and `tanuki-scheduler` output too? F174
  (dashboard naming a nonexistent `reconcile` subcommand) suggests yes, but the
  scope was not decided.

## Out of scope

- Whether `tanuki-ledger` may name a `/tanuki` form at all. D2 already answers
  it ("a `/tanuki` form where one exists, the tool form otherwise"); this story
  conforms to that answer rather than reopening it.
