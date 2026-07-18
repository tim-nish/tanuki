---
story: 1.16
epic: 1
title: "Story 1.16: Scenarios declare an entry_fixture; init pins stage artifacts with provenance"
status: published
issue: 143
umbrella: 136
---

# Story 1.16: Scenarios declare an entry_fixture; init pins stage artifacts with provenance

Umbrella: https://github.com/tim-nish/tanuki/issues/136 (spec: specs/spec-host-snapshot/SPEC.md "Stage-entry fixtures") — on conflict, the amended spec wins.

As a tanuki operator dogfooding a multi-stage target,
I want scenarios to declare a named entry fixture (stage-level artifact recipe) that `tanuki-loop init` pins into the run's host fixture with `{source_commit, producing_stage_version}` provenance,
So that a run's stage-entry state is snapshot-isolated, reproducible, and attributable like every other part of the host fixture.

**Acceptance Criteria:**

- Given a scenarios file with a per-scenario `entry_fixture` key naming a recipe (artifact paths relative to the host root + producing stage), when `init` runs on a hosted target, then the declared artifacts are pinned into the run's host fixture and `state.json` records their provenance.
- Given a declared fixture whose artifacts cannot be materialized, when `init` runs, then it fails closed (same rule as an uncloneable host).
- Given scenarios without `entry_fixture`, when `init` runs, then nothing changes from today.
- Given a hostless target with an `entry_fixture` key, when `init` or `doctor` reads the scenarios file, then it reports the key as out of scope rather than silently ignoring it.
- Given the pin-once contract, when iterations run, then all of them see the identical pinned fixture (one fixture per run, never per iteration).
