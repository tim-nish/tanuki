---
story: 1.32
epic: 1
title: "Story 1.32: Host-coupling detection reports portable/bound/host-coupled through existing read-only surfaces"
status: published
depends_on: ["1.31"]
issue: 185
umbrella: 173
---

# Story 1.32: Host-coupling detection reports portable/bound/host-coupled through existing read-only surfaces

Umbrella: https://github.com/tim-nish/tanuki/issues/173 (spec:
specs/spec-tanuki-scenario-lifecycle/SPEC.md "Host bindings & declared-input
configuration") — on conflict, the issue and spec win, issue first.

As a tanuki operator about to point a target at a different host,
I want a deterministic check that walks scenario prompts, `host_setup`, and
fixture declarations for host-coupled assumptions — literal paths that
resolve in the current host, SHA-pinned pointers, fixture path sets — and
classifies each scenario portable / bound / host-coupled,
So that "will this survive a host change" is a computed answer rendered
where I already look, not a mid-drive surprise.

## Context / decision

Decomposed from #173 (spec lane, ratified 2026-07-19). Render location is
part of the ratified contract: the report surfaces through the existing
read-only surfaces — `history`/`status` and the view catalog per the #168
Surfacing rule — never a new command or door. Implementation note: this
detector and `feature_drift` walk the same charter/prompt/`host_setup`
corpus; reuse the existing corpus walk (the #169 whole-token machinery)
rather than growing a second one.

## Acceptance Criteria

- Given scenarios using only declared bindings, when the check runs, then
  they classify `portable` (or `bound` where bindings exist but only one
  host declares values).
- Given a scenario with a literal path that resolves in the current host, a
  SHA-pinned pointer, or a fixture path set, when the check runs, then it
  classifies `host-coupled` and names each coupled assumption.
- Given `history`/`status`, when a target has classifications to show, then
  the per-scenario classification renders there — read-only, no new
  command, no state written.
- Given a hostless target, when the check runs, then it degrades to silence
  (nothing to couple to), never a crash.

## Out of scope

- Blocking anything — this story only reports; fail-closed enforcement is
  1.33's compat check.
- Regeneration offers for host-coupled scenarios (future; evaluated in the
  umbrella).
