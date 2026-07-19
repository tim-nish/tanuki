---
story: 1.39
epic: 1
title: "Story 1.39: [tool] finding-level, per-site behavior re-evaluation (fixes F77 half-fix; F79 regression)"
status: published
issue: 212
umbrella: 210
---

# Story 1.39: Finding-level, per-site behavior re-evaluation

Issue: https://github.com/tim-nish/tanuki/issues/212 — the canonical
discussion record; on conflict, the issue wins. Umbrella spec decision:
#210 (terminal reconcile contract, ratified 2026-07-19 —
`specs/spec-tanuki-loop/SPEC.md`: "a finding is resolved or superseded only
when closed at **every** known site").

As a tanuki-loop operator running reconcile,
I want each cited finding re-evaluated against current base behavior,
tracked per manifestation site,
So that a finding is never marked resolved or superseded while its behavior
is still live at any known site (the F77 half-fix: "superseded" shipped
while `cmd_journal` remained unfixed, needing a separate manual repair
turn).

## Context / decision

Decomposed from #210 (spec lane, ratified 2026-07-19). Hunk-level
comparison against *presence of a fix* is not the same as *behavior closed
at every site*. Consistent with the fail-closed rule: a site whose behavior
cannot be evidenced escalates as `needs-owner`, never assumed closed.

## Acceptance criteria

- **Given** a finding with multiple manifestation sites (F77:
  `cmd_review_consulted` fixed in base, `cmd_journal` not), **when**
  re-evaluated, **then** it is `still-applicable-at-remaining-sites` —
  never fully superseded while any site is live.
- **Given** a finding resolved by an independent landing (F79, resolved by
  the F5 working-note framework), **when** re-evaluated, **then** it is
  auto-classified resolved with its stored reproducing evidence.
- **Given** any resolved/superseded verdict, **then** closure evidence is
  persisted per site (the probe/behavior establishing closure), auditable
  and re-derivable, and surfaced in the reconcile report.
- **Given** a site whose behavior cannot be evidenced, **when**
  re-evaluated, **then** it escalates as `needs-owner`, never assumed
  closed.
