---
story: 1.43
epic: 1
title: "Story 1.43: README rework — workflow narrative, command sequence, and a complete linked command index"
status: published
issue: 228
depends_on: [1.42]
---

# Story 1.43: README rework

Issue: https://github.com/tim-nish/tanuki/issues/228 — the canonical
discussion record; on conflict, the issue wins. Depends on story 1.42 (#229,
docs information architecture): the README points into the dedicated docs
pages that story establishes.

As a new Tanuki user,
I want the README to orient me — the Tanuki-commands vs loop-commands
distinction, the basic command sequence, and a complete linked index of every
command,
so that I can determine what to run first and what follows without reading
command source.

## Context (from the issue)

Tanuki has gained many features and commands, but the README no longer
explains them adequately: the Tanuki-vs-loop workflow relies on implicit
knowledge, the basic command sequence is undocumented, one-line descriptions
are too terse to understand, and some newly added commands are absent.

Direction:

- README = orientation + the basic command sequence (including the
  Tanuki-commands vs loop-commands distinction) + a **complete** one-line
  command index with links. Depth moves to dedicated docs pages (defined by
  story 1.42).
- A separate Quickstart file only if the README's orientation section cannot
  stay short.
- **Pointer, not copy:** command definitions are already human-readable
  markdown — the README points into them; where a description is duplicated,
  declare precedence (the command file wins) so copies cannot drift.
- Internal/contract vocabulary must not reach the user surface undefined; the
  public pipeline vocabulary (Event / Finding / Proposal, charter) is fine and
  is introduced where first used.
- The basic-sequence workflow diagram is a Mermaid code block in the README
  (renders natively on GitHub, diffable) — never a committed raster image.
- Completeness and navigability matter more than keeping the README short.

## Acceptance criteria

1. **Given** the reworked README,
   **When** a reader looks for any shipped command,
   **Then** it appears in the command index with a link to its definition or
   doc page — the index is complete (no shipped command is missing).

2. **Given** a new user who has not read command source,
   **When** they read the README's orientation + basic command sequence,
   **Then** they can determine what to run first and what follows, and can tell
   Tanuki commands apart from loop commands.

3. **Given** the basic command sequence,
   **When** it is presented,
   **Then** it is a Mermaid code block in the README (no committed raster
   image).

4. **Given** any command description duplicated between the README and a
   command file,
   **When** the two could diverge,
   **Then** the README declares precedence (the command file wins), so copies
   cannot drift silently.

5. **Given** public pipeline vocabulary used in the README (Event / Finding /
   Proposal, charter),
   **When** a term is first used,
   **Then** it is introduced there; no internal/contract vocabulary reaches the
   user surface undefined.

## Notes

- Structure assessment (single README vs Quickstart split vs dedicated docs
  pages) is shared with story 1.42 and resolves against that story's IA — hence
  the dependency.
