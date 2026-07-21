---
story: 1.52
epic: 1
title: "Story 1.52: Two-outcomes-only delivery — remove direct-merge-to-main; PR is the default gate"
status: ready
umbrella: 263
---

Canonical discussion record: https://github.com/tim-nish/tanuki/issues/263
(structural constraint), subsuming https://github.com/tim-nish/tanuki/issues/262
(PR-as-default). On any conflict between this story and the issues, the issues win.

Governing invariant (added by the 2026-07-21 spec decision, triage of
#262/#263): **"Two-outcomes-only delivery"** in `specs/spec-tanuki-loop/SPEC.md`
— the loop's terminal delivery is exactly one of two states and the loop
**never** merges to `main`. This story implements the harness half of that
contract: it removes the direct-merge-to-`main` code path and makes PR delivery
the default.

**As** an operator running `tanuki-loop` on any target,
**I want** the loop to deliver its work only as a PR from the integration
branch (or by leaving the branch in place), never by merging into `main`
itself,
**so that** loop output can never reach `main` without passing through a
reviewable PR — eliminating by construction the footgun where a direct
`git merge integration → main` + `gate-push` landed on an unprotected `main`
with no review sink (motivating incident, 2026-07-21).

## Acceptance criteria

- **Given** any target's `loop` config, **when** a run reaches delivery,
  **then** the loop has **no** code path — attended or unattended — that merges
  the integration branch into `main`/base; the local `git merge integration →
  main` + `gate-push`-onto-base sequence is removed from `tools/tanuki-loop`
  (and from the reconcile terminal step, #210).
- **Given** a target with no explicit gate override, **when** a run closes
  successfully, **then** PR delivery (`gate-pr`, `integration → base`) is the
  default — the former `gate: "merge"` default no longer exists, and repeated
  runs deliver a PR consistently (no run silently local-merges by default),
  verified across repeated runs.
- **Given** a target configured `gate: "branch"`, **when** a run closes,
  **then** the loop leaves the integration branch in place, opens no PR, and
  records a branch-only terminal fact (integration-tip SHA, base SHA) — the
  second of exactly two allowed terminal states.
- **Given** `gate-push`-onto-base is invoked (directly or by any legacy
  caller), **when** it would land loop output on `main`, **then** it is removed
  or repurposed so it can never do so without a PR.
- **Given** any `gate`/delivery configuration, **when** a run completes,
  **then** no configuration can produce an unattended merge; the human review
  gate before merge is preserved in every mode (config selects delivery form
  only).
- **Given** settlement derivation, **when** a branch-only run's tip becomes
  reachable from the base (the human merged it), **then** `status.delivered` /
  `settlement` populate by reachability exactly as the PR path does — a landed
  branch-only run never shows `settlement: null`.
- **Given** the change lands, **when** the `tools/tests/test-*` suite runs,
  **then** fixtures assert: (1) no delivery path merges into `main`, (2) a
  default-gate run terminates in a PR, and (3) a `gate: "branch"` run
  terminates branch-only with a recorded terminal fact.
