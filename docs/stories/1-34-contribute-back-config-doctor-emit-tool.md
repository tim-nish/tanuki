---
story: 1.34
epic: 1
title: "Story 1.34: contribute_back config block, doctor validation, deterministic emit tool, contributed ledger status"
status: published
issue: 187
umbrella: 178
---

# Story 1.34: contribute_back config block, doctor validation, deterministic emit tool, contributed ledger status

Umbrella: https://github.com/tim-nish/tanuki/issues/178 (spec:
specs/spec-den-distill/SPEC.md §1–§3) — on conflict, the spec wins.

As a tanuki operator whose knowledge hub has a ratified staging intake,
I want an opt-in `contribute_back` config block (`{path, schema}`,
doctor-validated) and a deterministic zero-dep emit operation that renders a
promoted lesson candidate into one schema-conforming staging-intake file
with pinned provenance, recording a `contributed` ledger status,
So that a chronic pattern Tanuki discovered can be staged into the hub
mechanically, dedupe-safe, without any write ever landing outside the
configured staging directory.

## Context / decision

Decomposed from #178 (spec lane, ratified 2026-07-19). Selection uses the
existing promotion bar (chronic ≥3 / breadth ≥2 — no new thresholds); the
judgment happened at promotion, rendering is mechanical. The write surface
is exactly the configured staging directory — proposal-only,
staging-directory-only, the den stays raw history. This story is the
mechanism only; no interactive surface (1.35/1.36) calls it yet.

## Acceptance Criteria

- Given `contribute_back` unset, when anything runs, then behavior is
  byte-identical to today (the feature does not exist).
- Given a set block, when doctor validates, then path-exists /
  is-a-directory / schema-template-resolves are checked before any drive
  consumes it, failures named.
- Given a candidate that cleared the promotion bar, when emitted, then
  exactly one file lands in the configured staging directory, schema-valid
  against the configured template (frontmatter: slug, created date,
  `source_repo`, perishable flag, tags; body in full sentences), carrying
  finding id(s), recurrence/breadth counts, and run-manifest pointers.
- Given an emitted candidate, when later runs re-observe the pattern, then
  recurrence bumps but no duplicate staging file is emitted (`contributed`
  status is the dedupe key).
- Given a dismissed candidate, when emit selection runs, then it is never
  emitted and stays deduped-against.
- Given any emit, when it writes, then a test fixture asserts no write lands
  outside the configured staging directory (the allowlist).

## Out of scope

- The decision-pass walk and accept-time invocation (1.35).
- The "distill den" picker action and unconfigured notice (1.36).
- Any hub-side behavior — acceptance into the hub is the hub's gate.
