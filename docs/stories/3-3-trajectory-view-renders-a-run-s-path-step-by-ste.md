---
story: 3.3
epic: 3
title: "Story 3.3: Trajectory view renders a run's path step by step"
status: ready
depends_on: ["3.1"]
---

# Story 3.3: Trajectory view renders a run's path step by step

As a Tanuki operator,
I want `history --scenario <id> --trajectory` to render each run of a scenario
as its seq-ordered path,
So that I can read the simulated user's actual journey — Q→choice,
error→recovery→outcome — without opening raw logs.

**Acceptance Criteria:**

**Given** a scenario with runs carrying typed events,
**When** `tanuki-scheduler --target <t> history --scenario <id> --trajectory
[--run <run>]` is invoked,
**Then** each run renders newest first (`--run` filters to one) as a
seq-ordered path, one compact line per step — choices as
`#<seq> user_choice <point>: selected=<value> (alt: …)`, errors, recoveries as
`of #<seq>`, and the outcome line — matching the spec §3 shape. `--trajectory`
is a boolean mode flag composing with the existing `--scenario` selector; no
second scenario-valued flag is introduced. (FR10)

**Given** the renderer runs,
**Then** it reads only the per-run event files
(`~/.tanuki/<target>/events/<run>/<scenario>.events.jsonl` plus their
`raw.jsonl` evidence pointers), prints, and exits — it writes nothing, judges
nothing, scores nothing, and loads no ledger (verified). (FR11, NFR4)

**Given** a run that predates typed events,
**Then** it degrades to the `tool_error`/`result` chain with a
"(pre-trajectory run — choices not recorded)" note. (FR12)

**Given** `--trajectory` is absent,
**Then** the plain `history` view (pins/yield/findings) is byte-identical to
its pre-story output — trajectory is strictly additive behind the flag, and
every existing test stays green. (FR13, NFR5)
