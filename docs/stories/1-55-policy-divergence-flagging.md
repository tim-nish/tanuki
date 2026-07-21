---
story: 1.55
epic: 1
title: "Story 1.55: Policy-divergence flagging — proposal-only divergence candidates at pinned policy reads"
status: ready
umbrella: 271
---

Canonical discussion record: https://github.com/tim-nish/tanuki/issues/271
(owner decision record 2026-07-21, policy-advisory extension adoption). On any
conflict between this story and the issue, the issue wins.

Governing invariant (added by the 2026-07-21 spec decision, triage of #271):
**"§7 Policy-divergence flagging"** in `specs/spec-policy-advisory/SPEC.md` —
divergence flags are proposal-only, never auto-applied, never authoritative,
and active only when `policy_source` is configured.

**As** an operator running the decision pass with a `policy_source` configured,
**I want** implementation decisions that contradict or have outgrown a quoted
policy line to be flagged as divergence candidates,
**so that** drift between what the code/plan does and what my recorded policy
says surfaces at the gate — without anything being changed on my behalf.

## Acceptance criteria

- **Given** a configured `policy_source` and an implementation decision that
  contradicts or has outgrown a quoted policy line, **when** the gate reaches
  the pinned policy read, **then** it emits a **divergence candidate** as a
  **proposal-only** item carrying the divergence clause and the pinned quote
  (`file:line@commit`), attributed as policy advisory.
- **Given** a divergence candidate, **then** it is **never auto-applied and
  never authoritative** — the operator decides its fate exactly as for any
  proposal, and nothing about the code, the plan, or the policy repo is written.
- **Given NO** `policy_source` configured, **when** any run executes, **then**
  behavior and output are **byte-identical to today** (no divergence flags).
- **Given** the change, **then** the vocabulary is **generic** (zero
  policy-source-specific identity), activation rides the existing
  `policy_source` key (no new config surface), and the publication-boundary lint
  extends to divergence flags (run-output/audit only, never persisted as
  events, findings, or scheduler state).
