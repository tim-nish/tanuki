---
story: 1.38
epic: 1
title: "Story 1.38: [tool] deterministic disposition + port engine (no owner decision; fail closed on missing evidence)"
status: published
issue: 211
umbrella: 210
---

# Story 1.38: Deterministic disposition + port engine

Issue: https://github.com/tim-nish/tanuki/issues/211 — the canonical
discussion record; on conflict, the issue wins. Umbrella spec decision:
#210 (terminal reconcile contract, ratified 2026-07-19 —
`specs/spec-tanuki-loop/SPEC.md`, Non-negotiables "A declined gate leaves a
debt" + "Reconcile substrate" transition note).

As a tanuki-loop operator running reconcile,
I want every port and disposition that requires no owner semantic decision
applied by the tool inside a single reconcile invocation,
So that the attended sitting contains only genuine decisions instead of
mechanical hand-porting (run `loop-20260719-173009` needed every port done
by hand).

## Context / decision

Decomposed from #210 (spec lane, ratified 2026-07-19). "Deterministic"
means **requires no owner semantic decision** — NOT "algorithmically
decidable": the engine may perform agent-driven hunk isolation and
adaptation within the invocation. The hard rule: **missing behavioral
evidence fails closed** — the unit escalates to the gate as `needs-owner`,
never guessed or applied on a hunch. Today `tools/tanuki-loop` has only
`unresolved` (discovery/signals); there is zero reconcile execution code.

## Acceptance criteria

- **Given** an unresolved semantic change unit whose behavior the base
  already exhibits, **when** the engine classifies it, **then** it is
  dropped as already-landed/superseded.
- **Given** a still-applicable unit that cherry-picks cleanly with one
  shared verdict across hunks, **when** applied, **then** it lands as an
  atomic cherry-pick; **given** an entangled commit (F72: `SKILL.md` hunk
  already landed, two validator hunks genuinely missing; F10: one
  applicable hunk in a 3-finding commit), **when** applied, **then** only
  the applicable hunks land and every dropped hunk is named.
- **Given** a conflicting, ambiguous-supersession, or spec-lane unit,
  **when** classified, **then** it is NOT applied — it escalates to the
  single gate with the governing evidence.
- **Given** a unit whose behavioral evidence is missing, **when**
  classified, **then** it fails closed to `needs-owner`.
- The engine does not own the gate, merge, push, or cleanup (those are
  story 1.40 / #213 scope).
