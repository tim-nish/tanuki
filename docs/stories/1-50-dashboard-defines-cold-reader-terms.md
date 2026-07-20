---
story: 1.50
epic: 1
title: "Story 1.50: Dashboard defines its cold-reader terms (convergence, exploration quota, frozen, recurred)"
status: published
issue: 254
---

Canonical discussion record: https://github.com/tim-nish/tanuki/issues/254

**As** a cold reader of `tanuki-loop dashboard`,
**I want** each piece of loop jargon glossed where it first appears (or in a
short legend line),
**so that** I can answer the dashboard's five operational questions from the
screen alone, without a trip to the spec.

The terms that currently appear undefined: **convergence** (used before it is
defined), **exploration quota** (no gloss), **frozen** (overloaded as both a
count and a state), and **recurred** (unexplained). Definitions must be terse —
the goal is orientation, not a glossary — and must not restate meanings that
conflict with `specs/spec-tanuki-loop/SPEC.md`; on conflict, the spec wins. On
any conflict between this story and the issue, the issue wins.

## Acceptance criteria

- **Given** a cold reader running `tanuki-loop dashboard`, **when** the output
  first uses **convergence**, **then** a one-line gloss appears at (or before)
  that point, so the term is never used before it is defined.
- **Given** the dashboard shows the **exploration quota**, **when** it is
  rendered, **then** a terse inline definition accompanies it.
- **Given** **frozen** appears as both a count and a state, **when** the
  dashboard renders it, **then** the two uses are disambiguated (e.g. distinct
  wording or a one-line note) so a reader is not left guessing which is meant.
- **Given** the dashboard reports items that **recurred**, **when** rendered,
  **then** a one-line gloss explains what "recurred" means.
- **Given** all four terms are glossed, **when** a reader answers the
  dashboard's five operational questions, **then** they can do so from the
  screen alone — verified by the definitions being present and terse (inline at
  first use, or a single short legend line), adding no more than orientation.
- **Given** the change lands, **when** the dashboard fixtures run
  (`tools/tests/test-loop-dashboard`), **then** they assert the four terms'
  definitions are present, so the glosses cannot silently regress.
