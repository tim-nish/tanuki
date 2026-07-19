---
story: 1.25
epic: 1
title: "Story 1.25: tanuki-drive evaluates probe predicates and records an independent probe axis in the manifest"
status: published
issue: 165
umbrella: 163
---

# Story 1.25: tanuki-drive evaluates probe predicates and records an independent probe axis in the manifest

Umbrella: https://github.com/tim-nish/tanuki/issues/163 (spec: specs/spec-tanuki-trajectory/SPEC.md §2b "Probe coverage — evidence predicates") — on conflict, the amended spec wins.

As a tanuki operator comparing runs,
I want tanuki-drive to evaluate each scenario's declared probe predicates over its recorded evidence in the existing normalize/manifest step and write a second, independent per-scenario result axis (`probe`) into manifest.json,
So that a short-circuited run is mechanically distinguishable from one that reached its probe, and invalid control cells can be excluded from performance comparisons without reading transcripts.

## Context / decision

Decomposed from #163 (spec lane, 2026-07-19; finding F178, recurrence 2).
Evidence: in the retained `post409-c1` writing-assistant run, `draft-survey`
quit after 3 turns and `draft-postmortem-f2` at 16 — both manifest `ok`,
indistinguishable from the full-depth `post409-c2` cells, confounding the whole
#409 measurement series. The verdict derives from recorded evidence, never
driver self-report (F113 precedent — the weak driver is the component under
measurement). Depends on story 1.24 (#164) for the charter schema.

## Acceptance Criteria

- Given a scenario whose required predicate matches its recorded events, when normalize/manifest completes, then its result carries `probe: reached`; declared-but-unmatched yields `probe: short_circuited`; no probe block yields `probe: undeclared`.
- Given declared checkpoints, when the manifest is written, then the result carries `checkpoints: {matched: [...], unmatched: [...]}` as an unordered split — no furthest-stage arithmetic beyond set membership.
- Given any probe outcome, when the manifest is written, then the existing `status` field is byte-for-byte what it is today: the axes are recorded independently and never merged into one verdict.
- Given evaluation, when it runs, then it is deterministic code inside tanuki-drive (no model in the loop) over the scenario's `*.events.jsonl` (or raw stream for `on: raw`) — never driver self-report.
- Given the scheduler, when a run completes, then yield/streak/demotion arithmetic is untouched by the probe axis (v1 orthogonality).
- Given a regression fixture replaying the F178 evidence shape (a scenario that quit at 3 turns with a declared probe vs one that ran to depth), when both are normalized, then the first reads `status: ok, probe: short_circuited` and the second `status: ok, probe: reached`.

## Out of scope

- Charter schema and plan-gate authoring (story 1.24, #164).
- Rendering of the axis (story 1.26, #166).
- The F179 `turns_anomalous` statistic — deliberately undecided, gated on this contract landing; the field stays absent.
