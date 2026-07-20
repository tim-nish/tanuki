---
story: 1.42
epic: 1
title: "Story 1.42: Docs information architecture — separate current user docs, specs, development history, and archived planning"
status: published
issue: 229
---

# Story 1.42: Docs information architecture

Issue: https://github.com/tim-nish/tanuki/issues/229 — the canonical
discussion record; on conflict, the issue wins. Companion to #228 (README
rework, story 1.43, which depends on the structure this story establishes).

As a Tanuki contributor,
I want the material under `docs/` separated by kind — current user docs,
specifications, development history, and archived planning — with normative
content clearly distinguished from historical record,
so that I can tell at a glance which documents are authoritative today and
never have to read a story to understand a current doc.

## Context (from the issue)

`docs/` has become a catch-all and `docs/story` has grown substantially. The
layout should be reviewed against what an official open-source project needs.
The four kinds now mixed under `docs/`:

1. **Current user documentation** — what a user reads today.
2. **Specifications** — the current contracts (`docs/spec` or `specs/`).
3. **Development history** — completed stories; archive, never delete.
4. **Archived planning records** — superseded plans, clearly non-normative.

Old stories are compressed by **promotion**: anything still load-bearing
moves into a current spec or doc page (with a back-pointer); superseded
material is struck with a pointer to what replaced it. Where a directory
serves as both the current surface and the append-only archive, split it —
an archive area (e.g. `docs/archive/` or `docs/history/`) marked
non-normative keeps the public surface clean without losing provenance.

The concrete IA proposal is part of this story's execution (the issue filed
it for tracking; "do not implement yet" applied to filing time).

## Acceptance criteria

1. **Given** the reorganized `docs/` tree,
   **When** a contributor browses it,
   **Then** they can tell at a glance which documents are normative today and
   which are historical record (a clearly marked, non-normative archive/history
   area holds superseded planning and completed-story material).

2. **Given** any current (normative) doc page,
   **When** a reader reads it,
   **Then** it is self-contained — understanding it never requires reading a
   story.

3. **Given** the old stories under `docs/story`,
   **When** the reorganization runs,
   **Then** each is either promoted (load-bearing content moved into a current
   spec/doc with a back-pointer) or archived with a struck-through pointer to
   what replaced it — nothing is deleted, and provenance is preserved.

4. **Given** tooling and command files that reference spec/doc paths (e.g.
   `specs/spec-*/SPEC.md`, `${CLAUDE_PLUGIN_ROOT}/…`),
   **When** files move,
   **Then** every such reference is updated so no pointer is left dangling.

## Notes

- The concrete IA (target directory names, what moves where) is decided during
  execution; capture the proposal before mass file moves so the diff is
  reviewable.
- Coordinate with story 1.43 (#228): the README points into the docs pages this
  story establishes, so this story's structure lands first.
