---
story: 1.12
epic: 1
title: "Story 1.12: feature-drift as a third scenario-regeneration trigger"
status: published
depends_on: ["1.13"]
issue: 123
umbrella: 113
---

# Story 1.12: feature-drift as a third scenario-regeneration trigger

Umbrella issue: https://github.com/tim-nish/tanuki/issues/113

As a Tanuki operator whose target plugin keeps growing,
I want a feature-drift signal when the plugin gains surface no charter covers,
So that the matrix can't silently go stale and converge on a false quiet.

## Context / decision

Decomposed from #113 (spec lane, 2026-07-18). `specs/spec-tanuki-scenario-lifecycle/SPEC.md`
§3 now names three generation triggers (init, pool-empty, feature-drift). This
story implements the **feature-drift signal**: detect that the target gained a
skill/command/decision-point not covered by any existing charter, and surface
it as an advisory (never automatic) prompt to invoke `generate`. Detection is a
read-only signal; it never mutates the matrix.

## Acceptance Criteria

- **Given** a target whose plugin gained a skill/command/decision-point absent
  from every charter, **when** a read-only surface (history/dashboard) is
  viewed, **then** it reports a feature-drift signal naming the uncovered
  surface and the `generate` invocation.
- **Given** no uncovered surface, **when** the surface is viewed, **then** no
  drift signal appears (no false positives).
- The signal is advisory only: it never triggers generation automatically and
  never mutates the scenario matrix.
- A test asserts drift is reported for an added-but-uncovered surface and not
  reported when coverage is complete.

## Depends on

- 1.13 (`generate` first-class entry — the invocation the drift signal points at)
