---
story: 1.46
epic: 1
title: "Story 1.46: status delivered/settlement carries a typed empty-state on hostless targets"
status: published
issue: 240
depends_on: [1.47]
---

# Story 1.46: status settlement typed empty-state (hostless)

Issue: https://github.com/tim-nish/tanuki/issues/240 — the canonical
discussion record; on conflict, the issue wins.

As a cold reader of `status` on a hostless target,
I want the `delivered`/`settlement` fields to carry a typed empty-state label
instead of a bare `null`,
So that I can tell "not applicable for this target type" from "a delivery was
expected but did not happen".

## Context / decision

On a hostless target, `status`'s `delivered`/`settlement` render as `null`
with no label, conflating N/A with not-yet-delivered. This follows the
enumerated empty-states convention. **Depends on story 1.47 (#236):** the
provenance-aware empty-states rule ratified there is the contract this story
applies to the settlement fields — implement 1.46 against that enumeration,
not a bespoke one.

## Acceptance criteria

- **Given** a hostless target, **when** `status` renders `delivered`/
  `settlement`, **then** the fields carry a typed empty-state (e.g.
  `n/a — hostless`) with `expected: true`, not a bare `null`.
- **Given** a hosted target whose delivery has not happened yet, **when**
  `status` renders, **then** the settlement empty-state is distinct from the
  hostless N/A case (e.g. `pending`/`none`, `expected` per its run state).
- The new empty-state member(s) are added to the closed enumeration with one
  test fixture each (no prose-only reason at a call site).

## Out of scope

- The dashboard/view GAP-line provenance rule itself (story 1.47 / #236).
