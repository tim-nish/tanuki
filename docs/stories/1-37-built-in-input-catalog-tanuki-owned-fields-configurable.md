---
story: 1.37
epic: 1
title: "Story 1.37: Built-in input catalog — Tanuki-owned generic fields configurable without a hand-authored inputs block"
status: published
issue: 208
umbrella: 208
---

# Story 1.37: Built-in input catalog — Tanuki-owned generic fields configurable without a hand-authored inputs block

Umbrella: https://github.com/tim-nish/tanuki/issues/208 (spec:
specs/spec-tanuki-scenario-lifecycle/SPEC.md "Host bindings & declared-input
configuration" — built-in input catalog bullet + table) — on conflict, the
spec wins.

As a tanuki operator onboarding a new dogfood target,
I want `/tanuki <target> configure` to offer Tanuki's own generic settings
(`drive_model`, `drive_concurrency`) out of the box,
So that routine configuration never requires hand-authoring an `inputs`
block in `~/.tanuki/scenarios/<target>.scenarios.json` before the surface
becomes useful.

## Context / decision

Decomposed from #208 (spec lane, ratified 2026-07-19). The ratified
declared-input contract (#172) made per-target declaration the *only*
editable path, which was a scoping error for Tanuki-owned keys: `drive_model`
and `drive_concurrency` bind Tanuki's own `loop.*` settings and need zero
target knowledge to validate. The fix is a **built-in input catalog** shipped
by `tanuki-config`; target declarations shadow, and may explicitly suppress,
catalog entries. The catalog supplies only the *declaration* — values still
persist into the target's scenarios file at the `binds` path; the backing
JSON stays the source of truth and hand-editing remains legal. v1 catalog is
exactly the spec table's two fields; growing it means amending the spec
first (the admission rule). Non-goals: per-run override semantics, doctor
hooks, host bindings, #173 portability — all unchanged.

## Acceptance Criteria

- Given a fresh target whose scenarios file has **no** `inputs` block, when
  `tanuki-config --target <t> show` runs, then both catalog fields render as
  editable with source `[built-in input]`, and the "no inputs declared"
  one-liner no longer claims only hand-editing is available.
- Given the same fresh target, when the full configure flow runs
  (`check` → `set --dry-run` → `set`), then typed validation, the dry-run
  preview, and the persist all work end-to-end and the value lands in that
  target's scenarios file at the catalog entry's `binds` path.
- Given a target whose `inputs` block declares a field with the same name as
  a catalog entry (e.g. writing-assistant's narrowed `drive_model` enum),
  when `show`/`check`/`set` run, then the target declaration wins wholesale
  (test: a target-narrowed enum rejects a value the catalog would accept),
  and writing-assistant's observable behavior is unchanged.
- Given a target that explicitly suppresses a catalog field, when `show`
  runs, then the field renders read-only with the suppression stated, and
  `set` refuses it.
- Given any field that is neither declared nor in the catalog, when `show`
  runs, then it remains read-only exactly as today.
- Given the implementation diff, then the catalog contains zero
  target-specific knowledge (the #409 audit posture), and the spec's catalog
  table, the command doc's `configure` section, and init step 5 (mention
  that `configure` works out of the box) are amended in the same change.
- Given the tools test suite, then new fixtures cover: catalog-only target,
  shadowed field, suppressed field, and the non-catalog read-only case.
