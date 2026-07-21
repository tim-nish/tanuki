---
story: 1.54
epic: 1
title: "Story 1.54: Consult-first at the proposal gate (--brief) \u2014 covered forks auto-resolve to an overrideable FYI"
status: published
issue: 273
umbrella: 271
---

Canonical discussion record: https://github.com/tim-nish/tanuki/issues/271
(owner decision record 2026-07-21, policy-advisory extension adoption). On any
conflict between this story and the issue, the issue wins.

Governing invariant (added by the 2026-07-21 spec decision, triage of #271):
**"§6 Consult-first at the gate"** in `specs/spec-policy-advisory/SPEC.md` — the
single bounded relaxation of §4's "never pre-selects a disposition" rule,
applying to decision-point *forks* only, always overrideable, never
machine-final, and active only when `policy_source` is configured.

**As** an operator running the decision pass with a `policy_source` configured,
**I want** decision-point forks the policy source already answers to be
auto-resolved into an overrideable FYI (with the chosen option and a pinned
verbatim quote), and unanswered forks to escalate to me with up to 3 pinned
candidate answers,
**so that** the gate stops re-asking me forks I have already recorded a policy
for — while never taking a decision out of my hands.

## Acceptance criteria

- **Given** a configured `policy_source` and a decision-point fork the source
  covers, **when** the gate reaches that fork, **then** it auto-resolves the
  fork and demotes it to an **overrideable FYI** carrying the chosen option and
  a pinned verbatim quote (`file:line@commit`) — attributed as policy advisory,
  and reopenable by the operator.
- **Given** the same, **when** the operator overrides the FYI, **then** the
  override wins — the auto-resolution is **never machine-final**.
- **Given** a fork the policy source does NOT cover, **when** the gate reaches
  it, **then** it escalates to the operator with **up to 3 pinned candidate
  answers** (each `file:line@commit`) — ranked material, never a machine-final
  choice.
- **Given** a proposal's disposition (accept/dismiss/defer), **when** the gate
  walks it, **then** consult-first never auto-resolves it — the §4 rule for
  proposal dispositions is unchanged; only *forks* are auto-resolved.
- **Given NO** `policy_source` configured, **when** any run executes, **then**
  behavior and output are **byte-identical to today** (no forks auto-resolved,
  no candidates attached) — the silent-when-unconfigured check.
- **Given** the change, **then** the vocabulary is **generic** (zero
  policy-source-specific identity in code, UX, or docs), activation rides the
  existing `policy_source` key (no new config surface), and the
  publication-boundary lint extends to FYI quotes + candidate answers (never
  persisted as events, findings, or scheduler state).
