---
story: 5.13
epic: 5
title: "Story 5.13: copy-pasted helpers across the five tools regain parity, with a guard"
status: published
issue: 342
---

# Story 5.13: copy-pasted helpers across the five tools regain parity, with a guard

Canonical discussion record: [#342](https://github.com/tim-nish/tanuki/issues/342).
On conflict, the issue wins.

The five tools are deliberately standalone zero-dependency scripts
(`tools/tanuki-loop:7` — "Zero dependencies"), so shared helpers exist as
copies by design. The defect is that the copies have silently diverged and
nothing detects it. This story reconciles the diverged copies and adds a
parity guard; it does not introduce a shared module.

## Story

As a maintainer editing one of the five tools, I want the deliberately
duplicated helpers to be asserted identical by the test suite, so that a fix
applied to one copy cannot silently miss the others.

## Acceptance criteria

**AC1 — diverged copies are reconciled.**
Given the current divergences,
When this story lands,
Then each helper family has one agreed body across its members:
- `slug_target`: `tools/tanuki-config:561` (6-line variant, drops the
  `$TANUKI_ROOT` remediation hint) matches the four identical copies at
  `tools/tanuki-drive:1262`, `tools/tanuki-ledger:2305`,
  `tools/tanuki-loop:5978`, `tools/tanuki-scheduler:1394`.
- `open_scenarios`: `tools/tanuki-drive:176` (print + `sys.exit(2)`) agrees
  with `tools/tanuki-loop:412` and `tools/tanuki-scheduler:270` (`die()`)
  on error path and message.
- `load_config`: `tools/tanuki-drive:378`, `tools/tanuki-ledger:180`,
  `tools/tanuki-scheduler:113` share the read-and-merge core (per-tool
  `CONFIG_DEFAULTS` slices may legitimately differ; the merge/precedence
  logic may not).

**AC2 — divergent failure modes are unified deliberately.**
Given `tools/tanuki-ledger:1094` (`current_binding`, returns `None` on an
invalid `host_binding`) and `tools/tanuki-scheduler:166`
(`effective_binding`, `die()`s on the same input),
When this story lands,
Then one failure behavior is chosen and both sites implement it, with the
choice and its reason recorded in the PR description.

**AC3 — mirrored logic is covered by the guard.**
Given `tools/tanuki-ledger:120` (`priority()`, canonical) mirrored by
`tools/tanuki-loop:4851` (`_queue_sort_key`), and `tools/tanuki-drive:943`
(`render_probe`) mirrored by `tools/tanuki-loop:4874` (`_probe_text`),
When either side of a mirrored pair changes without the other,
Then the parity check fails naming the pair.

**AC4 — the guard runs in CI.**
Given the existing suite under `tools/tests/` (run by
`.github/workflows/tests.yml`),
When the parity check is added,
Then it runs as part of that suite, asserts byte- or AST-level equality for
the copy families in AC1 and semantic parity for the AC3 mirrors, and the
zero-dependency property of the five tools is unchanged (no shared import
between them).

## Story questions

- Byte-identical or AST-identical for AC1? Byte-identical is simplest and
  matches the four existing `slug_target` copies; docstring-only divergence
  would then also fail, which may be desired. Implementer's call.
- `cmd_init`/`cmd_status`/`main` also share names across tools but are
  intentionally tool-specific — the guard must key on an explicit list of
  parity families, never on name collision alone.
