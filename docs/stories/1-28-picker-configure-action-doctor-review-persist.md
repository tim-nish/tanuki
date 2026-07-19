---
story: 1.28
epic: 1
title: "Story 1.28: Configure-target action in the /tanuki picker with doctor hook and review-before-persist"
status: published
depends_on: ["1.27"]
issue: 181
umbrella: 172
---

# Story 1.28: Configure-target action in the /tanuki picker with doctor hook and review-before-persist

Umbrella: https://github.com/tim-nish/tanuki/issues/172 (spec:
specs/spec-tanuki-scenario-lifecycle/SPEC.md "Host bindings & declared-input
configuration") — on conflict, the issue and spec win, issue first.

As a tanuki operator,
I want "configure target" to be an action in the existing `/tanuki <target>`
action picker — showing effective config, running the target's opaque doctor
hook after a change, and applying only after I approve the shown diff and
storage path,
So that routine configuration never requires finding a hidden machine-state
file, and a broken value is caught before the next drive consumes it.

## Context / decision

Decomposed from #172 (spec lane, ratified 2026-07-19). No new top-level
command — command proliferation is itself a UX failure; the picker drives
the deterministic 1.27 operations and renders their output. The doctor hook
is target-owned and opaque: Tanuki shows the exact command line before
running it (the `test_cmd` posture), relays its output verbatim, and never
interprets it — Tanuki-side content knowledge is a contract bug
(writing-assistant#409 rule).

## Acceptance Criteria

- Given `/tanuki <target>`, when the action picker renders, then
  "configure target" is a named option; selecting it shows the effective
  configuration (1.27 `show`) with sources labeled and undeclared fields
  read-only.
- Given a change to a declared field with a doctor hook, when the value
  passes typed validation, then the doctor command line is shown, run, and
  its output relayed verbatim; a failing doctor blocks persist-by-default
  (the operator may explicitly override).
- Given a pending change, when the flow reaches persistence, then it shows
  the resulting diff and the exact storage path and applies only on
  approval — never silently.
- Given a decline at any step, when the flow ends, then nothing was written.
- Given a target with no `inputs` block, when "configure target" is chosen,
  then the surface says so in one line and points at declaring one — never a
  crash, never a guess.

## Out of scope

- The underlying show/set/check operations and validation (1.27).
- Per-run override lifecycle (1.29).
