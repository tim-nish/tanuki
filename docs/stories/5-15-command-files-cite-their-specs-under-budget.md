---
story: 5.15
epic: 5
title: "Story 5.15: command files cite their specs and come in under budget"
status: published
issue: 343
umbrella: 343
depends_on: [5.14]
---

# Story 5.15: command files cite their specs and come in under budget

Canonical discussion record: [#343](https://github.com/tim-nish/tanuki/issues/343).
On conflict, the issue wins.

Ruling (2026-07-27 triage, G1 alternative A): restated contract passages in
command files carry a cite to their owning spec and the spec wins on
conflict (`docs/README.md`, "Derived-surface rules"). This story applies the
rule to the existing restatements and brings both files under the declared
ceiling.

## Story

As a maintainer amending a spec, I want the command files' overlapping
passages to cite that spec (or defer to it), so that a spec change cannot
leave a byte-identical-but-now-wrong copy behind in a command surface.

## Acceptance criteria

**AC1 — the known restatements cite or defer.**
Given the overlaps measured 2026-07-27:
`commands/tanuki-loop.md:269-487` (morning gate) vs
`specs/spec-tanuki-loop/SPEC.md:752-824`;
`commands/tanuki-loop.md:488-538` (PR delivery) vs `SPEC.md:825-943`;
`commands/tanuki-loop.md:300-327` (queue discharge, byte-identical opening)
vs `SPEC.md:760-771`;
`commands/tanuki.md:205-303` (view catalog) vs
`specs/spec-tanuki-view/SPEC.md:95-215`;
`commands/tanuki.md:167-195` (init steps, step 3 byte-identical) vs
`specs/spec-tanuki-scenario-lifecycle/SPEC.md:30-41`,
When this story lands,
Then each passage either carries an explicit cite to its owning spec section
or is replaced by a pointer, and no byte-identical spec paragraph survives
in a command file without one.

**AC2 — the ceilings hold.**
Given the declared ceilings (`docs/README.md` "Derived-surface rules":
~13k tokens per command file, measured 12,612 / 12,405 on 2026-07-27),
Then both files measure under their ceiling after the edit, and the trim's
outcome is reported so the ceiling can be reviewed downward.

**AC3 — behavior text is not weakened.**
Given `docs/README.md:16` (command files remain "the source of truth for
each command's behavior" — procedure text),
Then no operational instruction is deleted merely for size: what the
operator must *do* stays in the command file; what the system *promises* is
cited to its spec.

**AC4 — the loop's sanctioned fork is untouched.**
Given `commands/tanuki-loop.md:8-10` (attended-command rules "copied here,
allowed to diverge" — owner decision 2026-07-13),
Then that copied material is exempt from AC1 and unchanged by this story.

## Story questions

- The four-part/five-part invariant fix and the retired-steps deletion are
  #341 and #339 (direct lane) — if they land first, AC1's line numbers
  shift; re-measure at implementation rather than trusting the ranges above.
