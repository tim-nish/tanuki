# Spec: Tanuki scenario lifecycle — init onboarding, ad-hoc scenarios, adaptive exploration

Status: RATIFIED 2026-07-16 (header sync — the revisions in this file were
already treated as ratified by the 2026-07-15 umbrella reconcile, d17e6fa,
and are implemented in the tools). AMENDED 2026-07-18 (triage of issue #137 —
see "Cheapest covering verify"). Extends
`docs/tanuki-spec.md` — originally with Driver/Miner contracts unchanged;
later revisions in this file also amend the Miner's compaction semantics
(driven-absence, "Verify/cap" revision) and add Driver manifest fields
("History" revision) — and **amends `specs/spec-tanuki-loop/SPEC.md`** in one
place (convergence — see "Loop amendment"). Supersedes the hand-derived
charter-rotation instruction in `commands/tanuki-loop.md` with a
deterministic scheduler.

Problem: scenario design is the biggest onboarding bottleneck — a new target
requires hand-authoring a charter matrix before the first run, and nothing
adapts the matrix afterward: the commissioning loop run drove the identical
5 scenarios twice. This spec makes scenarios **generated at init, verified by
replay, demoted on sustained low yield (never deleted), and widened by an
exploration quota** — with all arithmetic in tools and all judgment (charter
authorship) at the frontier tier, per the existing model-tier table.

## 1. Onboarding — `/tanuki init`

```
cd ~/work/<plugin-repo>
/tanuki init
```

1. Identify the repo (`git rev-parse --show-toplevel`); propose the directory
   name as the target slug; confirm.
2. **Register** `{"<abs repo path>": "<target>"}` in `~/.tanuki/registry.json`
   (Tanuki-owned, outside every target repo) via
   `tanuki-scheduler --target <slug> register --repo <path>`. **Nothing is
   ever written into the target repository** — the existing non-negotiable
   holds; a target the user does not own onboards identically.
3. Ask whether the plugin operates on a repository's content; if yes, record
   the `host` path (else hostless — per the optional-host contract).
4. **Generate the initial scenario matrix** (§3, "Generation") and write it to
   `~/.tanuki/scenarios/<target>.scenarios.json` after the user approves the
   proposed charters at a plan gate. Offer the `"loop"` block (derive a
   `test_cmd`) in the same sitting.

**Target resolution order** (all commands: /tanuki, /tanuki-loop):
1. Explicit `<target>` argument — always wins; the contract for scripts and
   headless runs.
2. cwd (or any parent directory) matches a `registry.json` entry.
3. Exactly one configured target exists — use it.
4. Otherwise: the interactive picker.
cwd is a convenience hint, never the only source of truth; resolution is
implemented by `tanuki-scheduler resolve --cwd <path>` (deterministic,
testable).

## 2. Ad-hoc scenarios (free-text)

```
/tanuki "try the export flow with a huge file"            # target resolved per §1
/tanuki my-plugin "try the export flow with a huge file"      # explicit
```

Free text is **never raw plugin input**: it becomes a one-off exploratory
scenario — id `adhoc-<n>` (prefix reserved; matrix ids must not use it),
charter auto-framed around the user's text, the standard simulated-user +
FRICTION LOG framing, normal event capture → mining → frontier dedupe →
delta report. Argument disambiguation: a non-flag argument that matches a
configured target is a target; one that matches scenario id(s) selects them
(existing semantics); anything else is ad-hoc text.

- **Persistence:** ad-hoc scenarios are NOT added to the target's matrix.
  They exist in the run's artifacts — written to `<run-dir>/scenarios.json`,
  driven from there; evidence pointers are ordinary (`<run>/adhoc-1#seq`),
  so any finding they produce lives in the ledger like any other. If an
  ad-hoc run surfaces a finding, the next generation pass may derive a
  durable charter from it (through the same plan gate — never silently).
- **vs `--ingest`:** `--ingest` records friction you *already experienced*
  (verbatim note event, no driving). An ad-hoc scenario asks Tanuki to
  *explore something now* (a driven, simulated session). Both feed the same
  ledger; document the pair as "report the past / probe the present."

## 3. Adaptive exploration (the scheduler)

### Generation (frontier judgment, human-gated)
Generation runs on **three triggers**: at **init**, whenever the **unexplored
pool empties**, and on **feature-drift** — the target plugin gained surface
(a new skill, command, or decision point) that no existing charter covers.
The first two alone leave a gap the exploration quota exists to prevent,
reintroduced one level up: the quota protects unexplored *declared* charters,
but pool-empty fires once and then never again (nothing new enters the pool),
so a plugin can grow indefinitely while every existing scenario passes and the
loop converges on a **false quiet** — the newest surface never dogfooded. The
feature-drift trigger closes that gap. It is **advisory and operator-invoked**,
never automatic: a drift signal *informs*, and the operator invokes generation
through the first-class `generate` entry (see `commands/tanuki.md`) — an
unattended loop never grows the matrix, and no tool mutates it.

**Surfacing (AMENDED 2026-07-19, #168).** An advisory trigger whose signal
appears only where the operator must remember to look is operationally dead.
The two computable triggers therefore render wherever the operator already
looks, always read-only, always pointing at the `generate` invocation:
feature-drift and pool-empty emit as `advisories` in `tanuki-scheduler
status`, as advisory lines in `history` (pool emptiness is printed, never
silent), in the status view, and — the moment that matters most — attached to
the loop's convergence report (`record-cycle`), so a run cannot converge on a
false quiet without the gap being named. The triggers themselves remain
operator-invoked; surfacing changes where they are *seen*, never who fires
them. Signal failure is explicit, never silent: when the advisory cannot be
computed (broken sibling, pre-revision tool, bad JSON), the convergence
report renders "scenario-generation advisory unavailable" naming the reason
— non-blocking, but a broken signal must never read as zero drift.

**Drift scope & matching (AMENDED 2026-07-19, #169 — operator ruled
narrow-the-spec).** Feature-drift is computed over **statically enumerable
surface only** — skills (`skills/*/SKILL.md`) and commands (`commands/*.md`).
Decision points are struck from the drift definition: observed
decision-point gaps are surfaced by `candidates` (trajectory-observed
unexplored branches), never by drift — the two signals partition the space
instead of overlapping it. Coverage matching is **exact or structured, never
substring**: a surface is covered iff its name equals a scenario's `covers`
tag (exact, case-insensitive) or appears as a whole `[a-z0-9_-]`-bounded
token in a charter or prompt. Mention-as-coverage remains the stated
heuristic limit: a charter that names a surface covers it, whether or not it
exercises it.

On any trigger the orchestrating
session generates charter candidates from: the plugin's README, `skills/*/
SKILL.md`, and commands (external axes × intra-command decision points), the
**ledger's friction history** (past finding clusters seed probes the docs
would never suggest — the blind-spot mitigation), and the **trajectory-observed
unexplored branches** — decision-point alternatives that recorded runs took a
*different* branch of (from `tanuki-scheduler history --scenario <id>
--trajectory`), which are exploration candidates the docs alone would never
surface. **Enumeration (AMENDED 2026-07-19, #171):** the deterministic
sources are enumerable through one machine surface — `tanuki-scheduler
candidates` emits the trajectory branches AND `uncovered_findings` (live
ledger findings no matrix scenario probes: empty, all-`adhoc-*`, or unknown
scenario links; ad-hoc promotion candidates flagged, host tags passed
through) — so trawling the ledger is never left to model diligence; only the
docs reading and the charter framing remain judgment. Generated charters are
presented at a plan gate; the user approves/edits/rejects before anything is
written. Generation is the charter row of the model-tier table — frontier or
human, never delegated below.

### Charter probe block (ADDED 2026-07-19 — findings F178/F179; contract home)

A charter MAY declare a `probe` block — the evidence-predicate contract
spec-tanuki-trajectory §2b consumes:

```json
"probe": {
  "required": {"on": "events", "type": "result", "match": "<regex>"},
  "checkpoints": {"<name>": {"on": "events", "match": "<regex>"}, …}
}
```

- `required` is the probe-completion predicate: the scenario's run
  **reached** its probe iff this predicate matches the recorded evidence
  (deterministic evaluation in tanuki-drive's normalize/manifest step —
  code, never a model, never driver self-report).
- `checkpoints` is an **unordered named set** of the same predicate shape;
  the manifest reports the matched/unmatched split as "furthest actually
  reached" without assuming any ordering — target-supplied stage labels are
  opaque strings, and Tanuki stays target-agnostic (no stage graph, no
  cross-checkpoint comparability).
- Absence of the block is legitimate: the result axis reads `undeclared`
  ("coverage not assessable"), never healthy-by-default.
- Probe predicates enter and change ONLY through the same plan gate as the
  charter they belong to — generation proposes them alongside new charters
  (a probe the human never reviewed is a coverage verdict the human never
  authorized), and no tool mutates the matrix.
- Coverage is independent of yield (a short-circuited run may still yield
  findings) and never enters the scheduler's yield/streak/demotion
  arithmetic in v1.

### Per-scenario state machine (deterministic, tool-owned)
State lives in `~/.tanuki/<target>/scheduler.json`, owned by the new
zero-dependency tool `tools/tanuki-scheduler`; the model never edits it.

```
              first drive                2 consecutive low-yield runs
 unexplored ────────────► active ──────────────────────► regression
                            ▲                                 │
                            └───── actionable finding ◄───────┘
                                   (re-promotion)      driven every
                                                       regression_every runs
```

- **unexplored** — generated, never driven. Highest priority (exploration).
- **active** — in the regular rotation.
- **regression** — sustained low yield; driven every `regression_every`
  (default 3) runs. **Never deleted**: a fix that regresses months later must
  still be caught, and any actionable finding re-promotes to active.
- Demotion needs **hysteresis**: `demote_after` (default 2) *consecutive*
  low-yield runs — one quiet run from the deliberately weak driver is noise,
  not a signal.

### Yield (deterministic arithmetic)
New `tanuki-ledger scenario-yield [--runs N]` computes, per scenario, from
evidence pointers already in the ledger: actionable findings (new or bumped)
attributable to that scenario in its last N driven runs, plus event counts.
"Low yield" = zero actionable findings in a run (`low_yield_threshold`,
default 0). Works retroactively on existing ledgers — evidence has always
carried scenario ids.

### Selection (`tanuki-scheduler plan`)
Each run's scenario set, in priority order, capped by `max_scenarios`
(order REVISED by the Verify/cap revision below — exploration is reserved
*before* verify fills):
1. **Exploration quota (reserved)** — at least `exploration_quota` (default
   1) `unexplored` scenario or unwalked decision-point branch, whenever any
   exist; `plan` reserves these slots before filling verify. This is the
   anti-Goodhart guard: the loop cannot manufacture convergence by only
   re-driving quiet ground.
2. **Verify set** — scenarios evidencing findings currently
   accepted/attempted and awaiting verification-by-absence (replay to
   confirm convergence), capped per the Verify/cap revision.
3. **Active rotation** — remaining active scenarios, least-recently-driven
   first.
4. **Regression pool** — members due per `regression_every`.
After mining, `tanuki-scheduler record-run --run <id>` folds the run's
`scenario-yield` into streaks and applies the state machine. `status` emits
machine-readable state; per-scenario state and yield live in the
**history view** (`tanuki-scheduler history`) — the tanuki-loop dashboard
keeps its fixed skeleton (spec-tanuki-loop "Fixed skeleton") and points at
history for the long view.

### Loop amendment (spec-tanuki-loop)
A drive→mine cycle counts toward the loop's quiet streak **only if its plan
satisfied the exploration quota** (or the unexplored pool was empty). The
two-consecutive-quiet convergence rule is otherwise unchanged. The loop's
charter-rotation step now calls `tanuki-scheduler plan` instead of
hand-deriving a variant matrix.

### Config keys (added to the table in docs/tanuki-spec.md)
`demote_after` 2 · `regression_every` 3 · `exploration_quota` 1 ·
`low_yield_threshold` 0.

## History & coverage (REVISED 2026-07-14 — cross-run scenario monitoring)

The dashboard is run-scoped ("what is happening now"). Historical monitoring
is target-scoped and cross-run — per scenario: execution count/dates/run ids,
decision-point choices used, findings per execution and cumulative,
actionable vs minor, recurrence, low-yield streaks, current classification,
and **why/when its priority changed**; repo-wide: coverage (configured /
unexplored / explored branches), recently productive, long-unrun, and the
selection history with reasons. All of it derives from **persisted artifacts,
never model memory** — which requires three additive persistence changes:

1. **Manifests carry the facts of the run** (`tanuki-drive`): each per-scenario
   result records its `decision_points` **verbatim** (the pins used, not just
   a count — two runs of one scenario down different branches must be
   distinguishable later), and the manifest records `completed_at`
   (ISO timestamp; run ids only carry a date).
2. **`record-run` snapshots yield before it erodes** (`tanuki-scheduler`):
   evidence capping keeps oldest + most-recent pointers, so late
   recomputation of old runs undercounts. `record-run` therefore persists the
   run's per-scenario yield (`{run: {scenario: {actionable, findings[]}}}`)
   into `scheduler.json` at record time, while attribution is fresh — the
   durable per-execution record.
3. **Transitions and plans are persisted, not just printed**
   (`tanuki-scheduler`): every state transition appends
   `{run, from, to, reason}` to the scenario's `transitions` list, and every
   `plan` (now taking `--run <id>`) appends
   `{run, plan, verify, explore, due, quota_met}` to a `plans` list — the
   selection history with reasons.

**Membership history (AMENDED 2026-07-19, #170).** A fourth additive
persistence change, same pattern: matrix *entry* is recorded, not just later
priority changes. The generate workflow authors the pass record at plan-gate
close and hands it to `sync --generation '{"trigger", "proposed",
"rejected"}'` (trigger is open vocabulary — `init`, `drift`, `pool-empty`,
`on-demand`, `host-change`, …); sync appends it to a `generations` list in
`scheduler.json`, adding what only the tool knows (`added` from the matrix
diff — never trusted from the caller — and the timestamp). A bare sync that
added ids appends a membership-only `source: "sync"` entry: hand-edits are
recorded, never dressed up as generation passes. Rendered in `history`
(repo-wide list; per-scenario `entered` in the deep view). Pre-revision
state simply has no list — the standing lossiness boundary applies.

**Interface:** a new `tanuki-scheduler history` subcommand — repo-wide
coverage table by default; `--scenario <id>` for the single-scenario deep
view (executions with dates/pins/yield, cumulative findings + recurrence,
streaks, classification, transition log). Surfaced as
`/tanuki <target> --history [scenario]`. The tanuki-loop dashboard stays
live-run-scoped and simply points at history for the long view.

**Lossiness boundary (stated, not hidden):** runs recorded before this
revision have no yield snapshot or transition log; history reconstructs them
best-effort from manifests + (capped) ledger evidence and labels them
`approx`. Everything after this revision is exact. The selection model
itself is unchanged — this section only *records* what it already does.

## ~~Exploration axes, coverage & debt~~ (REMOVED 2026-07-18 — operator decision)

**This entire section is retired.** In four days of operation no target ever
declared an `axes` block, the observed-only coverage diff accumulated 99
driver-improvised axis names with nothing to compare them to, and the
operator ruled the function has no demand. Removed with it:
`tanuki-scheduler history --scenarios` (the flag and the axis-coverage /
exploration-debt / recommendations / coverage-diff computation and output),
the `coverage` view (spec-tanuki-view D2, amended the same day), and the
generation pass's obligation to declare `axes`/`covers` (matrices may still
carry the keys; tools ignore them). **Kept:** `decision_points` event
capture and the typed trajectory events — the `trajectory` view consumes
them (spec-tanuki-trajectory §1/§3, unaffected); the repo-wide scenario
states (unexplored / long-unrun / productive) and the plan's exploration
quota, which never depended on axes. The original text follows for the
record:

History must answer *"what have I never exercised?"*, not only *"what did I
execute?"*. Coverage is the intersection of a **declared exploration space**
and the **execution record**. The execution record is already persisted (the
History & coverage revision); the space is not — it exists only in charter
prose, so "unexplored" is uncomputable. Two additive schema elements close
this, both in the scenarios file (Tanuki-owned, never the target repo):

1. **`axes`** (top-level, optional): the declared exploration space —
   `{"<axis>": {"values": ["…"], "note"?: "…"}}`, e.g. `framework:
   [F1,F2,F3,F4]`, `article_intent: [devlog, survey, tutorial, postmortem,
   eval]`, `host_state: [configured, unconfigured, multi-root]`,
   `error_path: [none, missing-config, empty-output]`, plus decision-point
   branch spaces and cross-command flows as axes like any other.
2. **`covers`** (per scenario, optional): structured coverage claims —
   `{"<axis>": ["<value>", …]}`. Canonical where loose prose keys (e.g. an
   `intent` field) are not: tools read only `covers`.

**Authorship is frontier judgment, gated:** the generation pass (init /
regeneration) declares and maintains `axes` and tags `covers`, presented at
the same plan gate as the charters — declaring the branching space IS the
charter row of the model-tier table. Tools never invent, rename, or extend
axes; they only compute over what generation ratified.

**Coverage semantics (deterministic, per axis value):**
- **explored** — ≥1 *executed* scenario covers the value;
- **authored** — covered only by scenarios never executed (charter exists,
  never run) — partially explored;
- **uncovered** — no scenario covers it: the true blind spot.
Axis rollup: fully / partially / never explored. Computed by
`tanuki-scheduler history --scenarios <file>` from `axes`/`covers` plus the
persisted execution record; rendered as a compact per-axis block and included
in `--json`.

**Exploration debt (auto-generated, end of history output):** axis values
uncovered (needs a new charter) · values authored-but-never-run (run the
existing scenario) · stale scenarios (long-unrun, already computed) ·
declared decision-point branches with no executed pin. Followed by at most
**3 recommendations**, ranked by a deterministic greedy rule — first the
existing never-run scenarios covering the most distinct debt values ("run
X"), then uncovered-value combinations ("author a charter covering
framework=F1 + intent=tutorial") for what no scenario covers. Recommendations
are **advisory only**: history never modifies the matrix — new charters enter
only through the plan-gated generation pass (proposals-only, inherited).

**Degradation:** a matrix with no `axes` skips coverage/debt with a one-line
pointer ("no axes declared — run a generation pass to declare the exploration
space"); existing targets are unaffected until their next generation pass
adds the block.

## Verify / cap interaction (REVISED 2026-07-15 — verify never starves exploration; absence means driven-absence)

The original Selection order (Verify → Exploration quota → Active → Regression,
capped by `max_scenarios`) has two coupled defects, observed on a live target
whose accepted-but-unverified backlog grew to fill the whole cap:

1. **Verify starves exploration.** `plan` fills the verify set *first*, up to
   `max_scenarios`. When the verify set (scenarios evidencing accepted findings
   awaiting verification-by-absence) is as large as the cap, the exploration
   quota gets zero slots — `quota_met` goes false while unexplored scenarios
   remain. That silently defeats the anti-Goodhart guard this spec already
   declares and, via the loop amendment, makes the loop unable to converge.
2. **Absence is measured as elapsed runs, not driven-absence.** `tanuki-ledger
   compact` tombstones an `accepted` finding as *fixed* once it is "unseen for
   ≥ N runs" — but the count is elapsed runs, regardless of whether the
   finding's scenario was actually **driven** in them. A verify-pinned scenario
   that stops being driven (crowded out of the cap by defect 1, or removed from
   the matrix) is declared fixed **without ever being replayed** — a false
   verification. The two defects compound: (1) drops the replay, (2) then calls
   it fixed.

**Fix — three deterministic changes (tool arithmetic only; the typed charter
grid and the dry rule are unchanged — no reward-maximizing / value-backup
search enters selection):**

- **Reserved exploration quota.** `plan` reserves `exploration_quota` slots for
  `unexplored` scenarios/branches *before* filling verify. Coverage exploration
  is never traded away to re-drive known-productive ground; it is protected as
  budget arithmetic, not left to cap tuning.
- **Bounded, fairly-rotated verify.** Verify fills the remaining budget
  (`max_scenarios − reserved`), **least-recently-planned first**, so every
  accepted-finding scenario eventually replays and the backlog drains while a
  run still stays within `max_scenarios`. When the verify set exceeds its
  budget the plan record carries `verify_deferred` (the count/ids not run this
  cycle) — the backlog is a **bounded, surfaced** work-in-progress signal,
  never a silent truncation.
- **Driven-absence compaction.** A run counts toward an accepted finding's
  verify-by-absence only if at least one of the finding's scenarios was
  **actually driven** in that run (read from the run manifest) and the finding
  did not recur. "Unseen for N runs" now means "replayed N times and gone,"
  not "N runs elapsed." A finding with no recorded scenarios falls back to
  elapsed-run counting (legacy compatibility); `dismissed` findings are
  unaffected (deadness is not scenario-conditional).

**Consequences / invariants.**
- `quota_met` can no longer be false while unexplored scenarios exist and
  `max_scenarios ≥ exploration_quota` — restoring the loop amendment's
  quiet-cycle precondition and, with it, convergence.
- A verify replay deferred by the cap is now **safe**: driven-absence means a
  deferred scenario accrues no false absence; it waits its turn in the LRU
  rotation. Raising `max_scenarios` drains a large backlog faster, but is a
  tuning knob, not a correctness dependency.
- Removing or renaming a matrix scenario referenced by an accepted finding no
  longer manufactures a false "fixed" tombstone (it simply stops accruing
  absence and surfaces via `verify_deferred`). Scenarios are still never
  deleted — regression pool, not removal (unchanged rule).

## Cheapest covering verify (AMENDED 2026-07-18 — triage of issue #137)

Findings often surface in expensive end-to-end scenarios while touching a
surface a cheap scenario also covers; pinning verify to the originating
scenario makes verify cost track *where a finding happened to appear*, not
what it takes to reproduce it.

**The extension — deterministic arithmetic only, no new judgment in the
loop:**

- **`covers:` tags.** A scenario may declare `covers: [<surface tag>, …]`
  (stage/surface names, target-local vocabulary). Mining records the
  finding's surface tag(s) alongside its scenario link when the evidence
  supports one; a finding may also carry none (then nothing changes for it).
- **Cheapest covering selection.** For each verify-set entry, `plan` picks
  the **cheapest scenario that covers the finding's surface** — by an
  explicit cost key when present, else `max_turns` — with the originating
  scenario as the fallback when no cheaper cover exists. A finding may be
  **pinned** to its originating scenario (per-finding flag) when
  reproduction genuinely needs the full path. Selection remains part of the
  pre-approved plan: deterministic inputs, no mid-run judgment.
- **Covering-absence compaction.** Driven-absence generalizes: a run counts
  toward an accepted finding's verify-by-absence when a **covering**
  scenario (originating included) was actually driven in it and the finding
  did not recur. "Replayed N times and gone" now means replayed via any
  declared cover.
- **Under-reproduction guard (re-pin).** A cheap cover can fail to reproduce
  what a full pipeline would. If a finding accrues cover-driven absence but
  **recurs anywhere** before tombstoning, the recurrence resets absence (as
  today) AND re-pins the finding's verify entry to its originating scenario
  — ledger arithmetic, no model judgment. The LRU rotation and the reserved
  exploration quota are unchanged; covers only change *which* scenario fills
  a verify slot, never how many slots verify gets.

**Consequences.** Verify cost decouples from surface-of-first-appearance;
short scenarios (see spec-host-snapshot "Stage-entry fixtures") become the
scheduler's preferred verify vehicles; a finding with no `covers` match
behaves exactly as before (originating scenario, driven-absence).

**Constraints carried by this spec** (folded so the implementation order needs
no external attachment):
- A scheduler must not concentrate budget on known-productive branches at the
  expense of under-tested ones; coverage-driven exploration is protected as
  architecture (a reserved quota), not a tunable. Exploitation of the fattest
  branch is explicitly rejected as the selection policy.
- The accepted-but-unverified set is **bounded work-in-progress**: the plan
  caps it and surfaces the overflow (`verify_deferred`) rather than letting it
  grow unbounded or silently dropping replays.
- All selection and compaction arithmetic stays deterministic tool code.

**Non-goals (unchanged by this revision):** demotion hysteresis, re-promotion,
regression cadence, the dry-rule convergence definition, and the plan-gated
charter-generation pass are all untouched. No new config keys; `max_scenarios`
and `exploration_quota` keep their meanings.

## Host bindings & declared-input configuration (RATIFIED 2026-07-19 — decomposed 2026-07-19)

Design records: issues #172 (declared-input configuration surface) and #173
(host portability); identity ruling in
`DECISION-2026-07-19-host-identity.md` (this directory). The invariants,
folded here so the spec is self-contained; the issues carry the full design:

- **Execution identity is `(scenario_id, host_binding_id)`.** Scheduler
  state — yield, streaks, demotion/regression, transitions — is kept per
  host binding; ledger evidence carries the binding id; recurrence and
  driven-absence evaluate one binding at a time. Single-binding targets are
  unchanged until a second binding is declared (no migration).
- **`host_binding_id` is logically stable and separate from the commit
  pin.** It names the declared binding, never a snapshot; re-pinning or
  advancing the host repo never changes identity. Renaming a binding is an
  explicit operator act.
- **Configuration is a declared-input surface, not a hidden file hunt.**
  Targets declare editable fields (`inputs` block: type from a small generic
  vocabulary, scope, binding point, optional opaque doctor hook); Tanuki
  validates by type only and never interprets meaning. The surface is an
  action in the existing `/tanuki` picker — no new top-level command. The
  backing JSON stays the source of truth; hand-editing remains legal.
- **A one-shot per-run override never touches canonical state.** It expires
  after exactly one drive, the manifest records the effective resolved
  inputs, an active override renders loudly, and the run is excluded from
  `record-run` and driven-absence — its evidence enters the ledger
  host-tagged. Full per-host accrual applies only to declared bindings.
- **Portability is fail-closed and surfaced.** The compatibility check runs
  at plan time; skips persist in the plan record (`compat_skipped`, the
  `verify_deferred` posture); a compat-skipped verify replay accrues no
  driven-absence; a compat-skipped exploration slot forces `quota_met`
  false. Host-coupling detection renders through the existing read-only
  surfaces, never a new door.

Decomposition: stories 1.27–1.29 (umbrella #172) and 1.30–1.33 (umbrella
#173) — created `ready`, flowing through /publish-issues → /implement-story.

## Compatibility and migration

- **Existing targets keep working untouched.** A hand-written matrix with no
  scheduler state initializes every scenario as `active` on first `plan`;
  behavior converges gradually as yields accumulate. `registry.json` absent →
  resolution falls through to today's explicit/picker behavior.
- Ad-hoc ids are namespaced (`adhoc-`); no collision with matrices.
- Yield computes from historical ledger evidence; no data migration.
- The loop amendment tightens convergence (a run skipping exploration can no
  longer count as quiet); it cannot loosen it.

## Rules

- Nothing is ever written into a target repository; registry, matrices,
  scheduler state, and ledgers are all Tanuki-owned, outside the target.
- All scheduling/yield/registry arithmetic is tool code; charter generation
  and ad-hoc framing are frontier judgment behind a plan gate.
- Scenarios are never deleted — regression pool, not removal.
- Explicit target arguments always work; cwd is a hint via the registry.
- Demotion requires hysteresis; convergence requires the exploration quota.
