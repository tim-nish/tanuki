---
story: 1.24
epic: 1
title: "Story 1.24: Charters declare a probe block; authoring goes through the plan gate"
status: published
issue: 164
umbrella: 163
---

# Story 1.24: Charters declare a probe block; authoring goes through the plan gate

Umbrella: https://github.com/tim-nish/tanuki/issues/163 (spec: specs/spec-tanuki-scenario-lifecycle/SPEC.md "Charter probe block", specs/spec-tanuki-trajectory/SPEC.md §2b) — on conflict, the amended spec wins.

As a tanuki operator whose charters exist to probe a specific stage,
I want a charter to optionally declare a `probe` block — one required evidence predicate plus an unordered named set of checkpoint predicates — reviewed at the same plan gate as the charter itself,
So that "did this run exercise what the charter exists to test" has a machine-checkable, human-authorized definition instead of being inferred from driver exit status.

## Context / decision

Decomposed from #163 (spec lane, 2026-07-19; findings F178/F179). The operator
selected the evidence-predicate alternative over a target-declared stage/probe
graph — full record in
`specs/spec-tanuki-trajectory/DECISION-2026-07-19-probe-coverage.md`.
Checkpoint names are opaque strings; Tanuki stays target-agnostic. Probes are
charter material: they enter and change only through the generation pass's
existing plan gate, and no tool mutates the matrix.

## Acceptance Criteria

- Given a charter with a well-formed `probe` block (`required: {on: events|raw, type?, match}` + zero or more named `checkpoints` of the same shape), when the scenarios config loads, then the block parses and is available to the drive layer unchanged.
- Given a malformed probe block (unknown `on` value, missing `match`, invalid regex), when the config is loaded or validated, then the defect is named and the load fails closed — never silently ignored or partially applied.
- Given the generation pass proposes new or regenerated charters, when the plan gate renders, then any proposed probe blocks are shown alongside their charters for approve/edit/reject, and nothing is written without approval (a probe the human never reviewed is a coverage verdict the human never authorized).
- Given a charter without a probe block, when anything loads or runs, then behavior is unchanged from today (the block is opt-in; absence is legitimate).
- Given checkpoint names, when any surface consumes them, then they are treated as opaque strings — no ordering, ranking, or cross-checkpoint comparability is assumed or derivable from the schema.

## Out of scope

- Predicate evaluation and the manifest `probe` axis (story 1.25, #165).
- Rendering (story 1.26, #166).
- Any scheduler consumption of probe data (excluded by the ratified contract, v1).
