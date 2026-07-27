---
story: 5.12
epic: 5
title: "Story 5.12: accepting a lesson candidate appends the journey entry"
status: published
issue: 298
umbrella: 298
---

# Story 5.12: accepting a lesson candidate appends the journey entry

Canonical discussion record: [#298](https://github.com/tim-nish/tanuki/issues/298).
On conflict, the issue wins.

Ruling (2026-07-27 triage, alternative A): the journey-doc convention is
adopted — the append is an accepted-item effect in the emitting repo, the
proposal-only boundary untouched. Served policy, consulted at
`product-lab@d261caee84c9e617818b5f92c8757fbb2decf1fa`: "NEEDS-RECORDING is
COMPUTED in the join step and DISCHARGED by a tracking artifact in the target
repo (an Issue, or an append under a NEEDS-RECORDING heading in that repo's
declared journey doc)" (topics/articles.md:49).
Spec text: `specs/spec-den-distill/SPEC.md` §4 ("Accept also appends the
journey entry") and the amended Acceptance allowlist line.

## Story

As the operator accepting a lesson candidate in the decision pass, I want the
same sitting to record the concrete episode in `docs/journey.md`, so that this
repo's own prose holds the atomic facts harvest can read, keyed to the hub
lesson they produced.

## Acceptance criteria

**AC1 — the file exists and is append-only.**
Given this story lands,
Then `docs/journey.md` exists with a header stating the entry shape and the
append-only rule (entries are harvest pins, never edited retroactively).

**AC2 — accept appends one entry.**
Given a lesson candidate is accepted in the decision pass
(`specs/spec-den-distill/SPEC.md` §4, "Accept writes the file right then"),
When the accept completes,
Then one entry is appended to `docs/journey.md` carrying: date, the concrete
episode as an atomic fact (event, number, quote, or result), the finding and
issue ids, and the hub lesson slug — and the accept's in-session receipt
names the journey append alongside the emitted file path.

**AC3 — the write allowlist admits exactly this.**
Given the fixture-asserted allowlist
(`specs/spec-den-distill/SPEC.md` §Acceptance),
When the accept path writes,
Then `docs/journey.md` is the only write outside the configured emission
directory.

**AC4 — unconfigured behavior unchanged.**
Given `contribute_back` is unset,
Then behavior is byte-identical to today: no journey write, no journey file
required.

## Story questions

- Where is the accept path implemented (the emit-tool call site in the
  decision pass)? Unverified — not read this sitting.
- Is `docs/` inside writing-assistant's declared `include:` set for this
  repo, as #298 assumes? Cannot-determine — the declared set was not
  consulted this sitting; if it is not, the file placement needs revisiting
  before the harvest value materializes (the append duty itself is unaffected).
