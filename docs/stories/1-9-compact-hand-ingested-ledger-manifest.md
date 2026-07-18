---
story: 1.9
epic: 1
title: "Story 1.9: compact fires on hand-ingested ledgers — ingest writes a run manifest so verify-by-absence can advance"
status: published
issue: 106
---

# Story 1.9: compact fires on hand-ingested ledgers — ingest writes a run manifest so verify-by-absence can advance

Canonical discussion record: https://github.com/tim-nish/tanuki/issues/106

As a tanuki operator building a ledger by hand (`ingest` / `ingest-note`) rather than by a driven `tanuki-drive` run,
I want `compact` to actually advance on my manual ledger,
So that accepted findings tombstone as the spec's own elapsed-run fallback already promises, instead of reporting `nothing to compact` forever.

## Context / decision

Triaged from #106 (spec lane, 2026-07-18). The governing invariant
(`specs/spec-tanuki-scenario-lifecycle/SPEC.md`, "Driven-absence compaction")
already promises that *"a finding with no recorded scenarios falls back to
elapsed-run counting (legacy compatibility)"*. The gap is purely mechanical:
`all_run_ids` / `driven_scenarios_by_run` enumerate `events/<run>/` run
directories with a manifest, and plain `ingest` never writes one — so the
elapsed-run fallback the contract guarantees can never advance for a manual
ledger. **The chosen resolution is contract-preserving:** make manual ingests
first-class runs by writing a minimal run manifest, so no spec text changes and
driven-run / scheduler / convergence semantics stay exactly as they are.

## Acceptance Criteria

**Given** a fresh target and a hand-written `events.jsonl`
**When** the operator runs `tanuki-ledger ingest <file>` (and `ingest-note`)
**Then** a minimal run manifest is written under `~/.tanuki/<target>/events/<run>/`
(a `manual-<YYYYMMDD>` run id, `driven_scenarios: []`, a `completed_at` stamp),
so the manual run is a first-class `all_run_ids` entry.

**Given** an accepted finding on a hand-ingested ledger whose only scenario is
the reserved `ingest` pseudo-scenario
**When** `compaction_unseen_runs` manual runs have elapsed since it was last seen
**Then** `compact` tombstones it via the existing elapsed-run fallback
(`verified_absent_runs` → `runs_since`), and `status`'s `runs: N` counter agrees
with what compact's gate counts.

**Given** a driven run (real scenarios in its manifest)
**When** `compact` runs
**Then** driven-absence semantics are unchanged — a driven finding still
compacts only after `N` driven-and-absent runs, never on elapsed time.

**Given** a `dismissed` finding
**Then** its compaction is unaffected (deadness is not scenario-conditional).

## Out of scope

- No change to the driven-absence invariant text (this story deliberately keeps
  the contract fixed; the alternative that amends the spec was considered and
  not chosen).
- No change to `--unseen-runs` semantics (tracked separately as #106-adjacent
  F73).

## Notes for the implementer

- The manifest shape should match what `tanuki-drive` writes closely enough that
  `all_run_ids` / `driven_scenarios_by_run` read it without special-casing;
  `driven_scenarios` is empty for a manual run, which routes the finding through
  the already-present `scen = set(scenarios) - {"ingest"}` → `runs_since`
  fallback in `verified_absent_runs`.
- Verify against the `ledger-lifecycle` dogfood scenario, which is where #106
  (finding F65, recurrence 8) was repeatedly observed.
