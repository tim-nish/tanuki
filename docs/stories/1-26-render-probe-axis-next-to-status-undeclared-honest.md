---
story: 1.26
epic: 1
title: "Story 1.26: Run surfaces render the probe axis next to status, with undeclared shown honestly"
status: published
issue: 166
umbrella: 163
---

# Story 1.26: Run surfaces render the probe axis next to status, with undeclared shown honestly

Umbrella: https://github.com/tim-nish/tanuki/issues/163 (spec: specs/spec-tanuki-trajectory/SPEC.md §2b "Rendering"; specs/spec-tanuki-view/SPEC.md D3/D4) — on conflict, the amended spec wins.

As a tanuki operator reading a completion report, run summary, or dashboard,
I want the `probe` axis rendered prominently next to `status` for every scenario, straight from the manifest,
So that a short-circuited scenario can never read healthier than one that completed its probe, and absent coverage is visible as absent rather than as health.

## Context / decision

Decomposed from #163 (spec lane, 2026-07-19). Rendering is governed by
spec-tanuki-view D3 (a view never renders a silent nothing) and D4 (the surface
renders; the tools compute): every rendered value is copied mechanically from
manifest.json, no model-derived or inferred field. Depends on story 1.25
(#165) for the manifest axis.

## Acceptance Criteria

- Given a completed drive, when the completion report and run summary render, then each scenario line shows both axes (`status` and `probe`, plus the matched/unmatched checkpoint split when declared).
- Given a scenario with `probe: undeclared`, when any surface renders it, then it reads "coverage not assessable — no probe declared" — never an implicit healthy state and never a silent nothing (D3).
- Given the loop dashboard's latest-drive section, when it aggregates scenario results, then the probe axis is included and every rendered value is copied mechanically from manifest.json (D4).
- Given any view, when it summarizes health, then it never merges `status` and `probe` into a single verdict, and never feeds `probe` into yield/streak/demotion displays as if it were yield.

## Out of scope

- The F179 `turns_anomalous` flag — its comparison statistic is deliberately undecided and gated on this contract landing; the field is absent until then, and when it later exists, absent must render as "not computed", never "not anomalous".
