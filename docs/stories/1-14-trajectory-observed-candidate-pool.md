---
story: 1.14
epic: 1
title: "Story 1.14: trajectory-observed unexplored branches seed the regeneration candidate pool"
status: ready
umbrella: 113
---

# Story 1.14: trajectory-observed unexplored branches seed the regeneration candidate pool

Umbrella issue: https://github.com/tim-nish/tanuki/issues/113

As a Tanuki operator generating new charters,
I want unexplored decision-point branches from recorded trajectories offered as candidates,
So that regeneration probes real paths the docs alone would never surface.

## Context / decision

Decomposed from #113 (spec lane, 2026-07-18). `specs/spec-tanuki-scenario-lifecycle/SPEC.md`
§3 now lists trajectory-observed unexplored branches among the generation
candidate sources. This story wires that pool: from `tanuki-scheduler history
--scenario <id> --trajectory`, surface decision-point alternatives that
recorded runs never took (e.g. `recover-mode: --adopt` never explored) as
generation candidates at the plan gate. Read-only; the operator still approves.

## Acceptance Criteria

- **Given** recorded trajectories with decision points whose alternatives were
  never taken, **when** `generate` runs, **then** those unexplored branches
  appear as charter candidates at the plan gate, each naming the decision point
  and the untaken alternative.
- **Given** a decision point all of whose alternatives have been explored,
  **when** `generate` runs, **then** it contributes no candidate from that point.
- The pool is read from persisted trajectory/scheduler artifacts only, never
  from model memory, and never mutates the matrix outside the plan gate.
- A test asserts an untaken alternative becomes a candidate and a fully-explored
  point does not.

## Depends on

- 1.13 (`generate` entry — where the candidate pool is presented)
