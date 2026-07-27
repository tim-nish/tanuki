---
story: 5.11
epic: 5
title: "Story 5.11: contribute_back enters the config catalog as dotted-path scalars"
status: published
issue: 306
umbrella: 306
---

# Story 5.11: contribute_back enters the config catalog as dotted-path scalars

Canonical discussion record: [#306](https://github.com/tim-nish/tanuki/issues/306).
On conflict, the issue wins.

Ruling (2026-07-27 triage, alternative A): no nested-object catalog type for
a two-leaf block — `contribute_back.path` and `contribute_back.schema` become
two typed scalar catalog entries; the nested JSON stays the storage shape.
Spec text: `specs/spec-den-distill/SPEC.md` §2 ("the block enters the catalog
as dotted-path scalars"). The doctor semantics are the ones §2 already
defines: path exists and is a directory, schema template resolves, block
validated before any drive consumes it (`specs/spec-den-distill/SPEC.md` §2).

## Story

As an operator configuring contribute-back on a fresh target, I want
`tanuki-config set`/`check` to declare and validate the block field by field,
so that I never hand-author raw JSON into a scenarios file that may not exist
yet.

## Acceptance criteria

**AC1 — two catalog entries.**
Given the declared-input catalog,
When it is listed,
Then `contribute_back.path` (existing directory) and
`contribute_back.schema` (resolvable intake template) appear as typed scalar
entries, each with its own doctor.

**AC2 — set assembles the nested shape.**
Given `tanuki-config set contribute_back.path <dir>` (and likewise
`.schema`),
When the write lands,
Then the target's configuration holds the nested
`{"path": …, "schema": …}` block — storage shape unchanged for every
existing consumer — and `--dry-run` renders the change before it lands.

**AC3 — both-or-neither guard.**
Given exactly one of the two leaves is set,
When `tanuki-config check` (or doctor) runs,
Then the half-configured block is flagged, per §2's block-doctor rule.

**AC4 — per-target scope enforced unchanged.**
Given a `contribute_back` value at machine-wide scope,
When resolution runs,
Then it is ignored and doctor flags it
(`specs/spec-den-distill/SPEC.md` §2, "Scope is per-target, never
machine-wide", issue #241 / F198).

## Story questions

- Is the catalog (story 1.37) genuinely typed-scalar-only today, and does its
  entry key syntax admit dots? Unverified — the catalog implementation in
  `tools/tanuki-config` was not read this sitting; #306's account is the
  issue's, not a code citation.
