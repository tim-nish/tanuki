---
story: 4.2
epic: 4
title: "Story 4.2: The decision pass completes promote \\u2192 decide \\u2192 file with no raw commands"
status: published
depends_on: ["4.1"]
issue: 54
---

# Story 4.2: The decision pass completes promote → decide → file with no raw commands

As a tanuki operator,
I want `/tanuki <target> --brief` (and the end of a normal run) to walk every proposal and watching finding, take my accept/dismiss/defer decision interactively, and file accepted issues itself after explicit confirmation,
So that I go from "run finished" to "issues filed, ledger updated" answering questions only (closes the workflow half of F23).

**Acceptance Criteria:**

**Given** a target with `proposed` findings
**When** the decision pass runs
**Then** proposals are walked top-down in small batches, each asking accept / dismiss / defer via the interactive question interface with no default disposition
**And** every disposition is written back via `tanuki-ledger set-status` by the command, never typed by the operator.

**Given** I accept a finding
**When** the pass offers filing
**Then** filing is a separate explicit confirmation, and on yes the command runs `gh issue create` with title = the human-readable problem statement (no prefixes), exactly one `<prefix>:<kind>` label (created if absent), and the finding id in the body footer.

**Given** below-bar ("watching") findings exist
**When** the pass finishes the proposals
**Then** watching findings are listed with priority and I can accept any of them right there (bar gates surfacing, not permission), with the same confirmed-filing offer.

**Given** the pass ends
**When** the command reports
**Then** it states what changed (per finding: disposition, issue URL if filed) and what remains undecided
**And** `commands/tanuki.md` documents this flow and `docs/tanuki-spec.md` carries the D5 acceptance rule: the command layer renders every human-needed view; a workflow forcing raw ledger JSON reading is a defect to file as a finding.

**Given** an unattended `tanuki-loop` run
**When** any loop phase executes
**Then** no disposition is taken and no issue is filed — the decision pass remains attended-only (NFR1, NFR3).
