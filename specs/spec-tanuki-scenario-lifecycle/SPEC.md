# Spec: Tanuki scenario lifecycle — init onboarding, ad-hoc scenarios, adaptive exploration

Status: PROPOSED 2026-07-14, awaiting operator ratification. Extends
`docs/tanuki-spec.md` (Driver/Miner contracts unchanged) and **amends
`specs/spec-tanuki-loop/SPEC.md`** in exactly one place (convergence — see
"Loop amendment"). Supersedes the hand-derived charter-rotation instruction
in `commands/tanuki-loop.md` with a deterministic scheduler.

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
   `tanuki-scheduler register --repo <path> --target <slug>`. **Nothing is
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
At init — and whenever the unexplored pool empties — the orchestrating
session generates charter candidates from: the plugin's README, `skills/*/
SKILL.md`, and commands (external axes × intra-command decision points), plus
the **ledger's friction history** (past finding clusters seed probes the docs
would never suggest — the blind-spot mitigation). Generated charters are
presented at a plan gate; the user approves/edits/rejects before anything is
written. Generation is the charter row of the model-tier table — frontier or
human, never delegated below.

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
Each run's scenario set, in priority order, capped by `max_scenarios`:
1. **Verify set** — scenarios evidencing findings currently
   accepted/attempted and awaiting verification-by-absence (replay to
   confirm convergence).
2. **Exploration quota** — at least `exploration_quota` (default 1)
   `unexplored` scenario or unwalked decision-point branch, whenever any
   exist. This is the anti-Goodhart guard: the loop cannot manufacture
   convergence by only re-driving quiet ground.
3. **Active rotation** — remaining active scenarios, least-recently-driven
   first.
4. **Regression pool** — members due per `regression_every`.
After mining, `tanuki-scheduler record-run --run <id>` folds the run's
`scenario-yield` into streaks and applies the state machine. `status` emits
machine-readable state; the tanuki-loop **dashboard gains a scenario
section** (state + recent yield per scenario).

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

## Exploration axes, coverage & debt (REVISED 2026-07-14 #2 — explainability)

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
