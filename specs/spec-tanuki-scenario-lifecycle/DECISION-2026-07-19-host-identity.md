# Decision: execution identity is (scenario_id, host_binding_id) — per-host scheduler state

Status: RATIFIED 2026-07-19 (operator ruling, triage of issues #172/#173).
Binds the spec-lane proposals #172 (declared-input configuration surface) and
#173 (host portability); implementation stories are cut from those issues
against this contract.

## The ruling

1. **Per-host state, keyed `(scenario_id, host_binding_id)`.** Of the two
   options in #173's Execution identity section, the operator ratified
   option 1: scheduler state — yield, low-yield streaks, demotion/regression
   arithmetic, transitions — is kept **per host binding**, not host-tagged
   inside a single shared row. Ledger evidence carries the
   `host_binding_id`; recurrence and driven-absence arithmetic evaluate
   against one binding at a time. A quiet run on host B can never demote a
   scenario productive on host A, and can never accrue verify-by-absence for
   a finding observed on host A.

2. **`host_binding_id` is logically stable and separate from the commit
   pin.** The id names the *binding* — the declared host identity a target's
   scenarios run against (e.g. the named repo binding from #173's declared
   bindings) — not a snapshot of it. Commit pins (host SHAs, fixture
   `source_commit`) identify **states** of a host and continue to live where
   they live today (fixture provenance, manifests); re-pinning, pulling, or
   advancing the host repo NEVER changes its `host_binding_id`. Renaming a
   binding is an explicit operator act, not a side effect of host movement.

3. **The #172 one-shot override keeps the canonical-state exclusion rule.**
   A run driven under a per-run (non-canonical) host override is excluded
   from `record-run` and from driven-absence entirely, exactly as #172's
   "Execution identity and state protection" section documents — it does not
   create or mutate per-host state for the override binding. Exploratory
   posture: findings enter the ledger host-tagged, nothing else moves. Full
   per-host state accrual applies only to hosts declared as bindings (#173),
   never to one-shot overrides.

## Why per-host state (and not host-tagged single state)

- Streak/demotion semantics are naturally per-binding; filtering a shared
  row's arithmetic by tag reconstructs per-host state at every read anyway,
  with more ways to get it wrong.
- History and the dashboard render one binding's story without cross-host
  bleed; a cross-host rollup is a pure read-time aggregation.
- The interim exclusion rule already shipped host-tagged *evidence*; that is
  required by both options and carries forward unchanged.

## Consequences

- `scheduler.json` layout gains a binding dimension for targets with more
  than one declared binding; single-binding targets (all current ones) are
  unchanged until a second binding is declared — no migration.
- `record-run`, `plan`, `history`, and `scenario-yield` operate on the
  effective binding; the plan record names it.
- #171's `candidates` host pass-through renders the binding id.
- The compatibility check (#173) and `compat_skipped` accounting are
  per-binding by construction.
