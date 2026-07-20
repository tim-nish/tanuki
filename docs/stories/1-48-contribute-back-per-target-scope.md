---
story: 1.48
epic: 1
title: "Story 1.48: contribute_back resolves per-target only; global-scope block ignored and doctor-flagged"
status: published
issue: 241
umbrella: 241
---

# Story 1.48: contribute_back per-target scope

Issue: https://github.com/tim-nish/tanuki/issues/241 — the canonical
discussion record; on conflict, the issue wins. Umbrella spec decision: #241
(per-target-only scope, ratified 2026-07-20 —
`specs/spec-den-distill/SPEC.md` §2).

As a tanuki operator running distill against any target,
I want `contribute_back` resolved only from the target's own configuration,
never from the machine-wide `~/.tanuki/config.json`,
So that one target — or a stale scratch run — cannot silently route every
target's lessons into its hub.

## Context / decision

Decomposed from #241 (spec lane, ratified 2026-07-20). `contribute_back` is
the one exception to the `~/.tanuki/config.json < target defaults < CLI`
precedence: the global tier does not participate at all. A global-scope
block is ignored for resolution and flagged by doctor.

## Acceptance criteria

- **Given** `contribute_back` set only in `~/.tanuki/config.json` (global),
  **when** distill resolves it, **then** it is treated as **unset** (the
  unconfigured notice renders; nothing is emitted).
- **Given** `contribute_back` set in a target's scenarios-file `defaults`
  block, **when** distill resolves it, **then** it is used normally.
- **Given** a global-scope `contribute_back` block, **when** `doctor` runs,
  **then** it flags the block as ignored (a likely leftover from another
  target's session), naming the scope.
- Config-resolution and doctor tests cover the ignored-global, used-per-target,
  and doctor-flag cases.

## Out of scope

- The distill emit/receipt behavior (unchanged; only the resolution scope
  changes).
