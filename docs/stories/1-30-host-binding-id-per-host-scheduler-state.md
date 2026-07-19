---
story: 1.30
epic: 1
title: "Story 1.30: host_binding_id and per-host scheduler state keyed (scenario_id, host_binding_id)"
status: published
depends_on: ["1.27"]
issue: 183
umbrella: 173
---

# Story 1.30: host_binding_id and per-host scheduler state keyed (scenario_id, host_binding_id)

Umbrella: https://github.com/tim-nish/tanuki/issues/173 (ruling:
specs/spec-tanuki-scenario-lifecycle/DECISION-2026-07-19-host-identity.md;
spec: specs/spec-tanuki-scenario-lifecycle/SPEC.md "Host bindings &
declared-input configuration") — on conflict, the decision record wins.

As a tanuki operator running one target's scenarios against more than one
host repository,
I want scheduler state — yield, low-yield streaks, demotion/regression
arithmetic, transitions — kept per declared host binding, keyed
`(scenario_id, host_binding_id)`, with ledger evidence carrying the binding
id and recurrence/driven-absence evaluating one binding at a time,
So that a quiet run on host B can never demote a scenario productive on
host A, nor falsely verify-by-absence a finding observed on host A.

## Context / decision

Decomposed from #173 (spec lane, ratified 2026-07-19 — option 1, per-host
state, over host-tagged single state). `host_binding_id` is logically
stable and separate from the commit pin: it names the declared binding,
never a snapshot; re-pinning or advancing the host repo never changes
identity, and renaming a binding is an explicit operator act. Single-binding
targets (all current ones) must be byte-for-byte unaffected until a second
binding is declared — no migration.

## Acceptance Criteria

- Given a target with one declared binding (or none — today's shape), when
  any scheduler command runs, then behavior and state layout are unchanged
  from today (no migration, no new required flags).
- Given a second declared binding, when runs are recorded against each, then
  each binding has independent streaks/demotion/regression arithmetic, and a
  low-yield streak accumulated on one binding leaves the other's
  classification untouched (test fixture).
- Given ledger evidence written under a binding, when recurrence or
  driven-absence is computed, then only evidence from the binding being
  evaluated counts.
- Given a host repo that is re-pinned, pulled, or advanced, when identity is
  resolved, then its `host_binding_id` is unchanged (pins identify states,
  never identity).
- Given `history`/`status`, when a multi-binding target renders, then state
  is presented per binding, and the plan record names the effective binding.

## Out of scope

- Binding placeholders in prompts/host_setup (1.31).
- Coupling detection (1.32) and the compat check / per-host fixtures (1.33).
- One-shot overrides — they never accrue per-host state (1.29's contract).
