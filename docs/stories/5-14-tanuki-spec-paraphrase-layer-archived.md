---
story: 5.14
epic: 5
title: "Story 5.14: tanuki-spec's paraphrase layer is archived behind pointers"
status: published
issue: 345
umbrella: 345
---

# Story 5.14: tanuki-spec's paraphrase layer is archived behind pointers

Canonical discussion record: [#345](https://github.com/tim-nish/tanuki/issues/345).
On conflict, the issue wins.

Ruling (2026-07-27 triage, G1 alternative A): `docs/tanuki-spec.md` is
canonical for exactly standing constraints, the three-role model, the public
vocabulary, and the config-key table (`docs/README.md`, "Derived-surface
rules" — amended this sitting). Its command-grammar / decision-pass /
file-layout prose is a paraphrase of the per-topic specs and is archived,
not maintained.

## Story

As a reader of `docs/tanuki-spec.md`, I want its non-canonical sections
replaced by pointers to the specs that own them, so that the whole-system
contract I read cannot drift from the per-topic contracts.

## Acceptance criteria

**AC1 — the paraphrase layer is gone from the live file.**
Given the command-grammar / decision-pass / file-layout sections
(`docs/tanuki-spec.md:398-500` at the 2026-07-27 measurement),
When this story lands,
Then each is reduced to a pointer naming its owner
(`specs/spec-short-command-surface/SPEC.md`, `specs/spec-tanuki-solve/SPEC.md`,
`specs/spec-tanuki-view/SPEC.md`, `commands/tanuki.md` for the argument
grammar), with any removed prose preserved per the repo's archive convention
(`docs/README.md` §4: superseded material struck through with a pointer, or
moved to a non-normative location).

**AC2 — the canonical remainder is untouched.**
Given the canonical scope (constraints, roles, vocabulary, config table),
Then those sections are byte-identical apart from the pointer edits, and
`specs/spec-tanuki-scenario-lifecycle/SPEC.md:241`'s inbound reference
("Config keys (added to the table in docs/tanuki-spec.md)") still resolves.

**AC3 — the ceiling holds.**
Given the declared ceiling (`docs/README.md` "Derived-surface rules":
docs/tanuki-spec.md ≤ ~8.5k tokens),
Then the file measures at or under it after the archive (measured 8,251
tokens before; the archive must not push it over via added pointer prose).

**AC4 — the README file-layout duplicate cross-references.**
Given the second copy of the file-layout inventory at `README.md:401-436`,
Then one side points at the other rather than both standing free.
