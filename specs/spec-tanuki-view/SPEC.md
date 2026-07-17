# Spec: /tanuki view — one human-facing surface for every read-only view

Status: IMPLEMENTED 2026-07-17 (D1/D2 landed in `commands/tanuki.md` and
`docs/tanuki-spec.md` on the operator's order; D3 had already landed ahead of
them — `EMPTY_STATES` in `tools/tanuki-loop` with
`tools/tests/test-loop-empty-states` — and D4 stands as the acceptance rule.
**OPEN-2 RESOLVED 2026-07-17 by operator ruling**: view vocabulary is
target-local in v1 — see OPEN-2. Drafted PROPOSED 2026-07-17 as a **new design
exercise**, grounded in the current repository — not a transcription of a
previously ratified discussion; **D3 AMENDED and OPEN-1 RESOLVED
2026-07-17** — the empty state is a closed enumeration, typed and tested (see
D3), and D3 landed ahead of D1/D2 with issue #66 as its first instance.
Touches `commands/tanuki.md`, `docs/tanuki-spec.md`, and — per OPEN-1's
resolution — `tools/tanuki-loop` (the dashboard's empty states gain typed
`{state, expected, reason, next}` alongside the prose they already print; an
additive change, no subcommand/flag/existing-field altered). `tools/tanuki-
scheduler` and `tools/tanuki-ledger` are unchanged: the tools stay the
deterministic machine substrate and this spec adds a presentation surface
over them. No change to the pipeline contract (events → findings → proposals
→ labeled issues) or to any isolation/safety property. OPEN-1, OPEN-2, and
OPEN-3 are resolved in place with the reasoning kept.

## Problem

Every read-only view Tanuki offers already exists and is already
deterministic — but they are scattered across three tools and three
unrelated entry shapes, so *which* command shows a given view is knowledge
the operator carries in their head:

| View | Substrate today | Human entry today |
|---|---|---|
| ledger status + derived `next` | `tanuki-ledger status` / `next` | `/tanuki <t> --status` |
| cross-run scenario history | `tanuki-scheduler history` | `/tanuki <t> --history [scenario]` |
| axis coverage + exploration debt | `tanuki-scheduler history --scenarios <file>` | `/tanuki <t> --history` (only when the matrix declares `axes`/`covers`) |
| per-run trajectory | `tanuki-scheduler history --scenario <id> --trajectory` | `/tanuki <t> --history <scenario>` |
| live loop dashboard | `tanuki-loop dashboard` | `/tanuki-loop` only |

Three defects follow from the shape, not from any one view:

1. **The catalog is unenumerated.** Nothing tells an operator what views
   exist. `dashboard` lives behind a different command entirely; coverage
   appears inside `--history` only when the matrix happens to declare axes,
   so its absence is indistinguishable from "no coverage problem".
2. **Discovery is by memorized flag** — precisely what
   `specs/spec-short-command-surface/SPEC.md`'s Principle forbids ("every
   required input is resolved by configuration, state, or an interactive
   question — never by memorized flags"). `--history <scenario>` silently
   switching to a trajectory render is a flag overload an operator must be
   told about.
3. **View selection is a typed guess.** The target picker
   (`commands/tanuki.md`, target resolution) already proves the pattern —
   "selection beats typing names with no completion" — but it resolves the
   *target* only. Having resolved it, the operator still types a flag.

This spec adds no view. It gives the existing views one door, one catalog,
and one picker.

## Principle (the one-line contract)

**Every read-only view is reachable from one option-free command; the tools
that compute the views remain the fully-optioned machine substrate.**
`/tanuki [target] view` is the human door. `tools/tanuki-*` keep their exact
flags, their `--json`, and their status as the contract for scripts, tests,
and headless runs. The view surface **reads and renders; it never computes**
what a tool could compute, and it never writes.

## Deliverables (ranked)

### D1 — `/tanuki [target] view` — the option-free entry

`view` is a bare word, precedented by `/tanuki [target] init` (an argument
grammar this repo already uses; not a new convention).

- `/tanuki <t> view` — resolve the target the way `/tanuki` already resolves
  it (explicit arg → `tanuki-scheduler resolve --cwd $PWD` → single
  configured target → the existing target picker; unchanged, not restated),
  then present the **view picker**: every view in the D2 catalog as a
  selectable option via the interactive question interface, each with a
  one-line description and a state-derived hint of whether it currently has
  anything to say (e.g. `coverage — 3 axis values uncovered`). Selection
  beats typing.
- `/tanuki <t> view <name>` — jump straight to a named view, for the
  operator who knows what they want. Names come from the D2 catalog and
  nowhere else.
- Every view is **read-only** and safe to run at any time, including
  alongside an unattended loop (the dashboard's existing property, inherited
  by the surface: it reads state files only).

Acceptance: an operator who knows only `/tanuki <t> view` can reach every
view this repo offers, without knowing that a scheduler, a ledger, and a
loop tool exist.

### D2 — the view catalog is a closed enumeration

The surface lists exactly these views; each names its substrate command, and
adding a view means amending this list (an unenumerated view is the defect
D1 exists to fix):

- **`status`** — ledger counts + the derived `next` step.
  Substrate: `tanuki-ledger status` / `next` (short-command-surface D2's
  enumerated-state derivation; not recomputed here).
- **`live`** — the loop dashboard: health, latest drive, this run, scheduler
  decisions, convergence, why-stopped/NEXT.
  Substrate: `tanuki-loop dashboard` (`--follow` remains available).
  This is the **live** view; `history` is the long view.
- **`history`** — cross-run per-scenario execution history: state,
  executions, streaks, recurrence, unexplored, long-unrun, recently
  productive, selection history with reasons.
  Substrate: `tanuki-scheduler history`.
- **`coverage`** — declared exploration space vs execution record: per axis
  value explored / authored / uncovered, axis rollup, exploration debt, and
  the ≤3 advisory recommendations.
  Substrate: `tanuki-scheduler history --scenarios <file>`
  (`specs/spec-tanuki-scenario-lifecycle/SPEC.md`, Exploration axes).
  Promoted to a **named view of its own** rather than a block that appears
  inside `history` only when axes happen to be declared — an absent
  declaration is a state to report, not a reason to render nothing.
- **`trajectory`** — one run's step-by-step path from its event files.
  Substrate: `tanuki-scheduler history --scenario <id> --trajectory
  [--run <id>]` (`specs/spec-tanuki-trajectory/SPEC.md`). Promoted out of
  the `--history <scenario>` flag overload into a named view.

### D3 — a view never renders a silent nothing

Inherited from the loop dashboard's **fixed skeleton** rule
(`commands/tanuki-loop.md`, Monitoring; `specs/spec-tanuki-loop/SPEC.md`):
a view whose substrate is missing degrades to an explicit one-line reason,
never a silent omission and never an empty screen. Applied to the surface as
a whole:

- Every view in the catalog is always *offered*; a view with no substrate
  states why in one line at the picker and in the view itself.
- The reason must carry an **expected-vs-gap signal** — whether emptiness is
  normal for this target's state (a fresh target has no runs) or a real gap
  the operator should close.

**The empty state is a closed enumeration, not prose** (OPEN-1 resolved
2026-07-17). Every section that can render empty resolves to exactly one
member of an enumerated set, emitted as typed data and rendered as the
one-line reason:

```
{state: <enum member>, expected: true|false, reason: "<one line>",
 next: "<the command that changes it>" | null}
```

- `expected: true` — emptiness is normal for this target's state (a fresh
  target has never synced; no run has driven yet). `expected: false` — a
  **gap**: the substrate should exist and does not.
- **One test fixture per enumerated state**, so a new state or a regression
  fails visibly rather than being re-reported by a later dogfood run.
- Any command the reason names must be explained where it is named — an
  operator meeting `tanuki-scheduler sync` for the first time in an error
  string learns nothing from the name alone (F47).

This is `specs/spec-short-command-surface/SPEC.md` D2's rule applied to the
same disease. D2 exists because the `next:` hint "has been fixed twice and
regressed twice because it is patched string logic"; the empty-state class
has now been reported three times — **F5** (sections omitted with no note),
**F47 / issue #66** (scheduler-decisions naming an unmet command with no
expected-vs-gap signal), and **F46 / issue #65** (latest drive reading
"none yet" after an iteration had run). Prose no test can check is what
regressed each time. Issue #66 is the first instance of this rule, not a
one-off fix.

### D4 — the surface renders; the tools compute

A standing acceptance rule, not a one-time change:

- Every number, verdict, and ranking a view shows comes from a substrate
  command's output (`--json` where a machine shape exists). The command
  layer composes, labels, and orders; it never re-derives.
- No view logic migrates *into* the tools to serve presentation, and no
  computation migrates *out* of them into the command layer. A view needing
  a number no tool emits is a substrate change, specified separately and
  landed in the tool with its own tests — not computed in prose.
- Consequently: this spec's implementation touches `commands/tanuki.md` and
  `docs/tanuki-spec.md`. **Refactoring `tools/tanuki-scheduler` is out of
  scope** (explicit operator instruction, 2026-07-17).

## OPEN questions (not decided in this spec)

**~~OPEN-1 — unexplained coverage.~~ RESOLVED 2026-07-17 — closed
enumeration, typed and tested.** Coverage stated *what* (`explored` /
`authored` / `uncovered`) but never *why*, and an empty view could not
distinguish "expected for a fresh target" from "a real gap". The answer is in
**D3 above**: every empty state is one member of a closed enumeration
carrying `expected: true|false`, emitted as typed data, with one test fixture
per state.

Two parts of the question as originally posed were wrong, and the record
should say so:

- It asked whether the **substrate** emits the code or the **command layer**
  derives it. That framing does not survive contact with the code: the
  dashboard **is** `tanuki-loop dashboard` — a tool that prints prose — so
  #66's fix is a substrate change either way. The real question was never
  *where the logic lives* but whether the reason is **enumerated typed data
  or prose**. Prose is what regressed, three times.
- It treated "is this a closed enumeration like D2's ledger states?" as open.
  It is not a close call: D2 exists because the same class of string logic
  "has been fixed twice and regressed twice", and the empty-state class is
  now on its third report (F5, F47/#66, F46/#65). The evidence had already
  decided it.

**Issue #66 is the first instance of the D3 rule**, not a one-off — its
`direct` classification stands, but it lands as an application of the
enumeration rather than a prose patch, which is why the two could not be
resolved independently.

**~~OPEN-2 — axis vocabulary.~~ RESOLVED 2026-07-17 (operator ruling).**
**View vocabulary is target-local in v1.** A view renders the selected
target's axis names exactly as emitted by its substrate. Tanuki does not
normalize axes and does not claim comparability across targets.
Cross-target comparison is out of scope and requires a separate future
specification. Applied to the questions as originally posed:

- *Is cross-target comparison a goal at all?* No — out of scope in v1;
  pursuing it is a new spec, not an amendment here.
- *May the surface display an axis differently from its declared key?* No —
  "exactly as emitted" settles presentation too: no titlecasing, no
  substitutions, no note-derived retitling. The lifecycle spec's no-rename
  rule and this ruling now cover stored state and presentation alike.
- *A recommended starter vocabulary?* None is owned or enforced by this
  spec; nothing in v1 normalizes toward one.
- *The residue of OPEN-1's partial answer* — whether the axis vocabulary
  that empty states are reported in is comparable across targets — is
  answered by the same ruling: it is not, and no view may imply it is.

The original questions, kept for the record:

- With no shared vocabulary, two targets may name the same concept
  differently (`error_path` vs `failure_mode`) and a `coverage` view is
  unreadable across targets. Is cross-target comparison a goal at all?
- May the surface **display** an axis differently from its declared key
  (titlecasing, a `note` as a subtitle) without "renaming" it in the sense
  the lifecycle spec forbids — i.e. is that rule about *stored state* or
  about *presentation*?
- Is there a recommended-but-not-enforced starter vocabulary (the lifecycle
  spec's examples — `framework`, `article_intent`, `host_state`,
  `error_path` — read like one already), and if so, who owns it: this spec,
  the lifecycle spec, or neither?
- Should `coverage` render an axis whose values are all `uncovered` and
  whose scenarios are all unexecuted — a declared-but-untouched space —
  differently from an undeclared one? **Partly answered by OPEN-1's
  resolution:** both are empty states, so both must be enumerated members
  carrying `expected` — "declared but never run" is expected on a fresh
  target and a gap on a mature one. What remains open here is only whether
  the *axis vocabulary* they are reported in is comparable across targets.

**~~OPEN-3 — flags vs. the catalog.~~ RESOLVED 2026-07-17 (issue #74).** The
question — whether `view` is a word or a flag, and whether `--status` /
`--history` survive — is settled surface-wide rather than by this spec:
`specs/spec-short-command-surface/SPEC.md` **D6** ratifies the command-shape
rule ("a bare word selects a mode that does not drive; flags modify a drive")
and applies it to every command in one change. Consequences for this spec:
`view` is a bare word (D1 as drafted, now on a stated rule rather than the
`init` precedent); `--status` and `--history` survive as aliases; and
`--history <scenario>`'s overload into a trajectory render is named properly
for the first time as `view trajectory`. D6 ratifies this surface's *shape*
only — the catalog (D2) and everything else here remain PROPOSED.

## Non-goals (constraints folded in, so implementation needs no attachments)

- **No scheduler refactor.** `tools/tanuki-scheduler` keeps its subcommands,
  flags, and output contract exactly. This spec proposes a surface over the
  substrate, never a reshaping of it (explicit operator instruction).
  **Amended 2026-07-17 (OPEN-1):** D3's empty-state enumeration is the one
  permitted substrate change, and it is additive — a section that can render
  empty gains a typed `{state, expected, reason, next}` alongside the prose
  it already prints. No subcommand, flag, or existing output field changes.
  The original non-goal assumed the explanation could live in the command
  layer; it cannot, because the dashboard that produces these empty states is
  itself a tool. Honoring the non-goal literally would have meant two
  descriptions of one truth — the tool's prose and the view's — which is the
  drift D4 forbids.
- **No new views.** The catalog is the views this repo already computes. A
  new view is a new spec.
- **No writes, no gates, no dispositions.** Views are read-only. The
  decision pass (`/tanuki <t> decide`, short-command-surface D1) is where
  state changes; a view may *point at* it as the next step, and never
  performs it. A view never modifies the scenario matrix — coverage
  recommendations stay advisory, entering only through the plan-gated
  generation pass (inherited, unchanged).
- **No new prompts in headless mode.** The picker is a convenience of
  attended commands only; `tanuki-loop`'s unattended phases keep their
  explicit-flags/config contract and never acquire a view dependency.
- **No downstream awareness.** The pipeline still ends at the labeled issue;
  no view invokes, names, or configures downstream tooling.
- **No cross-target vocabulary (v1).** View vocabulary is target-local: a
  view renders the selected target's axis names exactly as its substrate
  emits them — no normalization, no comparability claim across targets.
  Cross-target comparison is a separate future specification (OPEN-2's
  resolution).
- **No second vocabulary.** Views render the existing status set, the
  existing coverage states, and the declared axes. No view introduces a term
  its substrate rejects (the F99/F25 class — a view teaching a vocabulary a
  sibling command does not accept).

## Ordering

D1 and D2 land together (a door with no catalog is not a surface). **D3 is
unblocked** (OPEN-1 resolved 2026-07-17) and lands *ahead of* D1/D2: its
enumeration applies to the dashboard's empty states, which exist today and
have produced every report of this class, and issue #66 is its first
instance. D4 is a standing acceptance rule rather than a one-time change.
**OPEN-2** (resolved: target-local vocabulary) blocks nothing in D1/D2 —
`coverage` renders declared axes verbatim, which is now the ruled behavior,
not an interim one; any cross-target coverage comparison starts as a new
spec.

Sequencing against current work: originally **PROPOSED and unscheduled**,
following the in-flight sequence (commit the #72/#73 spec changes → Story
1.6 → Story 1.7) with no story created in that sitting. **Superseded
2026-07-17:** the operator ordered implementation directly; D1/D2 landed as
the "Views" section of `commands/tanuki.md`. The `live` view's substrate
description should still be re-checked against Story 1.7's outcome, which
changes what the decision pass is called.
