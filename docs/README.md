# Tanuki documentation

This page is the map of Tanuki's documentation: it tells you **which
documents are authoritative today (normative) and which are historical
record (non-normative)**, and where each kind of material lives. If you are
unsure whether a document still governs behavior, start here.

Material under this repository is separated into four kinds.

## 1. Current user documentation — *normative*

What a user reads to run Tanuki today.

- [`README.md`](../README.md) — orientation, the basic command sequence, and
  the complete command index. Start here.
- Command definitions (also the source of truth for each command's behavior):
  - [`commands/tanuki.md`](../commands/tanuki.md) — the `/tanuki` pipeline
    (drive → mine → consolidate → decide), views, `init`, and `ingest`.
  - [`commands/tanuki-loop.md`](../commands/tanuki-loop.md) — the unattended
    `/tanuki-loop` overnight loop, the morning gate, and reconcile.

A current user-documentation page must be **self-contained**: understanding it
never requires reading a development-history story. Where a command
description is duplicated into the README, the **command file wins** — the
README points into it rather than restating it.

## 2. Specifications — *normative*

The current contracts. These define behavior other components rely on; change
them deliberately.

- [`specs/`](../specs/) — one directory per contract
  (`specs/spec-*/SPEC.md`), each the current governing text for its area
  (e.g. `spec-tanuki-loop`, `spec-host-snapshot`, `spec-tanuki-view`).
- [`docs/tanuki-spec.md`](tanuki-spec.md) — the overall prototype spec:
  standing constraints, the three-role model, and the public vocabulary
  (Event / Finding / Proposal). Read this for the whole-system contract; read
  a `specs/spec-*/SPEC.md` for a specific area.

## 3. Development history — *non-normative*

Completed BMAD stories, kept as a record and never deleted. Historical, not a
description of current behavior.

- [`docs/stories/`](stories/) — the story registry. See
  [`docs/stories/README.md`](stories/README.md) for how to read it and why it
  is non-normative. A story tells you *how a change was planned and landed*,
  not *what the system contracts are now* — for the latter, follow the story's
  pointers into the specs above.

## 4. Archived planning records — *non-normative*

Superseded plans, clearly marked as no longer governing.

- Planning artifacts produced by the BMAD tooling live under `_bmad-output/`
  and are **local and untracked** (git-ignored) — they are working scratch,
  not part of the published surface.
- Superseded material that remains in the tree is struck through with a
  pointer to what replaced it, so provenance is preserved without implying the
  old plan still holds.

---

### How to tell normative from historical at a glance

- **Kinds 1 and 2** (this page's first two sections) are normative — the
  README, the command files, and the specs.
- **Kinds 3 and 4** are non-normative — anything under `docs/stories/` or
  `_bmad-output/`, and anything struck through. Each non-normative area
  carries its own marker (see `docs/stories/README.md`) so the boundary is
  visible from inside the directory, not only from this index.
