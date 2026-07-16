---
story: 4.3
epic: 4
title: "Story 4.3: The next step is derived from ledger state, exposed as `next`, and pinned by fixtures"
status: published
issue: 55
---

# Story 4.3: The next step is derived from ledger state, exposed as `next`, and pinned by fixtures

As a tanuki operator,
I want the "next:" hint computed from a closed enumeration of ledger states — also available as a `next` subcommand — with one test fixture per state,
So that "where was I / what now?" is always answerable and the hint can't silently regress a third time (closes F24/F36/F76).

**Acceptance Criteria:**

**Given** the ledger states: no ledger; events but no findings; findings all below bar; proposed awaiting decision; accepted awaiting fix-verification; all findings tombstoned or decided
**When** `status` renders its trailing line
**Then** the line comes from a single derivation function mapping each enumerated state to exactly one next command (a `/tanuki` form where one exists, the tool form otherwise), replacing the patched if/elif chain.

**Given** the same ledger
**When** I run `tanuki-ledger --target <t> next`
**Then** it prints only the derived next step, identical to `status`'s line (one derivation, two surfaces).

**Given** `tools/tests/`
**When** the test suite runs
**Then** one fixture per enumerated state asserts the exact hint, and a ledger state outside the enumeration fails visibly rather than falling through to a wrong hint
**And** the derivation reads only bounded views, never `ledger.json` whole (NFR5).
