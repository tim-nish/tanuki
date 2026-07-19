---
story: 1.40
epic: 1
title: "Story 1.40: [tool] cmd_reconcile: end-to-end orchestration — fixpoint, one gate, persisted resumable transaction, terminal merge/push/cleanup"
status: published
issue: 213
umbrella: 210
depends_on: [1.38, 1.39]
---

# Story 1.40: cmd_reconcile end-to-end orchestration

Issue: https://github.com/tim-nish/tanuki/issues/213 — the canonical
discussion record; on conflict, the issue wins. Umbrella spec decision:
#210 (terminal reconcile contract, ratified 2026-07-19 —
`specs/spec-tanuki-loop/SPEC.md`, seven-clause terminal invocation).

As a tanuki-loop operator,
I want one reconcile invocation that drives the whole terminal flow —
classify → apply → re-evaluate to a fixpoint, verify, one gate, then
merge/push/state-update/cleanup,
So that a session like writing-assistant `loop-20260719-173009` completes
in one invocation with one gate instead of ~5 human turns.

## Context / decision

Decomposed from #210 (spec lane, ratified 2026-07-19). Integration only:
composes the disposition engine (story 1.38 / #211) and per-site finding
re-evaluation (story 1.39 / #212) — it must not duplicate their scope. On
a `gate: "pr"` target the terminal merge is expressed as PR mechanics, the
gate approval being the owner's explicit decision (2026-07-17 ruling
preserved). Branch/worktree cleanup formerly owned by `/repo-cleanup`
becomes the terminal step here, per #210's ownership move.

## Acceptance criteria

- **Given** unresolved runs exist, **when** reconcile is invoked with no
  identifiers, **then** it enumerates all of them (branch filters optional,
  advanced-only; hashes/branch names appear only as evidence) and loops
  classify → apply → re-evaluate to a fixpoint before running the
  configured verification suite as a gate condition.
- **Given** the fixpoint is reached, **then** exactly one gate is
  presented, containing only genuine owner decisions (conflicts, ambiguous
  supersession, spec-lane routing, `needs-owner` items).
- **Given** gate approval, **then** merge → push → state update → cleanup
  complete with no further commands.
- **Given** the process is killed mid-flow, **when** re-run, **then** it
  resumes from the last persisted phase checkpoint — never re-porting,
  re-merging, re-pushing, or re-asking an answered decision (the gate
  decision itself is persisted).
- Terminal state is one of `complete` | `needs-owner` | `blocked`, with
  evidence.
