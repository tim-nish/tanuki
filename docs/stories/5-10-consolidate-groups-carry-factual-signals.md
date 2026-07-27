---
story: 5.10
epic: 5
title: "Story 5.10: consolidate candidate groups carry typed factual signals"
status: published
issue: 304
umbrella: 304
---

# Story 5.10: consolidate candidate groups carry typed factual signals

Canonical discussion record: [#304](https://github.com/tim-nish/tanuki/issues/304).
On conflict, the issue wins.

Ruling (2026-07-27 triage, alternative A): the tool emits richer **facts**,
never a class verdict — no `kind`/`hint` field. The merge/conflict/dependency
reading stays the decision pass's judgment, exactly as D1/D2 reserve it
("It proposes, never decides" — `specs/spec-tanuki-solve/SPEC.md` §D2).
Spec text: the amended §D2 sentence enumerating the signal vocabulary.

## Story

As a first-time operator reading `tanuki-ledger consolidate` output, I want
each candidate group to carry signals from which its class is inferable, so
that I can tell a merge from a conflict from a dependency without the tool
asserting the verdict for me.

## Acceptance criteria

**AC1 — typed signals per group.**
Given `consolidate` emits candidate groups as JSON,
When a group is emitted,
Then it carries typed factual signals alongside the existing
`ids`/`members`/`pairs`/`signals` shape — at minimum: `evidence_overlap`
(shared evidence pointers), `proposals_conflict` (mechanical contradiction
markers between proposal texts), `reframes` (a one-way reframing edge
between members) — each a fact the clustering computed.

**AC2 — no verdict field.**
Given any emitted group,
When its JSON is inspected,
Then no field names or implies the merge/conflict/dependency class; the
signal names describe evidence, not classification.

**AC3 — read-only contract preserved.**
Given the amended emitter,
When `consolidate` runs,
Then it still reads bounded views only and writes nothing
(`specs/spec-tanuki-solve/SPEC.md` §D2).

## Story questions

- What does the current `signals` array actually contain, and does
  `tools/tests/test-ledger-consolidate` (named at
  `specs/spec-tanuki-solve/SPEC.md:10`) pin the group schema? Unverified —
  neither the emitter nor the test was read this sitting; the new fields must
  extend, not break, the pinned shape.
