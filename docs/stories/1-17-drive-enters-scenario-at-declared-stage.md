---
story: 1.17
epic: 1
title: "Story 1.17: tanuki-drive materializes the declared entry fixture so the scenario starts at its probed stage"
status: published
depends_on: ["1.16"]
issue: 144
umbrella: 136
---

# Story 1.17: tanuki-drive materializes the declared entry fixture so the scenario starts at its probed stage

Umbrella: https://github.com/tim-nish/tanuki/issues/136 (spec: specs/spec-host-snapshot/SPEC.md "Stage-entry fixtures") — on conflict, the amended spec wins.

As a tanuki operator,
I want `tanuki-drive` to copy a scenario's declared entry fixture into the disposable clone before the driver's first turn,
So that a scenario probing stage N enters at stage N instead of re-running every stage before it, cutting its turns to roughly its probed stage's cost.

**Acceptance Criteria:**

- Given a scenario with a declared `entry_fixture` and a run whose fixture was pinned by init, when the drive prepares the scenario's disposable clone, then the fixture artifacts are present before the driver starts, and `host_setup` / `pollution_check` / touched-path guards behave unchanged.
- Given a scenario without `entry_fixture`, when it drives, then behavior is identical to today.
- Given the manifest, when a stage-entry scenario completes, then the manifest records which fixture it entered from (id + provenance), so findings can be attributed to the entered stage onward.
- Given an attended single run (`/tanuki`), when a stage-entry scenario is driven outside a loop run (no pinned run fixture), then the drive materializes the recipe from the live host clone it already makes — the fixture contract is a loop requirement, not an attended one.
