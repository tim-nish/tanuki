---
story: 1.49
epic: 1
title: "Story 1.49: gate-pr delivers a ready-for-review PR by default (Draft opt-in)"
status: published
issue: 251
---

Canonical discussion record: https://github.com/tim-nish/tanuki/issues/251

**As** an operator running `tanuki-loop` against a PR-protected target,
**I want** `gate-pr` to deliver a **ready-for-review** PR by default once the final
integration-HEAD test passes,
**so that** required status checks and review-request automation actually engage the
delivered PR without me having to run `gh pr ready` before the review gate can begin.

This implements the ready-by-default amendment to the delivery-boundary ruling
(`specs/spec-tanuki-loop/SPEC.md`, "Delivery is the loop's boundary", owner ruling
2026-07-17 / ready-by-default amendment 2026-07-20). The delivery/ratification boundary
is unchanged: still **no merge, no poll, no wait, no auto-merge** — only the PR's draft
status flips from Draft to ready-for-review. On any conflict between this story and the
issue, the issue wins.

## Acceptance criteria

- **Given** a PR-protected target (`gate: "pr"`) whose run has closed
  (`finish --reason cap|converged`) and whose recorded `test` pass matches the current
  integration HEAD, **when** `gate-pr` runs with no draft opt-in, **then** it opens one
  PR `integration → base` **marked ready for review**, and the loop ends there — no
  merge, poll, comment, or wait is added.
- **Given** the scenarios `loop` block sets `gate_pr_draft: true` (or `gate-pr` is
  invoked with `--draft`), **when** `gate-pr` runs, **then** it opens the PR as a
  **Draft** — the preserved 2026-07-17 behavior, now opt-in.
- **Given** `gate-pr` is re-run for a head branch whose PR already exists, **when** the
  PR is already in the target draft/ready state, **then** the step is a no-op (idempotent,
  failure-safe) and never errors; a PR in the other state is transitioned to the target
  state, not duplicated.
- **Given** the final integration-HEAD test has **not** passed against the current HEAD,
  **when** `gate-pr` runs, **then** it refuses (exit 3 + remedy, nothing mutated) exactly
  as the existing preconditions require — ready-for-review is only asserted after the
  recorded pass matches current HEAD.
- **Given** the change lands, **when** an operator reads `gate-pr --help` and the PR-gate
  section of `commands/tanuki-loop.md`, **then** the draft/ready lifecycle is documented:
  ready-by-default, the `gate_pr_draft` / `--draft` opt-in, and that the loop still never
  merges or waits.
