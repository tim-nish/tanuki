Tanuki — automated dogfooding of a Claude Code plugin: drive branching
scenarios in disposable clones, mine the recorded Events into deduplicated
Findings, and consolidate chronic ones into a capped, ranked Proposal brief.
Proposals-only: nothing merges, nothing writes to the target repos, findings
never auto-update specs. Contract: `${CLAUDE_PLUGIN_ROOT}/docs/tanuki-spec.md`.

Every `tanuki-*` tool named below is the executable under
`${CLAUDE_PLUGIN_ROOT}/tools/` (not on PATH — invoke by full path).

**Target resolution** (this command and /tanuki-loop, fixed order): an
explicit `<target>` argument always wins (the contract for scripts and
headless runs) → else the cwd's registered target
(`tanuki-scheduler resolve --cwd $PWD` walks parent dirs over
`~/.tanuki/registry.json`) → else the single configured target if exactly one
exists → else the interactive picker. cwd is a convenience hint via the
registry, never the only source of truth.

**Command shape (the rule — `specs/spec-short-command-surface/SPEC.md` D6):**
a **bare word selects a mode that does not drive**; **flags modify a drive**;
the bare default is driving. Every non-driving mode below is a word; the
previously documented flag spellings are retained as aliases and marked as
such. A new mode is a word if it does not drive — the rule answers it, not
precedent.

Argument handling ($ARGUMENTS):
- `init`: **onboarding** — run from inside the plugin repo (see "Init" below).
- *(empty)*: resolve per the order above; when nothing resolves, the
  **target picker** — list every
  `~/.tanuki/scenarios/*.scenarios.json` target with its one-line
  ledger `status` summary (runs, proposed awaiting decision, latest brief) and
  ask which to run via AskUserQuestion. Selection beats typing names with no
  completion.
- `<target>`: a slug with a scenario config at
  `~/.tanuki/scenarios/<target>.scenarios.json`. Example: `my-plugin`.
- `<target> <scenario-id>[,<scenario-id>…]`: drive only those scenarios.
- `"<free text>"` (with or without a preceding `<target>`): an **ad-hoc
  scenario** — see "Ad-hoc scenarios" below. Disambiguation, in order: a
  **mode word** (`init`, `decide`, `status`, `history`, `view`, `mine`,
  `ingest`, `generate`, `configure`, `distill`) always wins — the closed set above is reserved, so a target or
  scenario sharing one of those names is addressed as `/tanuki <target>
  <mode>` (the mode word is only read in mode position, after the target);
  then an argument matching a configured target is a target; then one
  matching scenario id(s) selects them; anything else is ad-hoc text. A
  quoted argument is always ad-hoc text, never a mode word.
- `<target> decide`: the **decision pass** — the complete, ledger-anchored
  path from whatever the ledger holds decidable to labeled issues
  (spec-short-command-surface D1). Orients off `status`/`next`; needs no
  recent run or brief. No driving.
  *Alias:* `<target> --brief` — reprints the latest brief, then continues
  into the same pass (one code path, not a second behavior).
- `<target> status`: run `tanuki-ledger --target <target> status` and show
  it, then `tanuki-scheduler --target <target> status` and render any
  non-empty `advisories` (uncovered plugin surface / unexplored pool empty,
  each pointing at `generate`) as one line each — advisory generation-trigger
  signals, #168. No driving. *Alias:* `--status`.
- `<target> history [scenario]`: cross-run scenario monitoring — run
  `tanuki-scheduler --target <target> history [--scenario <id>]` and show
  it. Repo-wide: per-scenario
  state/executions/streaks/recurrence, unexplored, long-unrun, recently
  productive, the selection history with reasons. With a scenario id: every
  execution (date, decision-point pins,
  yield, findings) plus its transition log. Built from persisted artifacts
  only (manifests, ledger, scheduler state) — never model
  memory. No driving. *Alias:* `--history`. The
  per-run trajectory render also has a named door of its own:
  `view trajectory` (see "Views" below). (The exploration-coverage/debt
  block and its `--scenarios` flag were REMOVED 2026-07-18 — operator
  decision; spec-tanuki-view D2.)
- `<target> view [name]`: the **view surface** — one option-free door to
  every read-only view this repo computes
  (`specs/spec-tanuki-view/SPEC.md` D1/D2). Bare `view` presents the view
  picker; `view <name>` jumps straight to a named view from the closed
  catalog (`status`, `live`, `history`, `trajectory`). Read-only,
  never writes, safe alongside a running loop. No driving. See "Views" below.
- `<target> mine <run-id>`: skip driving; mine + consolidate an
  existing run's events (use after a crashed or interrupted session).
  *Alias:* `--mine-only <run-id>`.
- `<target> ingest "<feedback>"`: record human feedback in natural
  language (no driving) and mine it immediately — see "Ingest mode" below.
  *Alias:* `--ingest "<feedback>"`.
  The human never classifies; the tool stores the note verbatim and the
  normal extraction + dedupe passes decide everything downstream.
- `<target> generate`: the **regeneration pass** — the first-class,
  option-free entry to charter generation (a bare word: it does not drive,
  spec-short-command-surface D6). Runs the same frontier-judgment, human-gated
  generation the init flow's step 4 runs (§3 "Generation" in
  `${CLAUDE_PLUGIN_ROOT}/specs/spec-tanuki-scenario-lifecycle/SPEC.md`),
  reachable on any of its three triggers — **feature-drift** (the target
  gained a skill/command/decision-point no charter covers), an **empty
  unexplored pool**, or on demand. Candidate pool: the plugin's docs (judgment
  — read them yourself), and the deterministic remainder enumerated by ONE
  machine surface, `tanuki-scheduler --target <t> candidates --json` (#171):
  trajectory-observed unexplored branches (decision-point alternatives
  recorded runs never took) plus `uncovered_findings` — the ledger's friction
  history no matrix scenario probes, with ad-hoc promotion candidates
  flagged (`adhoc_origin`) and host tags passed through. Proposes charters at
  the plan gate — each proposal MAY carry a `probe` block (one `required`
  evidence predicate plus named `checkpoints`; story 1.24, normative schema
  in the lifecycle spec's "Charter probe block"), rendered alongside its
  charter so probe and charter are approved, edited, or rejected together
  (a probe the human never reviewed is a coverage verdict the human never
  authorized); the user approves/edits/rejects; only then are they written
  and `tanuki-scheduler sync`'d (new ids enter as unexplored) — the sync call
  carries the pass's finalized record,
  `--generation '{"trigger": "...", "proposed": N, "rejected": N}'`, so the
  membership history is persisted at the gate, never reconstructed later
  (#170; trigger is open vocabulary: `drift`, `pool-empty`, `on-demand`, …).
  Advisory and
  operator-invoked — never automatic, never during an unattended loop, and no
  tool mutates the matrix. No driving.

- `<target> configure`: **declared-input configuration** (story 1.28 /
  issue #172; contract: the "Host bindings & declared-input configuration"
  spec section). A bare word — it does not drive. The flow, entirely over
  `tanuki-config` (deterministic substrate; render, don't compute):
  1. Render `tanuki-config --target <t> show` — effective configuration with
     each value's source labeled; undeclared fields are read-only. With no
     `inputs` block, show's one-line state is the whole answer plus a
     pointer at declaring one — never a guess.
  2. Pick the field to change via AskUserQuestion (declared fields only —
     selection beats typing), then collect the new value.
  3. `tanuki-config check --field F --value V` — typed validation; a
     failure is relayed and the flow returns to step 2.
  4. `tanuki-config set --field F --value V --dry-run` — show the resulting
     change and the exact storage path; ask for approval. Decline at any
     step ⇒ nothing was written.
  5. On approval, `tanuki-config set --field F --value V` — if the field
     declares a doctor, the exact command line is echoed before it runs
     (the `test_cmd` posture) and its output is relayed verbatim; a failing
     doctor blocks the persist by default. Overriding a failed doctor is
     its own explicit question, never a silent retry
     (`--override-doctor`).
  Read-only until step 5; the backing JSON stays the source of truth and
  hand-editing remains legal.

- `<target> distill`: **distill den** (story 1.36 / spec-den-distill §4) —
  a bare word, it does not drive. Runs the SAME lesson-candidate walk as the
  decision pass's 4.1c over the *existing* ledger: contributing past history
  never requires a new run. One code path, two entries — the walk, the
  accept-emits-via-`tanuki-ledger distill --id` semantics, the verbatim
  receipts, and the dispositions-by-`set-status` rule are 4.1c's, verbatim.
  Substrate for the pool and the state hint:
  `tanuki-ledger --target <t> distill --list` (`configured` + candidate
  count). Empty states are one typed line each, never a crash or an empty
  walk: no undecided candidates → "no lesson candidates awaiting decision";
  candidates present but `contribute_back` unset → the not-configured
  notice below. When a picker is presented for this target (e.g. the bare
  `/tanuki` target picker's per-target hint line), the distill hint is
  state-derived from the same `--list` output — "N lesson candidates
  awaiting decision" — never re-derived.

  **The not-configured notice (one line, everywhere candidates render):**
  "N lesson candidates; contribute-back not configured — configure
  `contribute_back {path, schema}` to stage them into your knowledge hub."
  Rendered by the brief's lesson-candidates section and by 4.1c/this mode in
  place of a walk; nothing is written while unconfigured.

## Init (`/tanuki init` — the normal onboarding flow)

Run from inside the plugin repo (contract:
`${CLAUDE_PLUGIN_ROOT}/specs/spec-tanuki-scenario-lifecycle/SPEC.md`):
1. Identify the repo (`git rev-parse --show-toplevel`); propose its directory
   name as the target slug; confirm with the user.
2. Register it: `tanuki-scheduler --target <slug> register --repo <root>` —
   the mapping lives in `~/.tanuki/registry.json`. **Never write any
   configuration into the target repository.**
3. Ask whether the plugin operates on a repository's content; if yes, record
   the `host` path in the scenarios file; otherwise omit `host` (hostless).
4. **Generate the initial scenario matrix** — this is the charter row of the
   model-tier table (frontier work, yours): read the plugin's README,
   `skills/*/SKILL.md`, and commands for the external axes and intra-command
   decision points, AND the ledger's finding history when one exists (past
   friction seeds probes the docs would never suggest). Propose 4–6 charters
   (including at least one pinned decision-point branch and one error-path).
   A charter that exists to probe a specific stage MAY declare a `probe`
   block (lifecycle spec "Charter probe block") — propose it with the
   charter; the two are reviewed together at the gate, never separately.
   Present everything at one plan gate, apply the user's edits, then write
   `~/.tanuki/scenarios/<slug>.scenarios.json` and run
   `tanuki-scheduler --target <slug> sync --scenarios <file>
   --generation '{"trigger": "init", "proposed": N, "rejected": N}'` —
   the persisted membership record, #170. (Declaring an
   `axes`/`covers` exploration space was REMOVED 2026-07-18 — operator
   decision; a matrix may still carry the keys, tools ignore them.)
5. Offer the `"loop"` block (derive a `test_cmd` per /tanuki-loop's rule).

Manual alternative (still supported): copy an existing scenarios file (or the
bundled `${CLAUDE_PLUGIN_ROOT}/templates/example.scenarios.json`) to
`~/.tanuki/scenarios/<new-target>.scenarios.json` and edit `plugin`/`host`/
scenarios. Offer init when the user names a target that has no config yet.
Regenerate more charters through the first-class `generate` mode (`/tanuki
<target> generate`) on any of its triggers — feature-drift, an empty
unexplored pool, or on demand (same plan gate).

## Views (`/tanuki [target] view [name]` — the read-only surface)

Contract: `${CLAUDE_PLUGIN_ROOT}/specs/spec-tanuki-view/SPEC.md`. The
principle: **every read-only view is reachable from one option-free command;
the tools that compute the views remain the fully-optioned machine
substrate.** This surface **reads and renders; it never computes** what a
tool could compute, and it never writes — no ledger writes, no scheduler
writes, no matrix edits, no dispositions (those live in `decide`), and no
prompts in headless mode (the picker is an attended convenience only).

**Entry (D1).** Resolve the target exactly as this command always does (the
fixed order above — nothing view-specific). Then:

- Bare `view`: present the **view picker** via AskUserQuestion — **all four**
  catalog views (`status`, `live`, `history`, `trajectory`) as named,
  selectable options, each with its one-line description and a
  **state-derived hint** of whether it currently has anything to say (e.g.
  `status — 4 proposed awaiting decision`, `live — no loop run yet`). Hints come
  from the substrates' own outputs (`--json` where it exists), never from
  re-derivation. The four views fit AskUserQuestion's four-option cap exactly,
  so **every view is always one of the named options** — never demote a view to
  the harness's free-text / `Other` affordance, and never drop a view because it
  was already viewed this session: membership and order come from the D2 catalog
  (below), not from session history (issue #103). Selection beats typing.
- `view <name>`: jump straight to that view. Names come from the catalog
  below and nowhere else; an unknown name gets the picker plus a one-line
  note, never a guess.

**The catalog (D2 — a closed enumeration).** Adding a view means amending
the spec's list first; an unenumerated view is the defect this surface
exists to fix. Each view names its substrate — run it, render its output:

- **`status`** — ledger counts + the derived next step, plus the scheduler's
  advisory generation-trigger signals when non-empty.
  Substrate: `tanuki-ledger --target <t> status` and `next` (the
  short-command-surface D2 derivation; never recomputed here), and
  `tanuki-scheduler --target <t> status` for `advisories` (#168 —
  rendered verbatim, one line per signal).
- **`live`** — the loop dashboard **of an actively running loop**: health
  verdict, latest drive, this run, scheduler decisions, convergence,
  why-stopped/NEXT.
  Substrate: `tanuki-loop --target <t> dashboard --live` (offer
  `--follow 10` when a loop is running). The **live** view; `history` is
  the long view. **Live means active**: a closed run — however recent — is
  history, never live. With no active run the substrate renders its typed
  empty state plus ONE line of historical context (run id, computational
  stop reason — cap/converged/breaker/cancelled, never a settlement, which
  is what became of the delivery — the delivery with its derived settlement
  result, and the frozen execution time); render it verbatim and do not add
  the closed run's detail sections. The line's settlement is derived
  offline (local reachability or the stale-marked cache) — a view never
  polls the forge. For a closed run's full dashboard the operator asks for
  it explicitly (`tanuki-loop dashboard`, no `--live`) — not through
  `view live`.
- **`history`** — cross-run per-scenario execution history: state,
  executions, streaks, recurrence, unexplored, long-unrun, recently
  productive, selection history with reasons.
  Substrate: `tanuki-scheduler --target <t> history`.
- **`trajectory`** — one run's step-by-step path (choices, errors,
  recoveries, outcome) from its event files alone.
  Substrate: `tanuki-scheduler --target <t> history --scenario <id>
  --trajectory [--run <run-id>]`. This is the proper name for what
  `--history <scenario>` used to overload; when no scenario id was given,
  pick one the same way as everything else — list the matrix's scenarios via
  AskUserQuestion, selection beats typing.

**No silent nothing (D3).** Every catalog view is always *offered*; a view
whose substrate is missing states why in **one line**, at the picker and in
the view itself, carrying the expected-vs-gap signal: whether emptiness is
normal for this target's state (a fresh target has no runs — expected) or a
real gap the operator should close (iterations ran but no plan persisted —
GAP). Where the substrate emits its typed empty state
(`tools/tanuki-loop` `EMPTY_STATES`: `{state, expected, reason, next}`),
render that verbatim — one truth, the tool's. Any command a reason names
must be explained where it is named, never assumed known.

**Render, don't compute (D4 — a standing acceptance rule).** Every number,
verdict, and ranking a view shows comes from a substrate command's output.
The command layer composes, labels, and orders; it never re-derives. A view
needing a number no tool emits is a substrate change with its own spec and
tests — never arithmetic done in prose here. No view introduces a term its
substrate rejects, invokes downstream tooling, or modifies anything.

**Vocabulary is target-local (v1 — spec-tanuki-view OPEN-2).** A view
renders the selected target's axis names exactly as its substrate emits
them: no normalization, no titlecasing, no substitutions, and no
comparability claim across targets. Cross-target comparison is out of scope
and requires a separate future specification.

## Ad-hoc scenarios (free text — "probe the present")

Free text is never raw plugin input: wrap it as a one-off exploratory
scenario and run it through the normal pipeline. Build a single-scenario file
in the run dir (`<run-dir>/scenarios.json`): id `adhoc-1` (the `adhoc-`
prefix is reserved — matrix ids may not use it; `tanuki-scheduler sync`
enforces this), a charter you frame around the user's text (goal + persona),
the user's text as the prompt core, and the standard simulated-user +
FRICTION LOG framing (tanuki-drive adds it). Then: drive that file → mine →
frontier dedupe → delta report, exactly like any run. Ad-hoc scenarios are
**not persisted** to the target's matrix and are invisible to the scheduler;
their findings live in the ledger like any other (evidence
`<run>/adhoc-1#seq`), and a later generation pass may derive a durable
charter from them — through the plan gate, never silently.

Distinguish from `ingest`: **`ingest` reports the past** (friction the
human already experienced; verbatim note event, no driving); **an ad-hoc
scenario probes the present** (a driven, simulated session exploring
something now). Both feed the same ledger.

Working-file discipline: every intermediate you produce (candidate files,
notes, extracts) is written under `~/.tanuki/<target>/events/<run>/` — never
the session temp/scratchpad dir, never the target repos. Everything a run
produced must be findable from `~/.tanuki` afterwards.

Cost display: user-facing output (plan, progress, brief, summaries) uses
**time and turns, never dollars**. USD lives in manifests as estimate
history only.

Vocabulary (fixed): an **Event** is a raw per-run fact, never judged;
a **Finding** is a judged, deduped, ledger-tracked signal with a recurrence
count; a **Proposal** is a finding promoted through the human gate. There are
no "observations".

Model tiers (fixed — see the routing table in ${CLAUDE_PLUGIN_ROOT}/docs/tanuki-spec.md): scenario
execution and friction extraction run on cheap models (a weak driver is a more
sensitive friction detector); semantic dedupe and the brief are yours (the
frontier session) and are never downgraded; normalization and the promotion
decision are code and never involve a model.

Configuration: `~/.tanuki/config.json` < the target file's `"defaults"` block
< CLI flags; all keys and built-in defaults are tabled in ${CLAUDE_PLUGIN_ROOT}/docs/tanuki-spec.md
and mirrored in `${CLAUDE_PLUGIN_ROOT}/templates/config.example.json`. Two
keys shape this
command's behavior directly: `model_ceiling` (highest tier Tanuki may launch —
applies to the driver AND the extraction subagent you spawn in step 2.2;
Fable/Mythos-class is above every ceiling) and `max_scenarios` (the hard
fan-out cap enforced by tanuki-drive).

## 0. Resolve and preflight

1. Read the scenario config; expand `~` in its `plugin`/`host` paths. `host`
   is **optional** — it exists only for plugins that operate on a repository's
   content (e.g. a docs generator that works over a host repo's files). A
   self-contained plugin omits it, and the driver fabricates an empty
   isolated workspace per scenario instead (omit `--host` from the
   tanuki-drive invocation). Generate a run id: `<YYYYMMDD>-<short-random>`.
2. Run `${CLAUDE_PLUGIN_ROOT}/tools/tanuki-preflight <plugin-repo>`. **If it fails, stop
   and report the failures** — mechanical violations are lint territory;
   dogfooding time is never spent rediscovering them. Do not
   "quickly fix" the violations yourself unless the user asks: report first.
3. `${CLAUDE_PLUGIN_ROOT}/tools/tanuki-ledger --target <target> init` (idempotent).

## 1. Drive (Driver — deterministic tool, cheap model inside)

**Scenario selection (deterministic — the adaptive scheduler).** Unless the
user named explicit scenario ids or gave ad-hoc text, the run's scenario set
comes from `tanuki-scheduler --target <t> plan --scenarios <file>` (sync
first if the matrix changed): verify set (accepted findings awaiting
absence) → exploration quota (unexplored branches) → active rotation → due
regression-pool members. Drive the planned subset via `--only`. A target
with no scheduler state yet behaves as before (first `sync` marks the whole
hand-written matrix active).

**Plan gate first (plan before fan-out).** Run
`tanuki-drive … --estimate` and show the user the plan: scenario ids, models,
turn caps, and the **expected duration** (`est_total_duration_s` when run
history exists — convert to minutes; never surface the USD fields), plus
**how it will execute**: as headless `claude` processes spawned by
`tanuki-drive` (not session subagents), normally in the background. Then,
unless the user already named explicit scenario ids in $ARGUMENTS (that *is*
the approval), ask with AskUserQuestion: proceed with all / trim to a subset
/ abort. Never start driving a full matrix silently. If the plan is over
`max_scenarios`, say so — approval here does not bypass the cap; only config
or `--allow-extra` does, and use `--allow-extra` solely when the user
explicitly chose the over-cap plan.

Then run, in the background if long:

```
${CLAUDE_PLUGIN_ROOT}/tools/tanuki-drive --target <target> \
  --plugin <plugin-repo> [--host <host-repo>] \
  --scenarios <config> --run <run-id> [--only s1,s2]
```

**Background liveness (a silent wait is a UX bug).** The drive writes
`<run-dir>/progress.json` — total/done, current scenario + stage, elapsed
seconds, KB captured, per-scenario results — atomically at every stage
transition and ~20s tick. While the drive runs in the background, NEVER end a
turn with a bare "waiting / I'll be notified": set a periodic check (a
monitor or scheduled wake, roughly every 2–3 minutes — not busy-polling) and
each time relay ONE compact line from progress.json, e.g.
`[2/5] draft-devlog: ok (34 events, 210s) — driving draft-survey (95s, 120KB)`.
Every waiting turn ends with the latest such line plus the expected total
duration and when the next update will come, so the user never has to ask
"are you still working?".

The tool clones both repos per scenario (never executes against the real
checkouts), drives each scenario headless on a cheaper model
(default `claude-sonnet-5`; a scenario config may set `"model"`
per scenario, e.g. `claude-haiku-4-5-20251001` for a sensitivity experiment —
never a frontier model, which would cope with the friction instead of
surfacing it), captures the raw stream, normalizes
it to Events, and records a post-run pollution check. While it runs, do
nothing else with the ledger. Read `manifest.json` when done; report any
scenario whose status is not `ok`, and any `plugin_clone_dirty: true`
(isolation violation — that is itself a finding).

## 2. Mine (Miner — cheap extraction subagent, then you for dedupe)

1. Ingest: `tanuki-ledger --target <target> ingest <run-dir>/*.events.jsonl`.
2. **Extraction (cheap subagent — not you).** Spawn one subagent at the
   highest tier the `model_ceiling` allows (default `sonnet`) whose task is:
   read every `*.events.jsonl` in
   `<run-dir>` (including FRICTION LOG `note` events; the raw stream only for
   genuinely ambiguous moments), and write `<run-dir>/candidates.json` — a
   JSON array of `{title, kind: friction|papercut|gap, evidence:
   ["run/scenario#seq", …], note?}`. Give the subagent these rules verbatim:
   - One execution yields **zero or more** candidates; never force one.
   - **Mechanical/spec violations are not findings**: if a run surfaced one
     (broken reference, schema violation), emit it under a separate
     `preflight_rule_candidates` key instead.
   - Simulator artifacts are not findings: friction caused by the simulated
     user being weak (misreading an instruction a real user would get) is
     dropped, with a note.
   - A candidate without at least one event pointer is not emitted.
   The subagent must not touch the ledger. If the run is tiny (≤2 scenarios
   or ≤15 events), skip the subagent and extract inline — the handoff would
   cost more than it saves.
3. **Semantic dedupe (you — never delegated).** Read `candidates.json`, then
   `tanuki-ledger --target <target> findings`, and for each candidate decide —
   same underlying problem as an existing finding →
   `upsert-finding --match <id> --evidence …` (bumps recurrence); genuinely
   new → `upsert-finding --title … --kind friction|papercut|gap --evidence …`.
   Recurrence integrity is the promotion signal; this judgment stays at the
   frontier tier. Spot-check any candidate whose evidence pointers look
   inconsistent before recording it — extraction is cheap-tier and may err.
   Every finding carries evidence pointers (`run/scenario#seq`) — a finding
   you cannot point at an event is not recorded (claim–evidence binding).
4. **Report the run delta** — the user must never have to remember prior
   runs. Print a compact table: bumped findings (id, recurrence
   before→after, one-line title) and new findings (id, kind, title). "Already
   known" is always accompanied by *which* finding and its new recurrence.
5. **Fold the run into the scheduler**: `tanuki-scheduler --target <t>
   record-run --run <run-id>` (idempotent) — yields update streaks; demotion
   (2 consecutive low-yield runs → regression pool), first-drive promotion,
   and finding-driven re-promotion are the tool's arithmetic, never yours.

## 3. Consolidate (Consolidator — you, then the human gate)

1. `tanuki-ledger --target <target> promote` moves findings past the bar
   (chronic ≥3 recurrences, or breadth ≥2 scenarios) to `proposed` and lists
   them; use `--dry-run` when you only need the listing. No separate
   `set-status` call is part of promotion.
2. **Policy advisory (opt-in — specs/spec-policy-advisory).** Run
   `tanuki-ledger --target <target> policy-surface`. `{"configured": false}`
   or `"available": false` → skip this step entirely. Otherwise judge each
   proposal against the emitted numbered lines — frontier judgment, yours,
   never delegated to a cheaper model and never reduced to code matching:
   where a proposal's problem or proposed fix tensions with a policy line,
   prepare one advisory line
   `policy: tension — <one clause> vs "<quote>" (<file>:<line>@<commit>)`.
   No tension → no line. The flags are advisory only: ranking is unaffected,
   and nothing from the policy surface may enter the ledger, events, or
   scheduler state. This step exists only here and at the gate (step 4) —
   never at ingest, drive, or extraction.
3. Write the brief to `~/.tanuki/<target>/briefs/<run-id>.md`:
   - **Delta this run** first: bumped vs new findings (from step 2.4).
   - **≤`brief_max_proposals` ranked proposals** (default 10; rank by
     recurrence, then breadth, then severity). Each item in this order:
     **Problem** (what a user hits, 1–3 sentences) → **Proposal** (the
     concrete spec-change or implementation sketch, issue-shaped, pasteable
     into `gh issue create` but **never auto-filed**) → **Evidence** (one
     compact line of pointers + recurrence, last — it exists for the rare
     dispute, not for reading). A step-2 `policy: tension` line, when one
     exists for the item, goes directly under its Evidence line.
   - **Watching**: one line per open finding below the bar, sorted by
     priority and prefixed `P1`/`P2`/`P3` (chronic-or-cross-scenario /
     recurrence-2-or-gap / rest — mirror `tanuki-ledger status`).
   - **Preflight-rule candidates**: mechanical items from the extraction
     pass's `preflight_rule_candidates` (step 2.2).
   - **Lesson candidates (for the den)**: transferable conclusions, proposed
     for your knowledge hub. They are FIRST-CLASS at the decision pass
     (story 1.35 / spec-den-distill §4): walked in 4.1c with
     accept-writes-the-staging-file semantics when `contribute_back` is
     configured. Writes go only to the configured hub staging directory —
     never anywhere else outside `~/.tanuki/`.
   - **consulted** (only when the brief carries any `policy: tension` line):
     one closing line naming which allowlisted policy files/lines were read
     and which applied — the audit trail for the advisory pass.

## 4. Decide (the human gate — part of the run, not homework)

This pass is reached two ways, and they are **one contract, not two
behaviors**: at the end of a normal run (below), and as `/tanuki <target>
decide` (alias `--brief`) with no run at all. Entry is **ledger-anchored**:
`decide` orients off `tanuki-ledger status` / `next` and decides whatever the
ledger holds decidable — a brief is reprinted as context when one exists,
never a precondition. If `next` reports accepted fixes awaiting
re-verification, say so and continue with what IS decidable now; this pass
decides and files, it never drives.

### 4.0a Promote (when entered directly)

A normal run has already promoted in step 3.1. Entered as `decide` there was
no step 3, so the pass promotes for itself — `tanuki-ledger --target <t>
promote` moves qualifying open findings past the bar to `proposed` and lists
them. This is the pass's own contract ("promotion moves status to `proposed`;
the command layer performs the transition — no separate unhinted `set-status`
call", spec-short-command-surface D1), and without it a `decide` sitting with
no recent run would present nothing and the ledger-anchored entry would be a
promise the pass doesn't keep.

### 4.0b Consolidate — before anything is presented

No finding reaches the approval screen unanalyzed. Filing an issue whose
conflict with another filed issue was detectable from the ledger is a defect
of this command, not of the operator's attention. Contract:
`${CLAUDE_PLUGIN_ROOT}/specs/spec-tanuki-solve/SPEC.md` D1/D3 (the normative
taxonomy) over D2's substrate.

**Skip when there is nothing to consolidate:** with fewer than two candidate
items (proposals + watching), go straight to 4.1 — a one-finding sitting
never runs a clustering stage it cannot need.

a. `tanuki-ledger --target <t> consolidate` — read-only deterministic
   candidate groups with a mechanical reason each. It proposes, never
   decides.
b. Judge each candidate group — confirm or discard, classify the confirmed
   ones, and **add any group the clustering missed** (semantic contradiction
   is not reliably mechanical; the tool proposes, this layer decides):
   - **merge** — same defect, different sightings. ONE presentation item:
     combined problem statement, union of evidence pointers and finding ids.
   - **conflict** — proposals that cannot both hold. ONE multi-outcome
     question naming each branch explicitly ("A: …; B: …; defer group") —
     **never split back into independent yes/no gates**.
   - **dependency** — one disposition reframes another. The reframing item
     is asked first; the dependent item's presentation states the dependency
     ("given the choice on F63, this proposal now reads…").
c. Build the presentation plan: conflict/dependency groups first (most
   recontextualizing first), then merges, then independents, each lane by the
   ledger's computed priority.

Consolidation is a **presentation and filing act only**: ledger entries are
never rewritten, merged, or deleted — groups are recorded through the
existing status vocabulary plus shared issue URLs, never history surgery.

### 4.1 Walk the plan

Do not end at "here is the report." Present each plan item in-session
(Problem → Proposal, evidence collapsed to a pointer line; include the item's
`policy: tension` line as context when step 3.2 produced one — the flag never
pre-selects a disposition, and a policy-motivated dismissal is recorded like
any other), walking **the consolidated plan, not the raw finding list**, with
AskUserQuestion, up to 4 per round, one disposition each — **never with a
pre-selected default**: a disposition is a multi-outcome decision, and every
`set-status` write and `gh` call in this pass is run by the command, never
typed by the user.

Per group kind:
- **merge group** → one disposition; on accept, every constituent finding id
  gets it (`set-status` per id) and the same issue URL.
- **conflict group** → one question, options = the branches (+ defer). The
  winning branch's finding is accepted; the loser dismissed, noting the
  arbitration via `upsert-finding --match <id> --note "superseded by
  <winner>: <branch>"` — or, if the human says both should be absorbed into
  one issue, treat as a merge with the winning proposal text.
- **independent** → accept / dismiss / defer as today.

Dispositions:
- **accept** → `set-status --id … --status accepted`; then offer — as a
  separate, explicit confirmation — to run the prepared `gh issue create` in
  the target repo. Accepted findings keep their recurrence tracking: the next
  run verifies the fix by their absence.
  **Labels (the only machine-readable contract Tanuki emits).** Every issue
  Tanuki files carries exactly one label: the kind `<prefix>:<kind>` where
  `<prefix>` is `issue_label_prefix` (config, default `tanuki`) and `<kind>`
  is the finding's kind (`friction`/`papercut`/`gap`). The prefix in the
  label name is the provenance marker — no bare `<prefix>` label is applied.
  Before filing, ensure the label exists in the target repo (`gh label list`,
  then `gh label create` if missing — color `d4a017`, description "Tanuki
  finding kind: <kind>"). The label goes on the `gh issue create` call via
  `--label`. Downstream tooling may key off these labels ("everything Tanuki
  filed" = the three kind labels enumerated); Tanuki itself never reads them
  back and knows nothing about what consumes them — it stops at the labeled
  issue.
  **Titles are for humans only.** The issue title is the finding's plain
  problem statement — no `[tanuki:F2]`-style prefixes, no machine-readable
  markers: provenance and kind already live in the labels, and a prefix a
  human can't act on is noise in every issue list. The finding id goes in
  the body footer instead ("Tanuki finding: F2 — ledger cross-reference"),
  informational only, never parsed by anything. For a **merge group** the
  footer lists EVERY constituent finding id.
  **Conflict evidence survives into the issue (spec-tanuki-solve D3).** For a
  resolved conflict group, the body also records the arbitration: the
  branches considered, the branch chosen, and the rejected alternative(s)
  with their finding ids — so a reader of the single filed issue sees that an
  alternative existed and was rejected, with no downstream tooling awareness.
- **dismiss** → `set-status --id … --status dismissed` (never resurfaces as
  new; still deduped against).
- **defer** → stays `proposed`; `tanuki-ledger status` and `/tanuki <target>
  status` keep it visible until decided.

After the proposals, offer the **Watching** list the same way: the promotion
bar gates surfacing, not permission, so the user may **accept any below-bar
finding right there** (same accept path — `set-status`, then the
separately-confirmed filing offer). Don't walk every watching item one by
one; present the list once and act only on the ones the user picks. A
watching item already inside a confirmed group was presented **with its
group** in 4.1, not here.

### 4.1c Lesson candidates (story 1.35 / spec-den-distill §4)

After the Watching offer, walk the **lesson candidates** the same way —
accept / dismiss / defer per item, never a pre-selected default. The pool:
the brief's "Lesson candidates (for the den)" section when a brief exists,
grounded against `tanuki-ledger --target <t> distill --list` (the
deterministic bar-clearing, not-yet-contributed enumeration — an item
outside that list is not emittable and says so).

- **accept** → run `tanuki-ledger --target <t> distill --id <F>` right then
  — exactly parallel to "accept → optionally file the issue right then".
  The RECEIPT is the tool's own output line (`contributed F… -> <path>`) —
  relay it verbatim, never a re-derived path.
- **dismiss** → `set-status --id … --status dismissed` (deduped-against, and
  `distill` will never emit it).
- **defer** → stays as-is; `status` keeps it visible until decided.
- `contribute_back` unset → the one-line not-configured notice (story 1.36)
  renders instead of a walk; nothing is written.

The unattended loop renders lesson candidates in its brief but NEVER walks
this gate and never emits — attended-gate work, the same class as issue
filing (spec-den-distill §1).

Close with: the run delta (when a run happened), dispositions taken per
finding (with the issue URL for each one filed), **contributed lesson
candidates with their written staging paths, counted alongside filed issues**
(story 1.35 — the operator ends the sitting knowing exactly what left Tanuki
and where it went; zero contributed renders as zero, never omitted when
candidates were walked), **groups reported as
groups** — merged ids together, conflict branches with the chosen one marked
— what remains `proposed`/`open`, the brief path when one exists, the derived
next step (`tanuki-ledger next`), and total duration/turns (never dollars).
The pass is complete only if the user never had to type a `tanuki-ledger` or
`gh` command themselves.

## Ingest mode (`ingest "<feedback>"`, alias `--ingest`) — human feedback is one more event source

Manual dogfooding feeds the *same* pipeline as a driven run; the only
difference is the event source. The hard UX rule (docs/tanuki-spec.md "Human
feedback ingest"): **the human never classifies.** "Is this an
Event or a Finding?" is internal storage vocabulary, never a question the
user answers. So this mode skips step 0's preflight and step 1's driving and
does exactly this:

1. **Record verbatim (the tool, unjudged).**
   `tanuki-ledger --target <target> ingest-note --text "<feedback>"` writes
   the feedback as one Event (`type: note, source: human`, run id
   `manual-<YYYYMMDD>`) into `events/manual-<date>/ingest.events.jsonl` and
   the ledger, and prints the run id + events-file path. Nothing is
   classified here — the note is stored exactly as the user phrased it.
2. **Mine it — extraction + dedupe (step 2, inline).** A single note is a
   tiny run, so skip the extraction subagent and extract inline (the step-2
   rule for ≤2 scenarios / ≤15 events): read the note, decide whether it
   yields a candidate finding at all (zero or more — never force one), its
   kind, and — the frontier judgment, never delegated — whether it **bumps
   an existing finding** (`upsert-finding --match <id>`; a manual re-hit of a
   known friction pushes it toward the chronic bar) or is **new**
   (`upsert-finding --title … --kind …`). Feedback phrased as a ready-made
   proposal still enters as an event with the proposed action attached; it
   does not skip dedupe or the gate.
3. **Report the delta (step 2.4).** Print the same compact bumped-vs-new
   table any run ends with — the user sees what their feedback changed
   without remembering prior runs.

The one-way flow is preserved end to end: a manual note is an event, and it
reaches a proposal only through the normal extraction → dedupe → promotion →
human-gate path. Promoted items surface in the next brief / `decide` exactly
like driven findings. Evidence pointers still bind every finding
(`manual-<date>/ingest#<seq>`).

Hard rules, restated because they are the product:
clones only; proposals only; one-way flow events → findings → proposals; the
decision pass ratifies — it never merges, and issue filing is its own
confirmation. **Tanuki ends at the labeled issue**: it knows nothing about
specs, stories, BMAD, or any implementation workflow downstream of the issue
tracker, and nothing downstream may require Tanuki internals — the labels are
the entire interface.
