---
story: 1.29
epic: 1
title: "Story 1.29: One-shot per-run override \u2014 single-drive expiry, manifest record, canonical-state exclusion, loud render"
status: published
depends_on: ["1.27"]
issue: 182
umbrella: 172
---

# Story 1.29: One-shot per-run override — single-drive expiry, manifest record, canonical-state exclusion, loud render

Umbrella: https://github.com/tim-nish/tanuki/issues/172 (spec:
specs/spec-tanuki-scenario-lifecycle/SPEC.md "Host bindings & declared-input
configuration"; ruling:
specs/spec-tanuki-scenario-lifecycle/DECISION-2026-07-19-host-identity.md) —
on conflict, the decision record wins.

As a tanuki operator running one experiment against a different host,
I want a `per-run`-scoped value to persist as an override consumed by
exactly one drive — recorded in that run's manifest, rendered loudly while
pending, and excluded from all canonical scheduler and ledger arithmetic,
So that an experiment can never silently rewrite the canonical default,
pollute streak/demotion state, or falsely verify a canonical finding by
absence.

## Context / decision

Decomposed from #172 (spec lane, ratified 2026-07-19). The exclusion rule is
the permanent contract, not an interim: a non-canonical override run
contributes nothing to `record-run` (no yield, no streaks, no
demotion/regression) and accrues no driven-absence; its findings enter the
ledger host-tagged (`host_binding_id`), and canonical recurrence arithmetic
ignores non-canonical evidence. Full per-host state accrual belongs to
declared bindings only (1.30), never to one-shot overrides.

## Acceptance Criteria

- Given a persisted per-run override, when one drive consumes it, then the
  override is gone afterward — a loop iteration counts as one drive and the
  override never persists across iterations.
- Given an overridden run, when its manifest is written, then it records the
  effective resolved inputs, so history can distinguish the run without
  model memory.
- Given a pending (unconsumed) override, when `status` or the dashboard
  renders, then the override is named loudly — a forgotten override must be
  impossible to miss.
- Given `record-run` on an overridden run, when it completes, then no
  scenario's `low_streak`/`runs_driven`/state changed and no yield snapshot
  was folded (test fixture proves a subsequent canonical run's
  driven-absence counting is unaffected).
- Given findings from an overridden run, when they enter the ledger, then
  their evidence carries the override's `host_binding_id`, and canonical
  recurrence/driven-absence arithmetic provably ignores it.

## Out of scope

- Declared bindings and per-host state accrual (1.30).
- The configuration surface itself (1.27/1.28).
