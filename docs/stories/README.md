# Development history — completed stories (non-normative)

**This directory is historical record, not current documentation.** It holds
the BMAD stories that planned and delivered Tanuki's changes. A story
describes *how a change was scoped and landed at the time*; it does **not**
describe what the system's contracts are now. Nothing here is deleted —
completed work is kept for provenance.

For what governs behavior today, see the normative material mapped in
[`../README.md`](../README.md):

- **Specifications** (the current contracts): [`../../specs/`](../../specs/)
  and [`../tanuki-spec.md`](../tanuki-spec.md).
- **Current user documentation**: the root [`README.md`](../../README.md) and
  the command files under [`../../commands/`](../../commands/).

When a story is still load-bearing, its content is **promoted** into a current
spec or doc page (with a back-pointer here); superseded stories are struck
through with a pointer to what replaced them. Until a story has been promoted
or struck, treat it as history: read it for context, follow its pointers into
the specs for the authoritative contract.

> Note: this directory is also the working registry that `story-sync` reads
> and writes (`.claude/story-sync.json` → `stories_dir`). That makes it both
> the active surface and the append-only archive; it is marked non-normative
> here so the public documentation surface stays clear without moving files
> the story workflow depends on.
