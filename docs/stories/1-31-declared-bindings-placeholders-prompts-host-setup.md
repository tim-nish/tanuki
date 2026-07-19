---
story: 1.31
epic: 1
title: "Story 1.31: Declared bindings/placeholders resolve in scenario prompts and host_setup per host"
status: published
depends_on: ["1.30"]
issue: 184
umbrella: 173
---

# Story 1.31: Declared bindings/placeholders resolve in scenario prompts and host_setup per host

Umbrella: https://github.com/tim-nish/tanuki/issues/173 (spec:
specs/spec-tanuki-scenario-lifecycle/SPEC.md "Host bindings & declared-input
configuration") — on conflict, the issue and spec win, issue first.

As a tanuki operator with a portable charter,
I want scenario prompts and `host_setup` to reference named bindings (e.g.
`{subject_sources}`, `{draft_location}`) declared per target or per host,
resolved against the effective host binding at drive time,
So that the same charter runs against structurally different repositories
with zero prompt edits between runs.

## Context / decision

Decomposed from #173 (spec lane, ratified 2026-07-19). Bindings are
target-declared and opaque — no target semantics inside Tanuki
(writing-assistant#409 acceptance rule). Undeclared literal paths remain
legal; detecting them is 1.32's business, not an error here.

## Acceptance Criteria

- Given a scenario whose prompt/host_setup references declared bindings,
  when driven under a host binding that declares values for them, then every
  placeholder resolves before the driver sees the text, and the manifest
  records the resolved values.
- Given a referenced binding with no value under the effective host, when
  the drive is planned or started, then it fails closed naming the missing
  binding — never a prompt with a raw `{placeholder}` handed to the driver.
- Given a scenario with no placeholders, when driven, then behavior is
  unchanged (literals stay legal).
- Given the same charter and two hosts with different binding values, when
  driven against each, then both runs succeed with zero edits to the
  scenario text (the acceptance target's substrate).

## Out of scope

- Deciding which scenarios are portable (1.32 reports it).
- Fixture selection per host (1.33).
