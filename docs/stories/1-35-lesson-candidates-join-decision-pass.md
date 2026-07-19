---
story: 1.35
epic: 1
title: "Story 1.35: Lesson candidates join the decision pass; accept writes the staging file with in-session receipts"
status: published
depends_on: ["1.34"]
issue: 188
umbrella: 179
---

# Story 1.35: Lesson candidates join the decision pass; accept writes the staging file with in-session receipts

Umbrella: https://github.com/tim-nish/tanuki/issues/179 (spec:
specs/spec-den-distill/SPEC.md §4; amended decision-pass contract in
docs/tanuki-spec.md) — on conflict, the spec wins.

As a tanuki operator finishing an attended run,
I want the decision pass to walk lesson candidates after the proposal walk —
accept / dismiss / defer per item, never a pre-selected default — with
accept writing the staging-intake file right then via the 1.34 emit
operation and printing the written path,
So that one `/tanuki <target>` run goes drive → mine → brief → decisions →
contributed staging proposals with zero steps outside the command.

## Context / decision

Decomposed from #179 (spec lane, ratified 2026-07-19). Exactly parallel to
"accept → optionally file the issue right then — still explicitly
confirmed". Dispositions are written back by the command (`set-status`),
never typed by the user; deferred candidates stay visible in `status`.
`/tanuki-loop` renders candidates in its brief but never walks the gate and
never writes — attended-only, same class as issue filing.

## Acceptance Criteria

- Given a brief with lesson candidates and `contribute_back` configured,
  when the decision pass runs, then candidates are walked per item after the
  proposals, and an accept writes exactly one staging file in-session and
  prints its path.
- Given a dismiss or defer, when chosen, then nothing is written and the
  disposition is recorded via `set-status`; deferred candidates remain
  visible in `status`.
- Given the run summary, when the sitting ends, then contributed items are
  counted alongside filed issues — the operator knows exactly what left
  Tanuki and where it went.
- Given an unattended loop run, when candidates exist, then the brief
  renders them but no gate is walked and nothing is emitted.
- Given no candidates, when the pass runs, then the proposal walk is
  unchanged from today.

## Out of scope

- The standalone "distill den" picker action and the unconfigured one-line
  notice (1.36).
- The emit mechanics themselves (1.34).
