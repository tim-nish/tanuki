---
story: 5.9
epic: 5
title: "Story 5.9: the morning queue's rendering is capped and ranked"
status: published
issue: 310
umbrella: 310
---

# Story 5.9: the morning queue's rendering is capped and ranked

Canonical discussion record: [#310](https://github.com/tim-nish/tanuki/issues/310).
On conflict, the issue wins. Coupled decision with #309 (story 5.8): one
2026-07-27 ruling (alternative A) covers both.

Ruling: served policy line, consulted 2026-07-27 (verbatim pin held in the
private triage ledger): "QUEUE.md is itself a
deferral generator, so it is capped (~10 visible) and RANKED"
(topics/knowledge-architecture.md:98). The cap bounds presentation, never the
discharge duty (story 5.8 walks the full file).
Spec text: `specs/spec-tanuki-loop/SPEC.md` §"Morning gate" ("capped (~10
visible) and ranked").

## Story

As the operator opening a long-running target's morning queue, I want the
rendered queue capped and ordered by priority, so that what to read first is
a property of the surface rather than my scroll position.

## Acceptance criteria

**AC1 — capped rendering.**
Given a `queue.md` with more entries than the cap (~10),
When the queue is rendered at the morning gate (or on the dashboard queue
line),
Then at most the cap is shown, with an explicit "+N more — full: queue.md"
marker; the full file is untouched.

**AC2 — ranked by ledger priority.**
Given queued entries whose findings carry the ledger's computed priority,
When the visible set is chosen,
Then it is the top of a ranking by that priority (ties broken
deterministically), not file order.

**AC3 — truncation never hides justification.**
Given a rendered entry,
When its reason string is truncated to one line,
Then the explicit ellipsis marker and the `… — full: queue.md` pointer are
present, per the existing dashboard free-text rule
(`specs/spec-tanuki-loop/SPEC.md` §"No model-supplied text except quoted
reasons").

## Story questions

- Which ledger field is "the computed priority" for a queue entry whose
  deferral is not finding-backed (e.g. a credentials blocker)? Unverified —
  the queue entry shape in `tools/tanuki-loop` was not read this sitting; if
  entries can be finding-less, the ranking needs a declared fallback ordering.
