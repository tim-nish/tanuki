---
story: 1.10
epic: 1
title: "Story 1.10: dashboard 'this run' shows the integration commit subjects"
status: published
issue: 115
---

# Story 1.10: dashboard 'this run' shows the integration commit subjects

Canonical discussion record: https://github.com/tim-nish/tanuki/issues/115

As an operator (or a cold reader) reading the loop dashboard,
I want the "this run" section to show what the integration commit(s) actually changed,
So that I can tell what landed without leaving the dashboard to diff the branch by hand.

## Context / decision

Triaged from #115 (story lane, 2026-07-18). The dashboard is contractually a
**state-file-only reader** (`specs/spec-tanuki-view/SPEC.md` D4 "render, don't
compute"; `--follow` safety depends on it), so the fix must not add a `git`
call to the render path. **The chosen resolution is contract-preserving:**
`iter-verify` records each commit's subject (and changed-file count) into
`state.json` for the iteration, and the dashboard renders those recorded
values — no spec text changes and the render-don't-compute invariant stays
intact. On conflict the issue wins.

## Acceptance Criteria

- **Given** an iteration that landed one or more commits, **when** `iter-verify`
  closes it, **then** each commit's subject line and changed-file count are
  recorded into that iteration's `state.json` record.
- **Given** a closed run with recorded commit subjects, **when** the dashboard
  renders the "this run" section, **then** it lists each landed commit's subject
  (and file count) read solely from state — no `git` invocation in the render
  path.
- **Given** a no-patch iteration (`iter-verify --no-patch`, no commit), **when**
  the dashboard renders, **then** the "this run" section shows no phantom commit
  line and reads cleanly.
- **Given** a run predating this change (no recorded subjects in state), **when**
  the dashboard renders, **then** it degrades to a typed one-line note rather
  than erroring — the same state-file-only reader discipline.
- A test exercises the iter-verify → state → dashboard path so a cold reader
  sees the landed subjects without diffing the branch.
