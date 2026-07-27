---
story: 5.6
epic: 5
title: "Story 5.6: status names the hand-ingest dead end, and set-status stays silent"
status: published
issue: 307
umbrella: 307
---

# Story 5.6: status names the hand-ingest dead end

Canonical discussion record: [#307](https://github.com/tim-nish/tanuki/issues/307).
On conflict, the issue wins.

> **Scope note.** #307 asked for a warning that did not exist when it was filed.
> The **behaviour** has since landed on `main` in `96f3a63` (PR #327): an
> ACCEPTED row whose verdict is `cannot-determine` is marked on the row and
> carries `-> drive <scenarios> …` beneath it
> (`tools/tanuki-ledger:1686-1691`). The 2026-07-27 owner ruling (commit
> `b064b16`) then fixed `status` as the earliest *required* surface and
> **declined** to relax the canonical-transition silence rule. This story
> covers only what is left: a regression test pinning both halves, so neither
> can be undone by a later edit that looks locally reasonable.

## Story

As **an operator who hand-ingested a finding scoped to a real scenario**,
I want **the dead end named the moment I ask where the finding stands**,
so that **I do not walk the documented lifecycle to its end before learning
that hand-ingest could never have finished it.**

## Context

Verification by absence requires a `tanuki-drive` re-verification; hand-ingest
never advances it. Before #327, the only surface that said so was `compact`
(`tools/tanuki-ledger:1351`) — the last step of the walk, which is exactly where
learning it is useless.

The natural fix — warn at `set-status --status accepted` — was implemented
during loop run `loop-20260726-204018` iter 2, tripped
`tools/tests/test-ledger-lifecycle-notice`, and was reverted. That test pins "a
canonical transition is silent" (story 1.5 / issue #67 / F63, spec decision
2026-07-17), and `open → accepted` is canonical. The owner ruling resolved the
collision **in the test's favour**: the warning belongs to `status`, and
`set-status` stays silent.

Both halves are currently unpinned. The `status` behaviour has no test at all,
and nothing records *why* `set-status` must stay quiet about this particular
fact — so a future contributor re-deriving the "obvious" fix would repeat the
reverted attempt.

Governing text: `specs/spec-tanuki-scenario-lifecycle/SPEC.md` §"Absence
verification is THREE-valued", the 2026-07-27 addition.

## Acceptance criteria

**AC1 — the row carries the verdict.**
Given an accepted finding scoped to a real scenario that no recorded run ever
drove,
When `status` renders the ACCEPTED list,
Then that finding's row is marked `cannot-determine` on the row itself, not only
in trailing prose.

**AC2 — the row names one runnable resolving command.**
Given the same finding,
When `status` renders its row,
Then a line beneath it names the scenario(s) to drive and gives a runnable
`tanuki-drive` command, and the reserved `ingest` pseudo-scenario never appears
among them.

**AC3 — absent-with-evidence is not warned about.**
Given an accepted finding that HAS been driven and sits below the compaction
threshold,
When `status` renders the ACCEPTED list,
Then its row carries no cannot-determine marking and no resolving-command line —
the requirement binds `cannot-determine` only.

**AC4 — set-status stays silent, deliberately.**
Given a finding transitioning `open → proposed → accepted`,
When each `set-status` call runs,
Then stderr is empty for every canonical step, and the test states in its own
comments that the hand-ingest warning is `status`'s job and must not be added
here — citing the 2026-07-27 ruling — so the reverted attempt is not repeated.

## Tasks

1. Extend `tools/tests/` with a fixture covering AC1–AC3 against a
   `TANUKI_ROOT` temp ledger: one never-driven accepted finding, one driven
   below-threshold accepted finding, assert the two rows differ in the ruled way.
2. Add the AC4 comment + assertion to the existing
   `tools/tests/test-ledger-lifecycle-notice`, or to the new fixture if that
   file's scope should stay narrow — either placement satisfies the AC, so long
   as one test fails if `set-status` starts printing on `open → accepted`.
3. Run the full suite (`for t in tools/tests/test-*; do "$t" || exit 1; done`).

## Out of scope

- Adding the warning to `upsert-finding` or `promote`. The ruling permits both
  and requires neither; adding one now would pin an unrequired surface.
- Relaxing the canonical-transition silence rule. Explicitly declined.
- `tools/tanuki-ledger:1540`, the reach-era summary line that still names no
  scenario. Changing it would alter output stories 5.3/5.4 ratified; it is
  tracked separately, not here.
