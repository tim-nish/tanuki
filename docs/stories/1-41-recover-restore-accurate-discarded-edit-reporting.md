---
story: 1.41
epic: 1
title: "Story 1.41: recover --restore misreports the uncommitted edits it discards (empty discarded list; truncated first filename)"
status: published
issue: 218
---

# Story 1.41: recover --restore reports discarded uncommitted edits accurately

Issue: https://github.com/tim-nish/tanuki/issues/218 — the canonical
discussion record; on conflict, the issue wins.

As a tanuki-loop operator running `recover --restore`,
I want the report of discarded uncommitted working-tree edits to list every
discarded path, accurately spelled,
so that I can trust the report when deciding whether the restore threw away
work I still needed.

## Context (from the issue)

Two sightings of one defect area — the report of discarded uncommitted
working-tree edits is untrustworthy:

- `discarded: []` rendered while edits were visibly thrown away
- the first filename is truncated (`pp.py` for `app.py`) because the `git()`
  helper strips the leading column off the first `git status --porcelain`
  line before splitting

Tanuki findings: F168, F186 (merge group).

## Acceptance criteria

1. **Given** a worktree with uncommitted edits (e.g. a modified `app.py`),
   **When** `recover --restore` discards them,
   **Then** the report's discarded list names every discarded path exactly
   (no empty list, no truncated first entry).

2. **Given** any caller parsing `git status --porcelain` output through the
   `git()` helper,
   **When** the helper returns output,
   **Then** leading status columns are preserved on every line — only
   trailing newlines are stripped.

3. **Given** the `git()` porcelain-stripping fix,
   **When** every porcelain-parsing call site is audited,
   **Then** each caller is verified (or corrected) to parse the preserved
   leading-column format correctly, with a test covering the first-line
   case that previously truncated.
