---
story: 2.3
epic: 2
title: "Story 2.3: record an unreached scenario execution as unreached in the manifest"
status: ready
umbrella: 299
---

# Story 2.3: An execution that never reached the subject is recorded `unreached`

Umbrella: [#299](https://github.com/tim-nish/tanuki/issues/299). This story is
the first half of that issue's decomposition; the accounting change that
consumes this record is story 2.4.

## Story

As **the consumer of a run's manifest**,
I want **an execution that never entered the plugin flow to be marked as such**,
so that **downstream accounting can tell "ran and found nothing" apart from
"never reached the thing it was testing."**

## Context

Loop run `loop-20260726-201337` (target `writing-assistant`, iter1) had 4 of 5
verify-set scenarios short-circuit at 3–13 turns: the driver read the target
skill's invocation payload as documentation and never entered the pipeline flow.
All four reported `status: ok`, `probe: undeclared`.

The two-axis rule correctly kept `ok` from *reading* healthy —
`commands/tanuki.md:413`, "the two axes are never merged into one verdict" — but
no machine-readable field distinguishes an unreached execution, so every
consumer treats it as a valid one.

The governing invariant was amended in the same sitting as this story
(`specs/spec-tanuki-scenario-lifecycle/SPEC.md:170`): absence verification and
quiet-accounting now read reach, while yield, streaks and demotion still do not.

## Acceptance criteria

**AC1 — reach is computed and recorded per scenario.**
Given a scenario execution completes,
When the manifest is written,
Then the per-scenario result carries a machine-readable reach verdict, and an
execution judged not to have entered the plugin flow is recorded `unreached`
alongside its existing `status` and `probe` fields.

**AC2 — the predicate is deterministic and uses data already captured.**
Given reach must be decided by code, never a model,
When the verdict is computed,
Then it derives from the manifest's own inputs — a declared probe's `required`
predicate matching, or a per-scenario turns/events floor when no probe is
declared — both of which the manifest already records (`num_turns`, `events`,
`probe`, `checkpoints`).

**AC3 — reach never collapses into `status` or `probe`.**
Given the two-axis rule at `commands/tanuki.md:413`,
When reach is rendered anywhere,
Then it is a third fact, not a merged verdict: a scenario may be
`status: ok` + `unreached`, and `probe: undeclared` never implies reached.

**AC4 — the run summary marks it distinctly.**
Given an operator reads the run summary or dashboard,
When any scenario was `unreached`,
Then it is visually distinct from `ok` — silence must not read as a clean
result.

**AC5 — yield accounting is untouched.**
Given the amended invariant keeps coverage out of yield/streak/demotion,
When `record-run` computes yield,
Then an `unreached` scenario that nevertheless produced actionable findings
still counts as productive, exactly as before.

## Story questions (unverified — not acceptance criteria)

- What turns/events floor separates unreached from reached for an undeclared
  probe? The observed run showed 3/3/9/13 turns for short-circuited scenarios
  against 34 for one that reached its flow, but that is one run's evidence, not
  a threshold. Choosing it is implementation work and should be configurable
  rather than hard-coded.
- Does `probe: reached` alone suffice where a probe *is* declared, or must the
  floor also apply? Not decided.
