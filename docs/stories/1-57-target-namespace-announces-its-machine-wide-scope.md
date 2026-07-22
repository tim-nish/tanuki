---
story: 1.57
epic: 1
title: "Story 1.57: Target namespace announces its machine-wide scope at every entry point"
status: published
issue: 289
umbrella: 289
---

Canonical discussion record: https://github.com/tim-nish/tanuki/issues/289
On any conflict between this story and the issue, the issue wins.

Governing invariant (added by the 2026-07-22 owner ruling, triage of #289):
`specs/spec-short-command-surface/SPEC.md` D4, bullet "Shared scope announces
itself" — a resource whose name reads private but whose scope is machine-wide
must be isolated OR warn loudly on collision; silence is not among the
options. This story implements the *warn loudly* half (isolation was
considered and declined: it would reverse the owner decision recorded at
`tools/tests/test-target-slug:12` that TANUKI_ROOT is the supported way to
relocate state).

**As** an operator picking a target slug for a run or a throwaway experiment,
**I want** every surface that resolves that slug to tell me the namespace is
machine-wide and what already lives there,
**so that** two sessions choosing the obvious name collide loudly instead of
quietly overwriting one another.

## Acceptance criteria

- **Given** a target whose `"loop"` block is **complete** (nothing missing),
  **when** `doctor` runs, **then** its scope block is still emitted. Today
  the `scenarios_file` block naming `canonical`, `canonical_file_exists` and
  the machine-wide note is built inside `if missing:`
  (`tools/tanuki-loop:2648`, block at `tools/tanuki-loop:2678-2695`), so the
  operator who already *has* a colliding config — precisely the one at risk —
  never sees it. Hoist it so it does not depend on a missing-config path.
- **Given** an `init` against a slug that already exists, **when** the reuse
  notice prints, **then** it names the namespace path and what already lives
  there (last run and config mtime) in addition to today's event/finding
  counts. Current behavior: `tools/tanuki-ledger:221-233` reports counts and
  a separate `ledger ready: <path>` line; `tools/tanuki-loop:489-510` is the
  loop-side equivalent. Both extend; neither is replaced.
- **Given** two sessions choosing the same slug, **when** the second one
  initializes, **then** a regression fixture asserts it is warned. Today
  `tools/tests/test-ledger-announce:5` asserts the complementary case
  ("writes to an existing target stay silent") and nothing covers the
  second-session warning — add the sibling fixture rather than changing that
  assertion, which remains correct for plain writes.
- **Given** any of the above, **then** no target-resolution semantics change:
  slugs resolve exactly as today and nothing is auto-suffixed or relocated —
  this story changes what is *said*, never what is *resolved*.

## Notes

The issue names the conformant pattern to imitate: `distill --check`'s notice
that a machine-wide `contribute_back` block is ignored, which states the key,
the reason, the deciding issue, and the correct location. Match that shape —
name the thing, why it matters, and where to go — rather than a bare warning.
