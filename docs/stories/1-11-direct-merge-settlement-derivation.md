---
story: 1.11
epic: 1
title: "Story 1.11: derive delivered/settlement on the direct-merge gate from reachability"
status: published
issue: 117
umbrella: 117
---

# Story 1.11: derive delivered/settlement on the direct-merge gate from reachability

Canonical discussion record: https://github.com/tim-nish/tanuki/issues/117

As an operator using a non-PR-protected target (direct-merge gate),
I want status.delivered/settlement to populate after a successful merge,
So that a landed run reads as landed instead of showing null, which looks like a lost delivery.

## Context / decision

Triaged from #117 (spec lane, 2026-07-18). Governing text amended in
`specs/spec-tanuki-loop/SPEC.md` §Settlement: settlement is now specified to
derive on the direct-merge path from the integration tip's reachability from
the current base — the same strict test the PR path uses — populating
`delivered`/`settlement` symmetrically. This story wires the derivation into
`tools/tanuki-loop`'s read-only status/dashboard surfaces. On conflict the
issue and the amended spec win.

## Acceptance Criteria

- **Given** a direct-merge (non-PR) gate run whose `integration → base` merge
  has landed, **when** `status`/`dashboard` render, **then** `settlement` is
  `landed` and `delivered` is populated from the local merge (integration-tip
  SHA, base SHA, merge commit) — read-only, derived, never declared.
- **Given** a direct-merge run whose merge has not yet run (or the base moved
  past the tip), **when** status renders, **then** `settlement` is `pending`,
  not `null`.
- **Given** the base ref is unreadable, **when** status renders, **then**
  `settlement` is `unknown` — never an optimistic default.
- **Given** a PR-gate (`gate=pr`) run, **when** status renders, **then** its
  settlement derivation is unchanged (no regression to the PR path).
- A test drives a direct-merge gate to a landed merge and asserts
  `settlement: landed` + a populated `delivered`, and asserts the pre-merge
  state is `pending` not `null`.
