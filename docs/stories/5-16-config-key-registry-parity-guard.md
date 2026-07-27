---
story: 5.16
epic: 5
title: "Story 5.16: the config-key registry gains a parity guard and the template catches up"
status: published
issue: 340
umbrella: 340
---

# Story 5.16: the config-key registry gains a parity guard and the template catches up

Canonical discussion record: [#340](https://github.com/tim-nish/tanuki/issues/340).
On conflict, the issue wins.

Ruling (2026-07-27 triage, alternative A): the table in
`docs/tanuki-spec.md` is the canonical key registry (declared in the table's
new preamble, amended this sitting; `specs/spec-tanuki-scenario-lifecycle/SPEC.md:241`
already assumed it). The three missing rows (`drive_concurrency`,
`driven_env_passthrough`, `reach_min_turns`) were backfilled at triage. This
story adds the mechanical guard and the template catch-up. Withdrawn from
the issue's evidence: `drive_model` vs `driver_model` is deliberate two-key
precedence (`tools/tanuki-drive:35-37,391-393`), not drift — no rename.

## Story

As a developer adding a config key in a tool, I want a check that fails
until the key has a registry row, so that the canonical table cannot lag the
code again.

## Acceptance criteria

**AC1 — code keys are a subset of registry rows.**
Given the per-tool defaults (`tools/tanuki-drive:80-96`,
`tools/tanuki-ledger:180`, `tools/tanuki-scheduler:113`) and the
`tanuki-config` CATALOG (`tools/tanuki-config:158-175`),
When the parity check runs (under `tools/tests/`, wired into
`.github/workflows/tests.yml`),
Then every key present in any tool's `CONFIG_DEFAULTS` or in CATALOG has a
row in the `docs/tanuki-spec.md` table, and a missing row fails the check
naming the key.

**AC2 — the template is a subset of the registry.**
Given `templates/config.example.json`,
Then every key it carries has a registry row, and it gains the keys ruled
missing at triage where they make sense as examples (at minimum
`driven_env_passthrough`; judgment on the rest recorded in the PR).

**AC3 — scenario-file loop-block keys are out of scope, stated.**
Given the `"loop"` block namespace (e.g. `drive_model`,
`tools/tanuki-drive:35-37`) is a different file's contract,
Then the parity check does not assert it against this table, and the check's
docstring says so.

## Story questions

- Should the check also assert registry rows point at real code keys (the
  reverse direction), so a removed key can't leave a ghost row? Cheap if the
  key list is already computed — implementer's call, note the choice.
