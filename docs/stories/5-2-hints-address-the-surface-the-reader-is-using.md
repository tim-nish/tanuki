---
story: 5.2
epic: 5
title: "Story 5.2: Operator-facing hints instruct the user to run internal tools directly, with no rule fixing which surface a hint may address"
status: published
issue: 311
umbrella: 311
---

# Story 5.2: Hints address the surface the reader is using

Canonical discussion record: [#311](https://github.com/tim-nish/tanuki/issues/311).
On conflict, the issue wins.

> **Revised 2026-07-26** after #311 was re-triaged through the spec lane. The
> first draft assumed D2 already settled this and read the current code as a
> plain violation. It did not: D2 fixed the *form* and left the *addressee*
> open, and the code's ordering was F84's deliberate fix. D2 has since been
> amended with the addressee rule, and the criteria below implement that
> amendment — the direction of AC1 is inverted from the first draft.

## Story

As **an operator following Tanuki's own guidance**,
I want **every hint to name one command that runs where I am standing**,
so that **following the tool's advice never dead-ends on a command I cannot
find, cannot run, or must fill in by hand.**

## Context

Six recorded findings oscillated over this: F24 (rec 8) made the hint adaptive;
F84 found it naming a `/tanuki` command the CLI does not implement — a dead end
from a shell; F90 found its fix naming a sibling tool `tanuki-ledger --help`
never mentions — a different dead end; F130/F120 hit it from the other side.
Each fix traded one unreachable reference for another because nothing said who
the hint was addressed to.

`specs/spec-short-command-surface/SPEC.md` D2 now says
(AMENDED 2026-07-26 — issue #311): exactly one command per hint; each layer
names the command runnable on its own surface; the command layer translates
when it relays; nothing presented as runnable carries an unresolved
placeholder; and a lint enforces it.

## Acceptance criteria

**AC1 — a tool's hint names exactly one command, in CLI form.**
Given any enumerated ledger state,
When `next` or `status` renders its hint,
Then it names exactly one command, in the form runnable in the shell the output
is read in — no ranked pair, no parenthetical alternative. Two states name a
`/tanuki` form alongside the CLI form today (`tools/tanuki-ledger:1519` and
`:1542`); after this story each names one.

**AC2 — the command layer translates rather than quoting.**
Given `/tanuki` relays a hint to a Claude Code user,
When the hint names a CLI command,
Then the command layer renders the equivalent `/tanuki` form instead of quoting
the tool's string. `commands/tanuki.md` carries this relay rule where the
command layer reads it.

**AC3 — nothing presented as runnable carries an unresolved placeholder.**
Given a hint renders a command line,
When the operator copies it,
Then it runs as written: every argument derivable from state is derived, and
where a value cannot be derived the hint describes the act instead of emitting
a command that only looks pasteable.
*Known instance, not on `main`:* commit `1d0b680` on branch
`tanuki-loop/tanuki/20260726-204018` emits
`tanuki-drive --target <t> --plugin <repo> --scenarios <file> --run <run> --only …`.
That branch is undelivered, so this is satisfied by amending it before delivery
or correcting it after.

**AC4 — a scope outside this target's matrix is never emitted as a drive target.**
Given an accepted finding's evidence names a scenario absent from the target's
matrix,
When a hint states how to resolve it,
Then it says the scenario is not in this target's matrix rather than emitting a
`--only` command that cannot succeed. Observed with F178, whose scenarios
(`draft-postmortem-f2`, `draft-survey`) belong to the `writing-assistant`
matrix.

**AC5 — `--help` names the wrapper it assumes.**
Given D3 requires help text to carry the contract,
When the operator reads `tanuki-ledger --help`,
Then the `/tanuki` command layer is named there — so a CLI reader who wants the
guided path can find it, closing F84's dead end from the other direction.

**AC6 — a lint enforces AC1, AC3 and AC4.**
Given the tracked tree,
When the lint runs in CI,
Then a hint string naming a bare sibling tool where the command layer owns the
surface, or containing an unresolved `<placeholder>` in a presented command,
fails the check. `tools/check-publication-boundary` is the shipped precedent for
a mechanical rule over the tracked tree.

**AC7 — the per-state fixtures are updated, not worked around.**
Given `tools/tests/test-ledger-next` currently pins the two-command prose,
When AC1 lands,
Then those fixtures pin the conforming single-command text.

## Story questions (unverified — not acceptance criteria)

- How many enumerated states name more than one command? Two were located by
  inspection (`:1519`, `:1542`); the full enumeration was **not** walked
  (cannot-determine — not read). Further tool-form mentions at
  `tools/tanuki-ledger:1348`, `:1356` and `:2330` sit in absence/compaction
  prose and need judging individually.
- Can a lint catch a command-layer relay that pastes raw tool output? AC6
  covers the tool's strings; the relay side is prose in `commands/tanuki.md`
  and may not be mechanically checkable. This is the counter-argument recorded
  at the decision gate and is **not** resolved here.
- Should the lint cover `tanuki-loop` and `tanuki-scheduler` output too? F174
  (dashboard naming a nonexistent `reconcile` subcommand) suggests yes; scope
  undecided.

## Out of scope

- Whether the CLI form or the `/tanuki` form is "primary". D2's amendment
  settles it per layer: each names what runs on its own surface, and the
  translation happens once at the boundary.
