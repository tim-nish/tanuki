---
story: 4.1
epic: 4
title: "Story 4.1: Promotion moves findings to proposed, preview via --dry-run"
status: published
issue: 53
---

# Story 4.1: Promotion moves findings to proposed, preview via --dry-run

As a tanuki operator,
I want `promote` to actually transition qualifying findings to `proposed` (with a `--dry-run` preview),
So that promoting is one step instead of a preview plus N unhinted `set-status` calls (closes F23).

**Acceptance Criteria:**

**Given** a ledger with findings that meet the promotion thresholds
**When** I run `tanuki-ledger --target <t> promote`
**Then** each qualifying finding's status changes from `open` to `proposed` and the output lists exactly which ids transitioned
**And** findings already `proposed`/`accepted`/`dismissed` are never touched.

**Given** the same ledger
**When** I run `promote --dry-run`
**Then** the qualifying findings are listed with their computed priority and nothing in the ledger changes (byte-identical `ledger.json`).

**Given** no findings meet the thresholds
**When** I run `promote` (with or without `--dry-run`)
**Then** the output states the active thresholds and that nothing qualified, and the ledger is unchanged.

**Given** the tool's help and the contract docs
**When** I read `promote --help` and `tanuki-ledger`'s module docstring
**Then** both describe the transition semantics and `--dry-run` (the "Preview only — never mutates state" wording is gone), consistent with `docs/tanuki-spec.md`'s "Ledger status of promoted findings moves to `proposed`"
**And** a `tools/tests/` fixture covers transition, dry-run, and no-qualifier cases.
