---
story: 1.44
epic: 1
title: "Story 1.44: normalize() tolerates markdown-decorated trajectory markers; never a silent drop"
status: published
issue: 238
---

# Story 1.44: normalize() tolerates markdown-decorated trajectory markers

Issue: https://github.com/tim-nish/tanuki/issues/238 — the canonical
discussion record; on conflict, the issue wins.

As a tanuki operator reviewing a run's trajectory,
I want trajectory markers to be lifted into typed `user_choice` events even
when the driven model wrapped them in markdown or indentation,
So that a decorated `**TANUKI-CHOICE ...**` line is never silently lost from
the trajectory record.

## Context / decision

`normalize()` currently lifts only clean `TANUKI-CHOICE ...` lines; a marker
wrapped in `**bold**`, or indented mid-paragraph, is neither lifted nor
counted in `trajectory_markers_dropped`. The invariant this restores: a
marker is either lifted into an event or counted as dropped — never a silent
miss.

## Acceptance criteria

- **Given** a raw stream with a `TANUKI-CHOICE` marker on its own clean line,
  **when** `normalize()` runs, **then** it is lifted into a typed
  `user_choice` event (unchanged from today).
- **Given** a marker wrapped in `**bold**`, **when** `normalize()` runs,
  **then** surrounding markdown decoration is stripped and the marker is
  lifted into a `user_choice` event.
- **Given** a marker indented mid-paragraph, **when** `normalize()` runs,
  **then** leading indentation/whitespace is tolerated and the marker is
  lifted.
- **Given** a marker that still cannot be lifted for any reason, **when**
  `normalize()` runs, **then** `trajectory_markers_dropped` is incremented so
  the loss is never silent.
- One test fixture per decoration form (clean, bold, indented) plus the
  drop-counter case.

## Out of scope

- Changing the marker syntax itself or the `user_choice` event schema.
