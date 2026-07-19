# Decision draft 2026-07-19 — probe coverage as a second result axis

Referenced by `SPEC.md` §2b ("the rejected stage-graph alternative is
recorded in the 2026-07-19 decision draft") and by the
spec-tanuki-scenario-lifecycle "Charter probe block". Status: **decided
2026-07-19** — operator selected Alternative B (evidence predicates).
Findings: F178 (recurrence 2), F179 (`~/.tanuki/tanuki/ledger.json`).

## Problem (F178)

The manifest's per-scenario `status` is driver-execution status only:
`ok` means the headless process completed, and nothing binds it to the
charter's probe being exercised. Recurrence-2 evidence, retained under
`~/.tanuki/writing-assistant/events/`:

- **post409-c1** (serial control, drive_concurrency acceptance, tools
  9cfc966): `draft-survey` quit after 3 turns (driver invoked the skill
  then "waited" for it); `draft-postmortem-f2` stopped at 16 turns.
  Both manifest `status: ok`.
- **post409-c2** (concurrent treatment): the same scenarios ran 62 and
  68 assistant turns respectively — the intended probe depth. Also
  manifest `status: ok`, indistinguishable from c1.
- Consequence: the c1-vs-c2 makespan comparison (706 s vs 633 s) is
  confounded by unequal work; driver-reach variance was the dominant
  noise source across the whole #409 measurement series
  (pre409-baseline: 2/5 scenarios short-circuited).

A scenario that short-circuits before the stage it exists to test reads
*healthier* (fewer turns, clean exit) than one that spent its budget on
the actual probe, and the scheduler counts its silence as
fix-verification.

## Alternatives evaluated

### A — target-declared stage/probe graph (REJECTED)

The target's scenarios config declares an ordered graph of stages
(`ingest → draft → review → publish`); the drive layer reports how far
along the graph each run got, and coverage is "reached stage ≥ the
charter's declared probe stage".

Why rejected:

- **Breaks target-agnosticism.** Tanuki would have to understand and
  trust target-domain semantics (what a "stage" is, that the graph is
  ordered, that runs traverse it monotonically). Every target invents a
  vocabulary; Tanuki's contracts elsewhere treat target strings as
  opaque.
- **Ordering is a fiction for real targets.** Agent runs skip, repeat,
  and interleave stages; a linear (or even DAG) declaration invites
  wrong "furthest reached" arithmetic and cross-checkpoint
  comparability claims the evidence cannot support.
- **A second declared-vocabulary surface.** A stage graph is exactly the
  observed-vs-declared vocabulary structure that the removed §2
  coverage diff was built on (removed 2026-07-18, spec-tanuki-view D2);
  reintroducing it as a side door recreates the maintenance drift that
  motivated the removal.
- **Mapping burden.** Someone must keep graph ↔ charter ↔ driver
  behavior aligned as the target evolves; drift produces confident but
  false coverage verdicts — worse than `undeclared`.

### B — explicit per-scenario probe-completion predicate (SELECTED)

Each charter MAY declare a `probe` block: one **required** predicate
plus an **unordered named set** of checkpoint predicates, each a
deterministic match (`{on: events|raw, type?, match}`) over the
scenario's own recorded evidence. Evaluation is code in tanuki-drive's
existing normalize/manifest step; the manifest gains an independent
`probe` axis: `reached | short_circuited | undeclared` plus the
matched/unmatched checkpoint split.

Why selected:

- **Target-agnostic.** Checkpoint names are opaque strings; predicates
  are string/regex matches over evidence Tanuki already records. No
  ordering, ranking, or cross-checkpoint comparability is assumed —
  "furthest actually reached" is reported only as the matched/unmatched
  split.
- **Evidence-derived, not self-reported.** The verdict comes from the
  recorded event stream, never from the driver saying it got there. A
  driver-emitted `TANUKI-PROBE` marker variant was considered and
  rejected inside this alternative: the weak driver is the component
  under measurement (F113 precedent).
- **Opt-in and honest about absence.** No probe block → `undeclared`,
  rendered as "coverage not assessable", never as healthy. No
  matrix-wide vocabulary to maintain.
- **Human-gated authoring.** Probes enter and change only through the
  charter plan gate — a probe the human never reviewed is a coverage
  verdict the human never authorized.

## Invariants carried into the ratified text

1. `status` (driver completed) and `probe` (charter's probe exercised)
   are two independent manifest axes, never merged into one verdict.
2. `probe` is independent of scheduler yield and stays out of
   yield/streak/demotion arithmetic in v1 (a short-circuited run may
   still yield findings; a reached run may yield nothing).
3. `undeclared` renders as "coverage not assessable — no probe
   declared" (spec-tanuki-view D3: no silent nothing; never
   healthy-by-default).
4. Views render only mechanically derived fields (spec-tanuki-view D4:
   the surface renders, the tools compute) — dashboard and final
   summary show the `probe` axis next to `status` verbatim from the
   manifest, no model judgment in the loop.
5. §2b is not §2 revived: per-scenario, opt-in, no declared/observed
   vocabulary diff, no charter seeding, no scheduler surface.

## F179 rider (same amendment)

`turns_anomalous` — a deterministic per-scenario bottleneck flag
computed against the scenario's own history, valid only within the
partition key (scenario id + charter revision, effective driver model,
entry-fixture mode/fixture id, target/plugin pin). The comparison
statistic is deliberately undecided; until chosen (a substrate change
gated on this contract landing), the field is absent and absent renders
as "not computed", never "not anomalous".

## Where the contract lives

- `specs/spec-tanuki-trajectory/SPEC.md` §2b — evaluation, manifest
  axes, rendering, boundary note.
- `specs/spec-tanuki-scenario-lifecycle/SPEC.md` "Charter probe block"
  — normative schema and plan-gate authoring rule.
- `docs/tanuki-spec.md` drive step 6 — the drive-phase behavior.
