---
story: 3.2
epic: 3
title: "Story 3.2: Coverage diff surfaces undeclared decision points as charter seeds"
status: published
depends_on: ["3.1"]
issue: 41
---

# Story 3.2: Coverage diff surfaces undeclared decision points as charter seeds

As a Tanuki operator,
I want `history` to diff the decision points my runs actually hit against the
points the matrix declares,
So that true exploration blind spots surface as plan-gated charter seeds
instead of waiting for a human to read raw logs.

**Acceptance Criteria:**

**Given** typed events exist and the matrix declares `axes`/`covers`,
**When** `tanuki-scheduler history --scenarios <file>` runs,
**Then** a coverage-diff block renders after the existing coverage/debt
section, listing each undeclared decision point (observed `user_choice.point`
absent from axes) with the scenarios and run count that hit it, the values
seen, `-> charter seed (plan-gated)`, and `run/scenario#seq` evidence
pointers. (FR6, FR7)

**Given** a marker introduces a `point` absent from `axes`,
**When** history renders,
**Then** it appears as a charter seed — and after the operator ratifies it via
the plan gate into `axes`/`covers`, it leaves the undeclared list on the next
render. The tool never writes the matrix. (FR7, NFR3)

**Given** declared axis values never observed in any trajectory,
**Then** they render as one pointer line deferring to the lifecycle spec's
`authored`/`uncovered` states, and when scenarios have executed runs that all
predate trajectory capture the block appends the
"(<k> scenario(s) executed before trajectory capture — observations
incomplete)" note. (FR8)

**Given** a matrix with no `axes` declared,
**Then** the block prints the observed-point list only, with the lifecycle
spec's existing "no axes declared" pointer; **and given** no typed events at
all, **then** it prints exactly "no trajectory events recorded — coverage diff
needs runs made after the marker grammar landed". (FR9)

**Given** the block's implementation,
**Then** it is deterministic set arithmetic over per-run event files — no LLM
stage, no ledger reads, no writes anywhere, and the pre-existing history
output above the block is unchanged. (NFR1, NFR4)
