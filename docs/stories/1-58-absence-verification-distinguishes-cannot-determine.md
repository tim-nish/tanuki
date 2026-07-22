---
story: 1.58
epic: 1
title: "Story 1.58: Absence verification distinguishes cannot-determine from absent-with-evidence"
status: published
issue: 290
umbrella: 290
---

Canonical discussion record: https://github.com/tim-nish/tanuki/issues/290
On any conflict between this story and the issue, the issue wins.

Governing invariant (added by the 2026-07-22 owner ruling, triage of #290):
`specs/spec-tanuki-scenario-lifecycle/SPEC.md`, "Verify / cap interaction" →
"Absence verification is THREE-valued" — read the substrate's population for
the scope being verified before reading your own count; zero entries for that
scope means `cannot-determine` regardless of the counter.

**Scope note (read before implementing).** The issue's *stated mechanism* is
stale and this story deliberately does not re-fix it. `cmd_ingest` DOES now
write run directories: `write_manual_manifest`
(`tools/tanuki-ledger:901-923`) is wired into both ingest paths
(`tools/tanuki-ledger:301`, `tools/tanuki-ledger:386`) by story 1.9 / #106,
and ingest/no-scenario findings advance via the elapsed-run fallback
(`tools/tanuki-ledger:1096-1097`). Likewise the "indistinguishable from a
genuine all-clear" symptom is already largely addressed by the per-finding
cause lines (`tools/tanuki-ledger:1165-1178`, F135) and the structural cause
line (`tools/tanuki-ledger:1196-1199`, F141). What remains unimplemented is
the ruling's mechanical criterion and a consumable verdict.

**As** a caller or operator reading a compaction report,
**I want** "this finding's scenario was never driven in any run" reported
differently from "it was replayed and stayed absent, just not enough times",
**so that** the system's least-informed state stops being reported with the
same confidence as its best-informed one.

## Acceptance criteria

- **Given** an accepted finding whose scenarios appear in **no** run manifest
  at all, **when** compaction is evaluated, **then** it is reported as
  `cannot-determine`, naming why (no runs recorded that drove this scope) —
  never with the same shape as a below-threshold `absent-with-evidence`.
  Today `verified_absent_runs` (`tools/tanuki-ledger:1078-1103`) returns
  `k=0` identically for both cases; the population read is the change.
- **Given** an accepted finding whose scenarios **were** driven in at least
  one recorded run without it recurring, **when** the count is below the
  threshold, **then** behavior is unchanged from today — it reports
  `absent-with-evidence` progress (`k/n`) and compacts on reaching `n`. No
  behavior change on the path that works.
- **Given** the report, **then** the three-valued verdict is consumable by a
  caller, not prose alone. `compact` currently registers no machine-readable
  output option (`tools/tanuki-ledger:2143`); the verdict must be reachable
  without parsing sentences.
- **Given** the status surface, **then** it carries the same distinction and
  the same vocabulary. `compaction_hint` (`tools/tanuki-ledger:1294-1310`)
  already re-derives the real-scenario split independently of `cmd_compact`;
  the two must stay in lockstep rather than growing a second vocabulary.
- **Given** the change, **then** a regression fixture covers both paths —
  a finding whose scope was never driven, and a driven finding below
  threshold. `tools/tests/test-ledger-compact-hint` is the existing fixture
  home for this surface.

## Story question (unverified — decision not located)

Whether `cannot-determine` should also gate `dismissed` findings is not
settled by the issue or the spec. `dismissed` compaction counts elapsed runs
by design ("deadness is not scenario-conditional",
`specs/spec-tanuki-scenario-lifecycle/SPEC.md` "Driven-absence compaction"),
so the population criterion may simply not apply there. Confirm before
implementing rather than inferring — this story's criteria above deliberately
speak only to `accepted`.
