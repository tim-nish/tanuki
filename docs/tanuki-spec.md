# Tanuki — automated dogfooding prototype (spec)

Status: prototype v0 (2026-07-13).
Goal: reduce the manual work of dogfooding a Claude Code plugin — explore
scenarios, collect events, derive findings, consolidate actionable proposals.
This is a working vertical slice, not a framework.

## Standing constraints (inherited, non-negotiable)

- **Three roles over one shared ledger**: the Driver *drives*, the
  Miner *mines*, the Consolidator *consolidates*. "Tanuki" names the umbrella;
  the seams stay.
- **Vocabulary**: **Event** = raw, factual, per-run record, never
  judged, 0..n per run. **Finding** = judged, deduped, ledger-tracked signal
  with recurrence count; chronic at 3 occurrences. **Proposal** = a finding
  promoted through the human gate. No "Observations". Information flows one
  way: events → findings → proposals.
- **Mechanical violations are lint territory**: a preflight stage
  catches them deterministically before any scenario runs; a mechanical
  violation surfacing *during* a run proposes a new preflight rule, not a UX
  finding.
- **Proposals-only**: nothing merges, nothing writes to the
  target repo, findings never auto-update specs. The output is a capped ranked
  brief plus proposal drafts for human review.
- **Isolation = run against clones, never the real repos**; verify
  pollution after every run (snapshot-diff-discard).
- **Model routing** (refined after early prototype runs): route
  by determinism first, then by judgment density — see the tier table below.
  A cheaper driver is not just cheaper, it is a **more sensitive instrument**:
  real users include weak agents, and a frontier driver quietly works around
  the very friction Tanuki exists to detect (an early finding surfaced only
  because the Sonnet driver churned where a frontier model would have coped).
- **Deterministic tools do deterministic work**: workspace prep,
  event capture/normalization, ledger mechanics, recurrence counting, and
  promotion thresholds are scripts; the LLM only judges (friction extraction,
  semantic dedupe, proposal writing).

## Model tiers (per stage — the routing table)

| Stage | Tier | Why |
|---|---|---|
| Charter/matrix design | **frontier or human** | Encodes the theory of the branching space (same rule as grid dimensions) |
| Scenario execution | **cheap (Sonnet default; Haiku experimental)** | High volume; weak-agent fidelity is a feature — friction must surface, not be coped with |
| Event normalization | **code, no model** | `tanuki-drive` parses the stream mechanically |
| Friction extraction | **cheap (Sonnet subagent)** | Reads the whole event volume (the token-heavy half of mining); "list the frictions" is near-mechanical |
| Semantic dedupe (bump vs new) | **frontier — never downgrade** | The load-bearing judgment: recurrence integrity drives promotion; wrong merges manufacture false chronics, wrong splits starve promotion (cheap in tokens, so downgrading saves ~nothing) |
| Promotion decision | **code, no model** | Thresholds in `tanuki-ledger promote`; no model may decide what is chronic |
| Proposal writing / brief | **frontier** | Architectural judgment, ranking, lesson candidates |

Downgrade order if cost ever forces a choice: execution model first (Sonnet→
Haiku per scenario), extraction second; dedupe and the brief never. Attribute
per-stage cost from run manifests before tuning further (measure, then optimize).

**Model ceiling.** `model_ceiling` (config, default `sonnet`) is the highest
tier Tanuki may *launch*: it governs the driver's headless runs and the
extraction subagent. Tiers: `haiku` < `sonnet` < `opus`; Fable/Mythos-class
models are above every ceiling value and can never be launched by Tanuki.
Unknown model names are treated as above the ceiling (fail closed). The
ceiling does NOT govern the orchestrating session (dedupe + brief) — that
model is chosen by the user when they start the session, and running /tanuki
from a cheaper session is the way to cap it.

## Configuration

Global file `~/.tanuki/config.json` (all keys optional); a per-target
`"defaults": {…}` block in `~/.tanuki/scenarios/<target>.scenarios.json`
overrides it; CLI flags override both. Built-in defaults apply when no file
exists — Tanuki runs with zero configuration.

| key | default | used by | meaning |
|---|---|---|---|
| `driver_model` | `claude-sonnet-5` | drive | model for scenario execution |
| `model_ceiling` | `sonnet` | drive (+ command, for extraction) | highest tier Tanuki may launch |
| `max_scenarios` | `6` | drive | hard cap per invocation; exceeding it requires `--allow-extra` (the fan-out cap) |
| `max_turns` | `40` | drive | per-scenario turn cap (scenario `max_turns` overrides) |
| `timeout_s` | `900` | drive | per-scenario wall-clock cap |
| `est_cost_per_scenario_usd` | `1.5` | drive `--estimate` | fallback when no run history exists |
| `min_recurrence` | `3` | ledger promote | chronic threshold |
| `min_scenarios` | `2` | ledger promote | breadth threshold |
| `compaction_unseen_runs` | `3` | ledger compact | runs of absence before a dismissed / verified-fixed finding tombstones |
| `demote_after` | `2` | scheduler | consecutive low-yield runs before a scenario demotes to the regression pool (hysteresis) |
| `regression_every` | `3` | scheduler | undriven runs before a regression-pool scenario is due again (never deleted) |
| `exploration_quota` | `1` | scheduler | unexplored scenarios each plan must include while any exist |
| `low_yield_threshold` | `0` | scheduler | actionable findings at or below which a run counts as low-yield |
| `brief_max_proposals` | `10` | command (consolidate) | ranked-proposal cap in the brief |
| `issue_label_prefix` | `tanuki` | command (decide) | label namespace on filed issues: one kind label `<prefix>:<kind>` |
| `policy_source` | unset | ledger policy-surface → command (consolidate + decide only) | opt-in advisory block `{path, files: [≤4 .md]}`: a local policy checkout read read-only and pinned at the brief/gate; never consulted at ingest/drive/extraction, never blocking (specs/spec-policy-advisory) |

**Plan gate (scenario count + cost).** The user never has to pre-compute a
scenario budget: `tanuki-drive --estimate` prints the plan — scenario ids,
models, and an estimated total cost (mean per-scenario cost from prior run
manifests when available, else `est_cost_per_scenario_usd`) — without driving
anything. Attended `/tanuki` surfaces this estimate and lets the user trim or
approve before execution; `/tanuki-loop` drives its scheduler-chosen set
without that approval step (§"The plan gate confirms execution", loop exempt).
`max_scenarios` remains the hard backstop regardless of approval. Manifests record actual per-scenario cost
(`cost_usd`), so estimates improve with every run.

## Prototype deviations (explicit, to revisit before any generalization)

- **No container / network-egress policy.** Isolation is clone + disposable
  workspace + post-run `git status` pollution check only. Acceptable because
  the target plugin is our own and read-mostly; not acceptable for third-party
  targets.
- **Ledger lives in `~/.tanuki/<target>/`**, not the den. Consolidator output
  *proposes* lesson candidates for the user's knowledge hub; a human moves them.
- **Scenario matrix is hand-written JSON** per target (no generation).
- **Single target shape**: a Claude Code plugin exercised in a host repo via
  `claude --plugin-dir` headless runs.

## Components

All tools are zero-dependency Python 3 in `tools/`,
orchestrated by `commands/tanuki.md`.

### 0. Preflight — `tools/tanuki-preflight <plugin-repo>`

Deterministic lint of the plugin repo. Checks (v0): every `skills/*/SKILL.md`
exists and has parseable frontmatter with `name` + `description`; relative
links in README/skills resolve; `scripts/*.sh` are executable and LF-only;
config examples exist where README references them. Output: pass/fail lines,
non-zero exit on failure. **The pipeline refuses to drive scenarios while
preflight fails** — dogfooding time is never spent rediscovering lint facts.

### 1. Driver — `tools/tanuki-drive`

Input: a run id, a scenario config (JSON), a path to the plugin repo, and —
only when the plugin operates on a repository's content — a host repo. Per
scenario, it:

1. Prepares a disposable workspace under `~/.tanuki/<target>/ws/<run>/<scenario>/`:
   `git clone` of the plugin repo and of the host repo when one is configured
   (local clones — the real repos are never executed against). A
   **self-contained plugin has no host**: the scenario then runs in a
   fabricated empty git-init'd workspace — still isolated, still
   pollution-checked, still never the user's real checkout.
2. Runs the scenario headless:
   `claude -p "<scenario prompt>" --plugin-dir <plugin-clone> --model <cheap>`
   (default `driver_model` from config; a scenario may override with a
   `"model"` field — e.g. a Haiku sensitivity experiment — never above the
   `model_ceiling`)
   with `--output-format stream-json` captured to
   `events/<run>/<scenario>.raw.jsonl`, a max-turn cap, and permissions
   confined to the workspace (prototype: `--dangerously-skip-permissions`
   inside the disposable clone — see deviations).
3. The scenario prompt wraps a **charter** (exploratory-testing style): the simulated user's goal, persona, and the branch being
   explored. Charter dimensions cover both *external* setup (host repo ×
   skill flow × article purpose × entry state) and **intra-command decision
   points**: the choices a single command presents during its
   flow — framework selection, review depth, visuals, any AskUserQuestion
   fork — are first-class grid dimensions. A scenario may pin a decision
   point ("at the framework question, choose X") so two charters can run the
   *same command* down *different internal branches*; repeated runs of one
   command are only redundant when they also repeat its internal path.
4. Normalizes the raw stream into **Events**
   (`events/<run>/<scenario>.events.jsonl`): one JSON object per event —
   `{run, scenario, seq, type, detail, evidence}` where `type ∈ {tool_error,
   permission_block, retry, user_choice, skill_start, skill_end, result,
   budget, note}`. Normalization is mechanical (parsed from the stream), never
   judged.
5. Verifies isolation: `git -C <clone> status --porcelain` on both clones is
   recorded into the run manifest; unexpected dirt in the plugin clone is
   itself an event (`type: note, detail: pollution`).

One run produces **zero or more** events. Repetition of an unchanged scenario
is only for chronic-vs-flaky; the matrix prefers breadth.

The driver is never silent: it prints the current stage per scenario
(`clone + setup` → `scenario running` → `normalize + verify isolation`) and,
while a scenario runs, a liveness line every ~20s with elapsed time and
captured transcript size. Completion lines report events, turns, and duration
(not dollars).

### 2. Miner — `tools/tanuki-ledger` + a cheap extraction pass + frontier dedupe

The ledger (`~/.tanuki/<target>/ledger.json`) is the single shared store.
Subcommands (deterministic): `init`, `ingest <events.jsonl>` (records events,
exact-dupe collapse by fingerprint), `findings` (list, with recurrence counts
and event pointers), `upsert-finding` (create or bump: `--match <id>` bumps
recurrence and appends evidence; no match creates a new finding with a fresh
id), `promote --min-recurrence N` (list chronic/high-confidence candidates),
`set-status`, `stats`, and `status` (the human-readable "where was I?" view:
runs, latest brief, findings awaiting decision with P1–P3 priorities,
accepted-awaiting-verification, top watching items).

**Finding lifecycle** (the decision states are first-class — Tanuki is
decision support, not reporting): `open` (below the promotion bar) →
`proposed` (in a brief, awaiting the human's decision) → `accepted` (human
said yes — issue filed or will-fix; recurrence still tracked so a later run
*verifies the fix landed* when it stops recurring) or `dismissed` (human said
no; kept for dedupe so it never resurfaces as new).

**Priority** is deterministic display logic, not judgment: P1 = chronic
(recurrence ≥3) or cross-scenario; P2 = recurrence 2 or kind `gap`; P3 =
the rest.

The judgment half runs in two passes at different tiers:

1. **Extraction (cheap — Sonnet subagent).** A subagent reads the run's
   normalized events (the token-heavy input) and writes **candidate findings**
   to `events/<run>/candidates.json`: `[{title, kind, evidence: ["run/
   scenario#seq", …], note?}, …]`. UX friction only — mechanical facts route to
   "propose a preflight rule"; simulator artifacts are dropped with a note.
   The subagent never touches the ledger.
2. **Semantic dedupe (frontier — the orchestrating session).** For each
   candidate, call `findings` to see what exists and decide *bump existing id*
   vs *new finding* (`upsert-finding`). This judgment owns recurrence
   integrity — the promotion signal — and is never downgraded. The dedupe
   call is the LLM's; the arithmetic is the tool's.

**Human feedback ingest.** Manual dogfooding is one more event
source, with one hard UX rule: **the human never classifies**. Feedback is
handed to Tanuki as free-form natural language (in-session, or
`/tanuki <target> --ingest "<feedback>"`); Tanuki records it *verbatim* as an
event (`type: note, source: human`, run id `manual-<date>`) — mechanical,
unjudged — and the normal extraction + frontier-dedupe passes decide
everything downstream: whether it yields a finding, its kind, and whether it
bumps an existing finding's recurrence (a manual re-hit of a known friction
pushes it toward the chronic bar) or creates a new one. Event-vs-finding is
internal storage vocabulary, never a question the user answers. The one-way
flow is preserved: a manual note cannot skip dedupe or the gate — even
feedback phrased as a ready-made proposal enters as an event with the
proposed action attached, and surfaces through the normal brief.

Every finding is **pointed**: it carries `evidence: [run/scenario#seq, …]`
(claim–evidence binding).

Finding shape: `{id, title, kind: friction|papercut|gap, first_seen,
recurrence, evidence[], status: open|proposed|accepted|dismissed}`.

**Ledger growth policy.**
`ledger.json` is an internal state file — deduplicated significant events
(skill start/end, tool errors, results, cost) with per-run occurrence
tracking plus findings, each holding pointers back to `raw.jsonl` for
evidence. Indefinite growth is acceptable *only* under three invariants:
1. **Read-path invariant:** no LLM stage ever loads `ledger.json` whole.
   All model-facing access goes through `tanuki-ledger` subcommand output
   (`findings`, `status`, `stats`, `promote`) — bounded views, not the file.
   Token cost is governed by the views, so file size is a disk concern only.
2. **Capped evidence:** an event or finding record keeps `first_seen`,
   `last_seen`, occurrence count, and at most a handful of exemplar
   `raw.jsonl` pointers — never one pointer per occurrence. The raw logs
   remain the complete archive; the ledger stores enough to *find* evidence,
   not to *be* it.
3. **Compaction of the dead tail:** records tied only to `dismissed` or
   verified-fixed findings and unseen for N runs compact to one-line
   tombstones (id, resolution, count) — kept for dedupe so nothing
   resurfaces as new, but stripped of per-run detail. Regression analysis
   over compacted history goes to `raw.jsonl`.

After dedupe, the Miner reports the **run delta** to the user: which findings
were bumped (id, recurrence before→after) and which are new — the human never
has to remember prior runs to know what "already known" means.

### 3. Consolidator — `tools/tanuki-ledger promote` + one frontier-model pass + the decision pass

Reads promotion candidates (chronic ≥3, or high-confidence: severe + ≥2
scenarios) and writes the run's **brief**:
`~/.tanuki/<target>/briefs/<run>.md` — capped at `brief_max_proposals` ranked
items. **Item order within each proposal: Problem → Proposal → Evidence** —
the problem statement and the concrete fix are what the reader decides on;
evidence pointers come last, compact, for the rare dispute (claim–evidence
binding is unchanged — every item still carries pointers; they're just not
the headline). Proposals are issue-shaped, ready to paste into
`gh issue create` — but never auto-filed. Findings below the bar are listed
one-line under "watching", **sorted and prefixed with their P1–P3 priority**.
A "delta this run" line-set states what's new vs bumped. Lesson-shaped
conclusions are listed under "lesson candidates (for the den)" as proposals
for the user's knowledge hub. Ledger status of promoted findings moves to `proposed`.

**The decision pass (the human gate is part of the run, not homework).**
A run does not end with a report. After presenting the brief in-session, the
command walks the promoted proposals (top-down, small batches) and asks for a
disposition per item: **accept** (optionally file the issue right then —
still explicitly confirmed), **dismiss**, or **defer** (stays `proposed`).
Dispositions are written back via `set-status`. `tanuki-ledger status` shows
anything left undecided, so a deferred decision is never lost, only visible.

**Issue labels — the downstream boundary.** A filed issue carries exactly one
label, created in the target repo if absent: the kind `<prefix>:<kind>` where
`<prefix>` is `issue_label_prefix` (default `tanuki`) — i.e. `tanuki:friction`,
`tanuki:papercut`, or `tanuki:gap`. The prefix in the label name IS the
provenance marker; no bare `<prefix>` marker label is applied (one label per
issue keeps an open-source label list clean). The kind set is closed (the
Finding vocabulary), so "everything Tanuki filed" is the enumeration of the
kind labels — e.g. `label:"tanuki:friction","tanuki:papercut","tanuki:gap"` in
issue search. This label is the *only* machine-readable
contract Tanuki emits: the pipeline ends at the labeled issue. The channel
split is strict — **labels carry the machine-readable identity (provenance,
kind); the title carries only the human-readable problem statement**, with no
prefixes or markers. The finding id appears in the issue body footer as an
informational ledger cross-reference, never parsed by anything. Tanuki knows
nothing about what happens downstream (spec workflows, story systems,
implementation commands), and downstream tooling may rely only on the labels
plus the issue's free-text body — never on Tanuki's brief format, ledger, or
directory layout. (The one-way flow extends by one hop: events → findings →
proposals → labeled issues.) Fix verification needs no back-channel either:
an accepted finding is verified by its *absence* in later runs, regardless of
who implemented the fix or how.

## Command — `/tanuki [target] [scenarios… | --brief | --status | --mine-only <run> | --ingest "<feedback>"]`

`commands/tanuki.md` orchestrates: resolve target config
(`~/.tanuki/scenarios/<target>.scenarios.json`) → preflight (stop on failure) →
plan gate → drive (subset of scenarios if named) → mine → consolidate →
decision pass. UX rules, from dogfooding Tanuki itself:

- **No-argument invocation is a picker, not an error**: `/tanuki` lists the
  available targets (every `~/.tanuki/scenarios/*.scenarios.json`) with each
  target's one-line `status` summary, and asks which to run
  (AskUserQuestion — selection beats typing names without completion).
  Adding a target = copying an existing scenarios file; no config edit is
  needed to switch targets.
- **The plan gate confirms execution before anything runs** (attended
  `/tanuki` only): scenario list, models, expected duration (from run history)
  and turn caps, and *how* it will execute (headless `claude` processes via
  `tanuki-drive`, normally in the background). Nothing drives until the user
  answers. **`/tanuki-loop` is exempt**: its scenario set is chosen
  deterministically by `tanuki-scheduler plan`, so that set is pre-approved and
  the loop drives immediately with no execution-confirmation gate — surfacing
  one would block unattended overnight runs (see `specs/spec-tanuki-loop/SPEC.md`).
- **Cost is displayed as time and turns, never dollars.** USD figures stay in
  manifests as estimate history; user-facing surfaces (plan gate, progress,
  brief) show duration and turn counts.
- **`/tanuki <target> --brief`** reprints the latest brief;
  **`/tanuki <target> --status`** runs the ledger's `status` view. Both are
  read-only re-entry points into an unfinished human gate.
- **`/tanuki <target> --ingest "<feedback>"`** records human feedback in
  natural language (see "Human feedback ingest") and runs extraction + dedupe
  on it immediately, reporting the delta (bumped vs new) like any run.
- **All intermediates live under `~/.tanuki/<target>/events/<run>/`** —
  never the session temp dir, never the target repos. The brief's canonical
  home stays `~/.tanuki` (writing it into a repo would pollute it); the
  in-session presentation + `--brief` are the access path.

## Files

```
tanuki/ (the plugin repo — installed root is ${CLAUDE_PLUGIN_ROOT})
  .claude-plugin/plugin.json                  # plugin manifest
  commands/tanuki.md                          # the /tanuki command
  commands/tanuki-loop.md                     # the /tanuki-loop command
  tools/tanuki-preflight                      # 0: mechanical lint
  tools/tanuki-drive                          # 1: Driver
  tools/tanuki-ledger                         # 2/3: ledger substrate
  tools/tanuki-scheduler                      # scenario scheduling + registry
  tools/tanuki-loop                           # loop safety substrate
  templates/example.scenarios.json            # copy to ~/.tanuki/scenarios/
  templates/config.example.json               # copy to ~/.tanuki/config.json
  docs/tanuki-spec.md                         # this file
~/.tanuki/scenarios/<target>.scenarios.json   # per-target scenario matrix
~/.tanuki/<target>/
  ws/<run>/<scenario>/{host,plugin}           # disposable clones
  events/<run>/<scenario>.{raw,events}.jsonl  # Events
  ledger.json                                 # Findings
  briefs/<run>.md                             # Proposals (human gate)
```

**Canonical scenarios location.** `~/.tanuki/scenarios/<target>.scenarios.json`
is the **single supported home** for a target's matrix — it sits next to the
rest of that target's state. No other path is read (an earlier ad-hoc
`~/.claude/templates/tanuki/` location is not supported and cost a config that
was nearly lost). The loaders (`tanuki-drive`, `tanuki-loop init`,
`tanuki-scheduler`) fail closed with this canonical path when the config is
absent, so a misplaced file surfaces immediately instead of a stale copy being
read silently.

## Out of scope for the prototype

Cross-repo consolidation beyond one target, transcript mining of real
sessions (dogfood-digest's job), containerized isolation, auto-filed issues,
any write to the target repos, scenario generation.
