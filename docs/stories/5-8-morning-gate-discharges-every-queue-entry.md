---
story: 5.8
epic: 5
title: "Story 5.8: the morning gate discharges every queue entry before the run closes"
status: published
issue: 309
umbrella: 309
---

# Story 5.8: the morning gate discharges every queue entry

Canonical discussion record: [#309](https://github.com/tim-nish/tanuki/issues/309).
On conflict, the issue wins. Coupled decision with #310 (story 5.9): one
2026-07-27 ruling (alternative A) covers both.

Ruling: deferrals bind per emitted item — served policy, consulted at
`product-lab@d261caee84c9e617818b5f92c8757fbb2decf1fa`: "A deferral is
legitimate only if the deferred work receives its tracking artifact
(story/issue …) in the same sitting … a machine-generated gap list is a
deferral generator that needs the rule to bind per emitted item"
(LESSONS.md:74). The attended morning gate is that same-sitting moment;
unattended iterations still write only `queue.md` (the no-outward-artifacts
rule, `specs/spec-tanuki-loop/SPEC.md` §"Defer / freeze, do not stop").
Spec text: `specs/spec-tanuki-loop/SPEC.md` §"Morning gate" ("Every queue
entry is discharged before the run closes").

## Story

As the operator at the morning gate, I want the gate to refuse to close while
any queue entry is undischarged, so that an unattended run's deferrals can
never silently delete scope.

## Acceptance criteria

**AC1 — the gate walks every entry.**
Given a run whose `queue.md` holds N deferral entries,
When the morning gate runs,
Then each entry is presented for a disposition and the gate does not complete
while any entry lacks one.

**AC2 — three dispositions, each recorded.**
Given a walked entry,
When the operator disposes it,
Then the disposition is one of: filed (issue/story URL recorded beside the
entry), executed at the gate, or no-artifact-needed with a reason — and the
record lands beside the entry in `queue.md`, not only in session output.

**AC3 — emission-time filing stays forbidden.**
Given an unattended iteration defers a finding,
When `tanuki-loop defer` records it,
Then no issue, story, or other outward artifact is created at emission time —
the queue line remains the only write, exactly as before this story.

## Story questions

- Where does the gate's queue read live in `tools/tanuki-loop`, and does a
  gate abort (failed final tests) preserve partial dispositions? Unverified —
  the gate implementation sites were not read this sitting.
