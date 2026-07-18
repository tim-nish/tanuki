---
story: 1.13
epic: 1
title: "Story 1.13: generate — first-class mode word for scenario regeneration"
status: ready
umbrella: 113
---

# Story 1.13: generate — first-class mode word for scenario regeneration

Umbrella issue: https://github.com/tim-nish/tanuki/issues/113

As a Tanuki operator,
I want a first-class `/tanuki <target> generate` entry to regeneration,
So that growing the matrix is a documented, invocable mode rather than one prose sentence.

## Context / decision

Decomposed from #113 (spec lane, 2026-07-18). `commands/tanuki.md` and
`specs/spec-short-command-surface/SPEC.md` now define `generate` as a bare mode
word that does not drive (spec-short-command-surface D6). This story implements
the entry: it runs the same frontier-judgment, human-gated generation the init
flow's step 4 runs, reachable on demand and on either the pool-empty or
feature-drift triggers.

## Acceptance Criteria

- **Given** `/tanuki <target> generate`, **when** invoked, **then** it runs the
  §3 generation pass (does not drive), proposes charters at the plan gate, and
  on approval writes them and runs `tanuki-scheduler sync` (new ids → unexplored).
- **Given** the disambiguation rules, **when** `generate` appears in mode
  position after a target, **then** it is read as the mode word (reserved
  closed set), and a quoted argument is never read as `generate`.
- **Given** the user rejects the proposed charters at the gate, **when** the
  pass ends, **then** nothing is written to the scenarios file.
- The pass never drives and never mutates the matrix except through the
  approved plan gate.
- A test covers invocation, the plan gate, and the approve/reject paths.
