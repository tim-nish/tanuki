---
story: 1.36
epic: 1
title: "Story 1.36: Distill-den picker action over the existing ledger, with the unconfigured one-line notice"
status: published
depends_on: ["1.34", "1.35"]
issue: 189
umbrella: 179
---

# Story 1.36: Distill-den picker action over the existing ledger, with the unconfigured one-line notice

Umbrella: https://github.com/tim-nish/tanuki/issues/179 (spec:
specs/spec-den-distill/SPEC.md §4) — on conflict, the spec wins.

As a tanuki operator with chronic lesson candidates already in the ledger
from past runs,
I want a "distill den" action in the existing `/tanuki <target>` picker that
runs the same candidate walk over the existing ledger without driving
anything, and a one-line notice wherever candidates render while
`contribute_back` is unset,
So that contributing past history never requires a new run, and an
unconfigured target tells me what I'm missing instead of silently rendering
prose.

## Context / decision

Decomposed from #179 (spec lane, ratified 2026-07-19). No new top-level
command — command proliferation is itself a UX failure (the #172 rule);
`/tanuki <target> distill` is acceptable as direct syntax only if the
grammar requires it. The walk and writes must behave identically to the
in-run pass (1.35) — one code path, two entries.

## Acceptance Criteria

- Given a pre-existing ledger with undecided candidates, when "distill den"
  is chosen from the picker, then the same walk as 1.35 runs (accept writes
  + receipt, dismiss/defer via `set-status`), with no drive.
- Given candidates present and `contribute_back` unset, when the brief or
  the decision pass renders, then the one-line notice appears ("N lesson
  candidates; contribute-back not configured — …") and nothing is written.
- Given no undecided candidates, when the action is chosen, then it says so
  in one line — never a crash, never an empty walk.
- Given the picker, when it renders, then "distill den" is a named action
  with a state-derived hint (candidate count), consistent with the existing
  picker conventions.

## Out of scope

- Emit mechanics (1.34) and the in-run pass (1.35).
- Configuring `contribute_back` through the declared-input surface — that
  routes through #172's stories (1.27/1.28) once they land.
