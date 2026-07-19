---
story: 1.27
epic: 1
title: "Story 1.27: Declared inputs block with typed generic validation and deterministic show/set/check operations"
status: published
issue: 180
umbrella: 172
---

# Story 1.27: Declared inputs block with typed generic validation and deterministic show/set/check operations

Umbrella: https://github.com/tim-nish/tanuki/issues/172 (spec:
specs/spec-tanuki-scenario-lifecycle/SPEC.md "Host bindings & declared-input
configuration") — on conflict, the issue and spec win, issue first.

As a tanuki operator who needs to change routine target configuration,
I want the scenarios file to carry a target-authored `inputs` block — each
entry naming a field, a generic type (`repo-path`, `path`, `string`,
`enum`), a description, a scope (`per-target` | `per-run`), and where it
binds — with deterministic tool operations that show the effective
configuration, set a declared field, and check a value by type,
So that configuration is machine-validated at the point of change instead of
discovered broken at the next drive.

## Context / decision

Decomposed from #172 (spec lane, ratified 2026-07-19). Tanuki provides the
workflow; targets declare the fields — no target semantics inside Tanuki
(the story-1.17 ownership rule). Only declared fields are editable;
undeclared fields render read-only. The effective chain is: built-in
defaults < `~/.tanuki/config.json` < file `defaults` < persisted per-run
override < CLI flags.

## Acceptance Criteria

- Given a scenarios file with a well-formed `inputs` block, when the config
  loads, then each declared field is available with its type, scope,
  description, and binding point; a malformed block fails closed naming the
  defect.
- Given a `show` operation, when run, then it renders the effective
  configuration resolved through the full chain, labels each value's source,
  and marks undeclared fields read-only.
- Given a `set` operation on a declared field, when the value passes typed
  validation (path exists; repo-path is a git toplevel; enum membership),
  then the write is atomic (temp + rename) to the backing JSON; when
  validation fails, nothing is written and the failure names the type rule.
- Given a `set` on an undeclared field, when attempted, then it is refused —
  only declared fields are editable.
- Given a hand-edit made directly to the JSON, when `show` runs afterward,
  then it renders the edited state correctly (the file stays the source of
  truth).
- Given any operation, when it validates, then it checks type only — no
  target-semantic interpretation anywhere in Tanuki.

## Out of scope

- The picker action, doctor hook, and review-before-persist flow (1.28).
- Per-run override lifecycle and expiry (1.29).
- Host bindings and portability (1.30–1.33).
