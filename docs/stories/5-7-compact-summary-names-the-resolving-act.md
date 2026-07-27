---
story: 5.7
epic: 5
title: "Story 5.7: compact's summary line names the resolving act as one runnable command"
status: published
issue: 330
umbrella: 330
---

# Story 5.7: compact's summary line names the resolving act

Canonical discussion record: [#330](https://github.com/tim-nish/tanuki/issues/330).
On conflict, the issue wins.

Ruling (2026-07-27 triage, alternative A): the aggregate summary adopts the
resolving act — one runnable `tanuki-drive` command over the union of the
cannot-determine set's drivable scenarios — so the same fact is no longer
reported two ways in one tool, with the weaker form in the summary.
Spec text: `specs/spec-tanuki-scenario-lifecycle/SPEC.md` §"The dead end is
named before compaction" ("`compact`'s aggregate summary names the act too").

## Story

As an operator reading the awaiting-findings summary, I want the summary to
end with the exact command that makes absence measurable, so that I never
hand-trace what the ACCEPTED rows one surface above already tell me.

## Acceptance criteria

**AC1 — the summary emits the union command.**
Given awaiting findings whose absence verdict is cannot-determine and at least
one of which carries a drivable scenario,
When the summary line is rendered (`tools/tanuki-ledger:1536-1540`, the
"drive the scenario(s) to make absence measurable" tail),
Then it ends with one runnable command over the union of the set's drivable
scenarios, in the same form `resolving_action` renders per finding
(`tools/tanuki-ledger:1096-1117` — `tanuki-drive --target <t> --plugin <repo>
--scenarios <file> --run <run> --only <union>`), with the reserved `ingest`
pseudo-scenario excluded from the union.

**AC2 — the degenerate branch matches the per-finding renderer.**
Given every member of the cannot-determine set lacks a drivable scenario
(evidence is manual/ingest only),
When the summary is rendered,
Then it states that absence cannot be measured until the findings are seen in
a driven scenario — the same degenerate branch `resolving_action` takes
(`tools/tanuki-ledger:1108-1112`) — and never the generic
"drive the scenario(s)" with no command.

**AC3 — the ACCEPTED rows are unchanged.**
Given the same state,
When `status` renders ACCEPTED rows,
Then the per-row resolving act (`tools/tanuki-ledger:1346`, `:1691`) is
byte-identical to before this story.

## Story questions

- Do existing tests pin the current wording of the summary tail at
  `tools/tanuki-ledger:1540`? Cannot-determine — test coverage of that line
  was not located this sitting (commit `b3f637c` pins story 5.6's `status`
  wording; whether the compact/awaiting summary is also pinned was not read).
  If pinned, the pin moves with the ruling.
