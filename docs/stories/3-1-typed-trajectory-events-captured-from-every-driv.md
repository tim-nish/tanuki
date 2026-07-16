---
story: 3.1
epic: 3
title: "Story 3.1: Typed trajectory events captured from every drive"
status: ready
---

# Story 3.1: Typed trajectory events captured from every drive

As a Tanuki operator,
I want every driven scenario's decisions, recoveries, and terminal outcome
lifted mechanically into typed events,
So that downstream views can render the simulated user's actual path without
any LLM judgment or new tooling.

**Acceptance Criteria:**

**Given** any driven scenario,
**When** `tanuki-drive` injects its simulated-user preamble,
**Then** the preamble (drive's own boilerplate, never per-scenario prose)
instructs the driver to emit one
`TANUKI-CHOICE point=<name> selected=<value> [alternatives=<v1|v2|…>]` line
per decision and one terminal
`TANUKI-OUTCOME disposition=<success|gave_up|partial> <free text>` line. (FR1)

**Given** well-formed markers in `<scenario>.raw.jsonl`,
**When** the existing normalize pass runs,
**Then** each marker lifts verbatim (string match only) into a typed event in
`<scenario>.events.jsonl`, one-to-one — `user_choice` carrying
`point`/`selected`/optional `alternatives`, `outcome` carrying
`disposition`/`detail` — and malformed markers are ignored with a
`trajectory_markers_dropped` count in the run manifest. (FR2, FR3)

**Given** a `tool_error` followed (same scenario) by the next marker or tool
success whose detail names the same command/subcommand,
**When** normalize runs,
**Then** a `recovery` event is emitted whose `of_seq` is that `tool_error`'s
`seq` — and an error with no matching continuation yields no recovery event.
(FR4)

**Given** a completed scenario run whose driver emitted no outcome marker,
**When** normalize runs,
**Then** an `outcome` with `disposition=partial` is synthesized from the
driver `result` event, flagged `synthesized: true` — every completed run
yields ≥1 `outcome` event. (FR5)

**Given** the new typed events,
**Then** their evidence pointers resolve — `run/scenario#seq` names the event
in `events.jsonl` and the `raw.jsonl` pointer indexes the actual marker line
(the legacy event-seq pointer format is not inherited for typed events). (NFR8)

**Given** runs that predate the marker grammar,
**Then** nothing is backfilled; the existing exact-dupe collapse,
capped-exemplar evidence, and one-way flow apply to the new types unchanged;
and every existing test stays green with existing outputs byte-identical when
no typed events exist. (NFR5, NFR10)
