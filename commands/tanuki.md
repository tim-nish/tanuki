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
  scenario** — see "Ad-hoc scenarios" below. Disambiguation: a non-flag
  argument matching a configured target is a target; one matching scenario
  id(s) selects them; anything else is ad-hoc text.
- `<target> --brief`: reprint the latest brief and resume the decision pass
  on anything still `proposed`. No driving.
- `<target> --status`: run `tanuki-ledger --target <target> status` and show
  it. No driving.
- `<target> --history [scenario]`: cross-run scenario monitoring — run
  `tanuki-scheduler --target <target> history --scenarios <matrix-file>
  [--scenario <id>]` and show it. Repo-wide: per-scenario
  state/executions/streaks/recurrence, unexplored, long-unrun, recently
  productive, the selection history with reasons — plus, when the matrix
  declares `axes`/`covers`, the **exploration coverage** block (per axis
  value: explored / authored-never-run / uncovered), the **exploration
  debt** summary, and ≤3 advisory next-charter recommendations (never
  auto-applied — new charters enter only through the plan-gated generation
  pass). With a scenario id: every execution (date, decision-point pins,
  yield, findings) plus its transition log. Built from persisted artifacts
  only (manifests, ledger, scheduler state, the matrix) — never model
  memory. No driving.
- `<target> --mine-only <run-id>`: skip driving; mine + consolidate an
  existing run's events (use after a crashed or interrupted session).
- `<target> --ingest "<feedback>"`: record human feedback in natural
  language (no driving) and mine it immediately — see "Ingest mode" below.
  The human never classifies; the tool stores the note verbatim and the
  normal extraction + dedupe passes decide everything downstream.

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
   (including at least one pinned decision-point branch and one error-path),
   **and declare the exploration space with them**: a top-level `axes` block
   ({axis: {values, note?}}) plus per-scenario `covers` tags ({axis:
   [values]}) — declaring the branching space is part of the same frontier
   judgment, and coverage/debt reporting is uncomputable without it. Tools
   never invent or extend axes; they compute over what this gate ratifies.
   Present everything at one plan gate, apply the user's edits, then write
   `~/.tanuki/scenarios/<slug>.scenarios.json` and run
   `tanuki-scheduler --target <slug> sync --scenarios <file>`. On any
   regeneration pass, update `axes`/`covers` the same way.
5. Offer the `"loop"` block (derive a `test_cmd` per /tanuki-loop's rule).

Manual alternative (still supported): copy an existing scenarios file (or the
bundled `${CLAUDE_PLUGIN_ROOT}/templates/example.scenarios.json`) to
`~/.tanuki/scenarios/<new-target>.scenarios.json` and edit `plugin`/`host`/
scenarios. Offer init when the user names a target that has no config yet.
Regenerate more charters (same gate) whenever the unexplored pool empties.

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

Distinguish from `--ingest`: **`--ingest` reports the past** (friction the
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
     for your knowledge hub — a human moves them; this command never writes
     outside `~/.tanuki/`.
   - **consulted** (only when the brief carries any `policy: tension` line):
     one closing line naming which allowlisted policy files/lines were read
     and which applied — the audit trail for the advisory pass.

## 4. Decide (the human gate — part of the run, not homework)

Do not end at "here is the report." Present the brief's proposals in-session
(Problem → Proposal, evidence collapsed to a pointer line; include the item's
`policy: tension` line as context when step 3.2 produced one — the flag never
pre-selects a disposition, and a policy-motivated dismissal is recorded like
any other), then walk them top-down with AskUserQuestion, up to 4 per round,
one disposition each — **never with a pre-selected default**: a disposition
is a multi-outcome decision, and every `set-status` write and `gh` call in
this pass is run by the command, never typed by the user:
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
  informational only, never parsed by anything.
- **dismiss** → `set-status --id … --status dismissed` (never resurfaces as
  new; still deduped against).
- **defer** → stays `proposed`; `tanuki-ledger status` and `/tanuki <target>
  --status` keep it visible until decided.

After the proposals, offer the brief's **Watching** list the same way: the
promotion bar gates surfacing, not permission, so the user may **accept any
below-bar finding right there** (same accept path — `set-status`, then the
separately-confirmed filing offer). Don't walk every watching item one by
one; present the list once and act only on the ones the user picks.

Close with: the run delta, dispositions taken per finding (with the issue
URL for each one filed), what remains `proposed`, the brief path, and total
duration/turns (never dollars). The pass is complete only if the user never
had to type a `tanuki-ledger` or `gh` command themselves.

## Ingest mode (`--ingest "<feedback>"`) — human feedback is one more event source

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
human-gate path. Promoted items surface in the next brief / `--brief` exactly
like driven findings. Evidence pointers still bind every finding
(`manual-<date>/ingest#<seq>`).

Hard rules, restated because they are the product:
clones only; proposals only; one-way flow events → findings → proposals; the
decision pass ratifies — it never merges, and issue filing is its own
confirmation. **Tanuki ends at the labeled issue**: it knows nothing about
specs, stories, BMAD, or any implementation workflow downstream of the issue
tracker, and nothing downstream may require Tanuki internals — the labels are
the entire interface.
