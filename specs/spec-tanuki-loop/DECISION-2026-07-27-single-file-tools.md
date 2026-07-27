# DECISION 2026-07-27 — single-file tools are the chosen shape

Ruled at triage of issue #347 (hygiene sweep 2026-07-27).

**Decision.** Each tool under `tools/` — including `tanuki-loop` at 6,602
lines / 128 top-level functions as of this ruling — is deliberately one flat
zero-dependency file, invoked by path, with **no shared module between
tools**. The stated constraint this preserves is `tools/tanuki-loop:7`
("Zero dependencies"); the copies it forces are guarded by the cross-tool
parity check (issue #342 / story 5.13), not by consolidation.

**Why recorded.** Size alone kept re-raising the split question. Restructure
is a hypothesis about what the size is made of, and no attribution
measurement supports it yet — so the shape is ratified as intentional rather
than left to be re-litigated by the next size observation.

**Reopen trigger** (named now so a split is not invented under pressure
later): the #342 parity guard failing repeatedly across mirrored pairs
despite reconciliation, or a measured cohesion breakdown (which content
classes the lines are made of, and that they do not belong together) — not a
line count.
