# Spec: Tanuki trajectory — typed path events, coverage diff, and the trajectory view

Status: **§2 (coverage diff) REMOVED 2026-07-18** with the whole coverage
function (operator decision — see spec-tanuki-view D2 and the lifecycle
spec's matching removal note); §1 (typed events) and §3 (trajectory view)
stand unchanged and remain in force. **AMENDED 2026-07-19 (findings
F178/F179, operator selection of the evidence-predicate alternative): §2b
probe coverage** — a per-scenario, opt-in second result axis, deliberately
NOT a revival of §2 (see §2b's boundary note). RATIFIED 2026-07-16 (operator, after
the 2026-07-16 consistency
review amended build-order labels, the mixed pre/post-marker coverage note,
and the recovery/evidence acceptance criteria). The *concept* was already
ratified operator policy (2026-07-15 rulings on the operator's policy
surface); this spec is its executable design. Extends `docs/tanuki-spec.md`
(the Event vocabulary — mechanical enrichment only) and
`specs/spec-tanuki-scenario-lifecycle/SPEC.md` (consumes its `axes`/`covers`
declarations and lands on its `history` surface). It amends no existing
contract: the pipeline nouns (Event → Finding → Proposal) and the one-way
flow are unchanged.

Problem: per-run choice data is captured (Events record what happened; the
raw stream is persisted per scenario), but nothing renders the simulated
user's **actual path** — Q1→selected A, Q2→selected B, error→recovery→outcome.
`tanuki-scheduler history` shows decision-point pins, yield, and findings,
never the step-by-step trajectory. Worse, the decision points a run *actually
hit* are never compared with the ones the matrix *declares* (`axes`/`covers`),
so undeclared decision points — the true exploration blind spots — surface
only when a human reads raw logs.

## 0. Concept constraints (ratified — the design must not drift from these)

- **Trajectory is a view, never a fourth pipeline noun.** It is a
  deterministic rendering over the persisted per-scenario event/raw stream.
  It is never stored whole (not in the ledger, not as a new artifact class),
  never judged, and never becomes a reward/search structure — adaptive
  scheduling by value-backup over trajectories (MCTS and kin) is rejected
  policy: exploitation starves coverage, and consumable/deduped reward breaks
  the stationarity such search assumes.
- **Capture never judges.** The capture-side guarantee is *mechanical* Event
  enrichment: typed markers lifted verbatim from the raw stream by string
  match. Any judgment (is this friction? should this become a charter?)
  stays downstream at extraction / the human gate.
- **The ledger growth policy holds.** No stage loads the ledger whole; the
  renderer reads per-run event files and prints — it adds nothing to
  `ledger.json`.
- **Coverage-diff output is advisory and plan-gated.** Undeclared decision
  points surface as *charter seeds*; a seed becomes a charter only through
  the plan gate. Never auto-charter, never an automatic spec/matrix edit.

## 1. Capture guarantee — typed trajectory events (prerequisite)

The existing Event `type` vocabulary (`docs/tanuki-spec.md`) is
`{tool_error, permission_block, retry, user_choice, skill_start, skill_end,
result, budget, note}`. This spec adds **two** members and gives one existing
member structured fields — no new choice type is introduced; `user_choice`
already names the concept and is reused:

| type          | status   | meaning                                             | structured fields |
|---------------|----------|-----------------------------------------------------|-------------------|
| `user_choice` | existing | the simulated user selected an option at a decision point | gains `point` (decision-point name), `selected`, `alternatives?` (list) — optional, marker-lifted; a `user_choice` without them is a legacy event |
| `recovery`    | **new**  | a successful continuation bound to a prior error    | `of_seq` (the `seq` of the `tool_error` it recovers) |
| `outcome`     | **new**  | the terminal disposition of the scenario            | `disposition` (`success` \| `gave_up` \| `partial`), free-text `detail` |

Mechanics (all inside `tanuki-drive`'s existing normalize pass — no new tool,
no LLM in the loop):

1. **Marker grammar.** The drive-injected simulated-user preamble (drive's
   own boilerplate, never per-scenario prose) instructs the driver to emit
   one line per decision:
   `TANUKI-CHOICE point=<name> selected=<value> [alternatives=<v1|v2|…>]`
   and one terminal line:
   `TANUKI-OUTCOME disposition=<success|gave_up|partial> <free text>`.
2. **Mechanical lift.** Normalize scans `<scenario>.raw.jsonl` for the marker
   grammar and lifts each match verbatim into a typed Event
   (`{run, scenario, seq, type, detail, …structured fields}`) in
   `<scenario>.events.jsonl`. String match only; malformed markers are
   ignored with a count in the manifest (`trajectory_markers_dropped`).
3. **Recovery binding.** Normalize emits a `recovery` event when a
   `tool_error` is followed (same scenario) by a successful step that the
   raw stream ties to the same operation — the deterministic rule: the next
   marker or tool success whose detail names the same command/subcommand.
   No inference beyond that rule; unmatched errors simply have no recovery
   event.
4. **Guarantee.** Every completed scenario run yields ≥1 `outcome` event
   (normalize synthesizes `disposition=partial` from the driver `result`
   event when the driver emitted no marker — flagged `synthesized: true`).
   Existing runs predate the grammar; nothing is backfilled.

The existing exact-dupe collapse, capped-exemplar evidence, and one-way flow
apply to the new types unchanged — they are Events like any other.

## ~~2. Coverage diff — observed vs declared~~ (REMOVED 2026-07-18 — operator decision)

**This section is retired** with the coverage function: the
`observed_decision_points` / coverage-diff computation and the `--scenarios`
flag are gone from `tanuki-scheduler history`. The typed events §2 consumed
are NOT retired — §1's capture and §3's trajectory view stand. The original
text follows for the record:

The payoff piece, and deterministic set arithmetic end to end:

- **Observed decision points** = the distinct `user_choice.point` values in the
  execution record's event files, per scenario and per target.
- **Declared decision points** = the matrix's `axes` keys plus every axis
  claimed in any scenario's `covers`.

`tanuki-scheduler history --scenarios <file>` gains a **coverage-diff block**
(rendered after the existing coverage/debt section):

```
coverage diff (observed vs declared):
  undeclared decision points (observed in trajectories, absent from axes):
    <point> — seen in <k> scenario(s) (<ids>), <n> run(s); values seen: <v1, v2…>
      -> charter seed (plan-gated)
  declared but never observed: <axis>=<value>, …   (lifecycle states: authored | uncovered — pointer)
```

- Each **undeclared** point is printed as a charter seed with its evidence
  pointers (`run/scenario#seq`). The seed's only exit is the plan gate: the
  generation pass may present it for ratification into `axes`/`covers` or a
  new charter. The tool never writes the matrix.
- The **declared-but-never-observed** side defers to the lifecycle spec's
  existing per-value coverage states — it is the union of `authored`
  (covered by a never-executed scenario) and `uncovered` (no scenario covers
  it) — rendered as one pointer line, no duplication. Note the deliberate
  approximation: "never observed" here means *never executed* in the
  lifecycle sense; an `explored` value whose runs all predate the marker
  grammar is excluded even though no trajectory ever recorded it. When a
  scenario has executed runs but none carry typed events, the block appends
  `(<k> scenario(s) executed before trajectory capture — observations
  incomplete)` so the mixed pre/post-marker case is visible.
- Degradation: no `axes` declared → the block prints the observed-point list
  only, with the lifecycle spec's existing "no axes declared" pointer; no
  typed events yet → one line: `no trajectory events recorded — coverage
  diff needs runs made after the marker grammar landed`.

## 2b. Probe coverage — evidence predicates (ADDED 2026-07-19; findings F178/F179)

Problem (F178): the manifest's per-scenario `status` is **driver-execution
status only** — `ok` means the headless process completed. Nothing binds it
to the charter's probe being exercised, so a scenario that short-circuits
before the stage it exists to test reads *healthier* (fewer turns, clean
exit) than one that spent its budget completing the actual probe, and the
scheduler then counts its silence as fix-verification. Coverage must become
a **second, independent result axis** — never a redefinition of `status`.

**Contract — two axes, never conflated:**

- `status` (existing): did the driver complete. Unchanged.
- `probe` (new, per scenario result in `manifest.json`):
  `reached | short_circuited | undeclared`, plus
  `checkpoints: {matched: [names…], unmatched: [names…]}` when declared.
- The axes are independent of each other AND of scheduler yield: a scenario
  can short-circuit yet yield findings, or reach its probe and yield
  nothing. `probe` never enters the scheduler's yield/streak/demotion
  arithmetic (v1), and no view may merge the axes into one health verdict.

**Declaration (charter schema — normative home:
spec-tanuki-scenario-lifecycle "Charter probe block").** A charter MAY
declare a `probe` block: one **required predicate** and zero or more named
**checkpoint** predicates. Each predicate is deterministic and evaluated
over the scenario's recorded evidence (its `*.events.jsonl`, optionally the
raw stream): `{on: events|raw, type?: <event type>, match: <substring or
regex over detail>}`. Checkpoints are an **unordered set** — no ordering,
ranking, or comparability among them is ever assumed. Target-supplied stage
labels are opaque strings; Tanuki remains target-agnostic (the rejected
stage-graph alternative is recorded in the 2026-07-19 decision draft).

**Evaluation (code, no model, inside tanuki-drive's existing
normalize/manifest step):** after normalize writes the scenario's events,
evaluate the predicates; `reached` iff the required predicate matched,
`short_circuited` iff declared and unmatched, `undeclared` iff no probe
block. The verdict derives from recorded evidence, never from driver
self-report (a `TANUKI-PROBE`-style marker was rejected: the weak driver is
the component under measurement — F113 precedent).

**Rendering (spec-tanuki-view D4 applies):** the drive completion report and
run summary render the `probe` axis prominently next to `status`; an
`undeclared` scenario renders as "coverage not assessable — no probe
declared", never as healthy (no silent nothing). Predicate authoring enters
through the generation pass's existing plan gate (charters and their probes
are reviewed together; no tool writes the matrix).

**Bottleneck flag (F179, same amendment):** the manifest gains a
deterministic per-scenario `turns_anomalous` signal computed by code against
that scenario's own execution history, valid ONLY within the partition key
(scenario id + charter revision, effective driver model, entry-fixture
mode/fixture id, target/plugin pin) — cross-partition comparison is
undefined. The comparison statistic (percentile or otherwise) is
**deliberately undecided**: choosing it is a substrate change gated on this
contract landing first; until then the field is absent, and absent renders
as "not computed", never as "not anomalous".

**Boundary note — why this is not §2 revived:** the removed coverage diff
was a matrix-wide observed-vs-declared set arithmetic over decision-point
vocabulary. §2b is per-scenario, opt-in, carries no declared/observed
vocabulary diff, proposes no charter seeds, and touches no scheduler
surface. It answers one question the retired section never asked: *did this
scenario's run exercise the thing the charter exists to probe?*

## 3. Trajectory view — `tanuki-scheduler history --scenario <id> --trajectory`

The Q-by-Q rendering (build step 3 — second consumer). `--trajectory` is a boolean mode
flag composing with `history`'s existing `--scenario <id>` selector (no
second scenario-valued flag is introduced):

```
tanuki-scheduler --target <t> history --scenario <id> --trajectory [--run <run>]
```

- Renders each run of the scenario (newest first; `--run` filters to one) as
  a seq-ordered path, one compact line per step:
  ```
  <run>  (<date>, <events> events)
    #3 user_choice doctor-entry: selected=--help        (alt: README|source)
    #5 error    Bash exit 2 — missing --loop-repo
    #6 recovery of #5 — re-ran with --loop-repo
    #9 outcome  success — ready report understood
  ```
- Reads only the per-run event files
  (`~/.tanuki/<target>/events/<run>/<scenario>.events.jsonl`, plus their
  `raw.jsonl` evidence pointers); prints and exits. It writes nothing, judges nothing, scores
  nothing, and loads no ledger.
- Runs that predate typed events degrade to the `tool_error`/`result` chain
  with a `(pre-trajectory run — choices not recorded)` note.
- The plain `history` view is unchanged (pins/yield/findings); trajectory is
  strictly additive behind the flag.

## Build order

1. §1 typed event enrichment (prerequisite for both consumers).
2. §2 coverage diff — the ratified first payoff.
3. §3 trajectory renderer.

## Constraints

- Deterministic tools only: marker lift, recovery binding, set diff, and
  rendering are string/set arithmetic — no LLM stage anywhere in this spec.
- One-way flow preserved: Events → Findings → Proposals; nothing here flows
  backward, and nothing here creates a Finding or Proposal directly.
- Charter seeds enter only via the plan gate; the tools never modify
  `axes`, `covers`, scenarios, or any spec.
- Ledger growth policy: no whole-ledger reads; no trajectory storage in
  `ledger.json`; per-run event files remain the substrate, `raw.jsonl` the
  archive.
- Publication boundary: this spec states the mechanism only; decision
  provenance lives on the operator's private policy surface.

## Non-goals

- A stored trajectory entity, database, or fourth pipeline noun — the view
  is recomputed from events on every invocation.
- Trajectory-driven scheduling, scoring, or search (MCTS or any
  value-backup structure) — adaptivity stays a plan-gated scheduling nudge.
- Auto-creating charters, editing the matrix, or updating specs from
  observations — findings never auto-update specs.
- Judging events at capture (friction classification stays in extraction).
- Backfilling markers into historical runs.
- Any served API or cross-repo access layer.

## Acceptance

- A fresh drive of one scenario yields `user_choice` (with `point`/`selected`)/`outcome` events in its
  `events.jsonl` matching well-formed markers in `raw.jsonl` one-to-one; a
  run whose driver emits no outcome marker still gets a synthesized
  `outcome`.
- A scenario whose raw stream contains a `tool_error` followed by a
  successful retry of the same command yields a `recovery` event whose
  `of_seq` is that `tool_error`'s `seq`; an error with no matching
  continuation yields none.
- Evidence pointers on the new events resolve: `run/scenario#seq` names an
  event in that scenario's `events.jsonl`, and the implementation must make
  the `raw.jsonl` pointer index the actual marker line (today's
  `<scenario>.raw.jsonl#<event-seq>` format counts events, not raw lines —
  fix or replace it for typed events rather than inheriting it).
- `history` prints the coverage-diff block; introducing a marker with a
  `point` absent from `axes` surfaces it as a charter seed with correct
  evidence pointers; ratifying it via the plan gate removes it from the
  undeclared list on the next render.
- `history --scenario <id> --trajectory` renders the path above from event files alone
  (verified: no ledger read, no writes anywhere).
- All three additions leave every existing test green and every existing
  output (plain `history`, dashboard, brief) byte-identical when the new
  flags are absent and no typed events exist.
