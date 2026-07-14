# Tanuki 🦝

Automated dogfooding for Claude Code plugins. Tanuki drives a plugin through
branching user scenarios in disposable clones, mines the recorded **Events**
into deduplicated **Findings** with recurrence tracking, and consolidates the
chronic ones into a short, ranked brief of **Proposals** for human review.

**Proposals-only, always:** nothing merges, nothing is auto-filed, nothing
writes to the target repos, findings never auto-update specs. The human gate
is everything after the brief.

Spec: [docs/tanuki-spec.md](docs/tanuki-spec.md).

## Install

Tanuki is a Claude Code plugin; this repository is its marketplace. In
Claude Code:

```
/plugin marketplace add tim-nish/tanuki
/plugin install tanuki@tanuki
```

Requirements: `claude` CLI on PATH, `git`, Python 3. No other dependencies.
Zero configuration needed — see [Configuration](#configuration) to tune.

## Quickstart

```
cd ~/work/<plugin> && /tanuki init            # onboard: register + GENERATED scenario matrix
/tanuki                                       # target from cwd registry, else picker
/tanuki "try drafting an article about X"     # ad-hoc scenario (full pipeline, one-off)
/tanuki my-plugin                             # scheduler-planned run (plan gate asks first)
/tanuki my-plugin first-run                   # named scenarios = pre-approved
/tanuki my-plugin --ingest "…"                # log YOUR OWN dogfooding feedback (see below)
/tanuki my-plugin --status                    # where was I? (pending decisions)
/tanuki my-plugin --brief                     # reopen the latest brief + resume deciding
/tanuki my-plugin --mine-only <run>           # re-mine an existing run
/tanuki-loop                                  # unattended loop, zero-config (see below)
```

## How it works

```
preflight (lint, code)          mechanical violations stop here — never
        │                       rediscovered by dogfooding
        ▼
plan gate (you approve)         scenarios × est. cost, from run history
        ▼
DRIVER  tanuki-drive            per scenario: fresh clones of plugin+host →
  (cheap model, e.g. Sonnet)    headless claude run under a charter →
        │                       Events (normalized mechanically) →
        ▼                       post-run pollution check
MINER   extraction subagent     events → candidate findings (cheap model)
        + frontier dedupe       candidates → ledger upserts: bump recurrence
        │                       vs create (frontier judgment, never delegated)
        ▼
CONSOLIDATOR                    promotion by thresholds (code), then the
  (frontier + code)             brief: ≤10 ranked proposals, watching list,
        │                       lesson candidates
        ▼
DECISION PASS (in-session)      each proposal: accept / dismiss / defer —
                                written back to the ledger; a run ends in
                                decisions, not a report
```

A weak driver model is deliberate: real users include weak agents, and a
frontier model quietly copes with exactly the friction Tanuki exists to
surface.

## Logging your own dogfooding (`--ingest`)

Manual dogfooding is one more event source — hand Tanuki your observation in
plain language and it flows through the exact same pipeline as a driven run:

```
/tanuki my-plugin --ingest "The README says to look for the fact-sheet, but
outside the Claude Code UI I'd like to eliminate every remaining Human Gate."
```

The one hard rule: **you never classify.** Don't decide whether it's an Event
or a Finding, a bug or a papercut — that's internal vocabulary. Tanuki records
your words *verbatim* as an event (`type: note, source: human`, run id
`manual-<date>`), then immediately runs the normal extraction + frontier
dedupe: if your feedback is semantically an already-known friction, that
finding's **recurrence bumps** (pushing it toward the chronic bar — a manual
re-hit counts like any other); if it's new, a new finding is created. Either
way you get the same delta report a driven run ends with (bumped vs new).

Feedback phrased as a ready-made proposal doesn't skip the line: it enters as
an event with the proposed action attached and surfaces through the normal
brief and decision pass. No driving happens, no preflight — `--ingest` is
cheap and instant; use it the moment friction bites, as often as you like.

## Files

In this plugin (`${CLAUDE_PLUGIN_ROOT}` once installed):

| path | role |
|---|---|
| `commands/tanuki.md` | the `/tanuki` command (orchestration, incl. `--ingest`) |
| `commands/tanuki-loop.md` | `/tanuki-loop`: unattended cumulative dogfooding ([own spec](specs/spec-tanuki-loop/SPEC.md)) |
| `tools/tanuki-preflight` | mechanical lint of the plugin repo |
| `tools/tanuki-drive` | Driver: isolation, headless runs, Event capture, live progress |
| `tools/tanuki-ledger` | Findings ledger: ingest, ingest-note, dedupe support, promote, compact, scenario-yield |
| `tools/tanuki-scheduler` | adaptive scenario scheduling + repo→target registry ([spec](specs/spec-tanuki-scenario-lifecycle/SPEC.md)) |
| `tools/tanuki-loop` | deterministic safety substrate for `/tanuki-loop` |
| `templates/example.scenarios.json` | starting point for a per-target scenario matrix |
| `templates/config.example.json` | all config keys + defaults |

Your per-target scenario matrices live at
`~/.tanuki/scenarios/<target>.scenarios.json` — `/tanuki init` generates them;
Tanuki never writes configuration into the target repository.

Generated per run under `~/.tanuki/<target>/`:

| path | what | you do |
|---|---|---|
| `events/<run>/manifest.json` | statuses, models, costs, pollution checks | glance: non-`ok` or `plugin_clone_dirty` needs attention |
| `events/<run>/progress.json` | live drive status (done/total, current scenario + stage, elapsed) | `cat` it anytime a backgrounded drive feels quiet |
| `events/manual-<date>/ingest.events.jsonl` | your `--ingest` feedback, verbatim | nothing — evidence like any other event |
| `events/<run>/<scenario>.raw.jsonl` | full transcript | audit trail only |
| `events/<run>/<scenario>.events.jsonl` | normalized Events | nothing — evidence your findings point at |
| `events/<run>/candidates.json` | extracted candidates | nothing — intermediate |
| `ledger.json` | persistent Findings (recurrence, evidence, status) | never hand-edit; use `tanuki-ledger set-status` |
| `ws/<run>/…` | disposable clones + scenario artifacts | inspect if curious, delete freely |
| `briefs/<run>.md` | **the deliverable**: run delta, ranked Proposals (Problem → Proposal → Evidence), prioritized watching list | reviewed in-session at the decision pass; reopen anytime with `--brief` |

## The unattended loop (`/tanuki-loop`)

Overnight cumulative dogfooding: **drive → mine → classify → implement →
test → commit**, repeated on a dedicated integration branch in an isolated
worktree, until **two consecutive quiet cycles** (convergence) or a fixed
iteration cap. The Human Gate is *relocated, not removed* — the loop
prepares, you ratify once in the morning; `integration → main` is never
merged unattended, in any phase. Full contract:
[spec-tanuki-loop](specs/spec-tanuki-loop/SPEC.md).

```
/tanuki-loop                        # target picker; config from the scenarios file
/tanuki-loop my-plugin              # zero flags — settings stored once (see below)
/tanuki-loop my-plugin --iterations 2                # per-run override (commissioning)
<plugin-root>/tools/tanuki-loop --target my-plugin dashboard --follow 10   # watch it
```

Per-target settings live in the scenarios file's `"loop"` block, written once:

```json
"loop": {
  "test_cmd": "<the repo's regression gate, run every iteration>",
  "iterations": 5, "wall_time_s": 21600, "token_budget": 2000000
}
```

`test_cmd` is the deterministic regression gate — exit 0 lets the iteration
commit; non-zero is an immediate-stop breaker. Exclude checks that already
fail on the base branch (baseline, not regressions). On a target's first loop
run without a `"loop"` block, the command derives a candidate `test_cmd`,
confirms it with you, and offers to save the block.

While it runs: the **dashboard** (above) answers what ran recently, what is
running now (live scenario + stage), findings discovered/unresolved,
deferred/frozen items, why the loop stopped, and what runs next. Each
iteration after the first **rotates charters** — regression scenarios are
kept unchanged (their silence verifies landed fixes) while `decision_points`
are varied to internal branches the run hasn't walked.

The **morning gate**: review the integration diff + the deferred-judgment
queue + the audit, then (merge-first, idempotent) final tests → you merge
`integration → main` → issues are materialized **one per resolved problem**,
describing what landed, linked to the merge, closed as completed. Spec-class
decisions are never made unattended — they wait in the queue for an attended
triage sitting.

## The human gate (the decision pass)

The run itself walks you through the promoted proposals — each shown as
**Problem → Proposal** (evidence pointers last; you rarely need them) — and
records one disposition per item: **accept** (optionally filing the prepared
`gh issue create` after a separate confirmation — filed issues carry the
`tanuki` marker label plus a `tanuki:<kind>` label, the only machine-readable
contract Tanuki emits; the pipeline ends at the labeled issue), **dismiss**
(never resurfaces as new), or **defer** (stays pending). Deferred decisions
are never lost: `/tanuki <target> --status` shows everything awaiting you,
with P1–P3 priorities, and `--brief` reopens the latest brief to resume
deciding. Lesson candidates you move by hand into your own knowledge hub —
Tanuki never writes there. After fixes land, **run Tanuki again**: accepted
findings that stop recurring are your regression check; unfixed ones climb
toward chronic. Finding lifecycle: `open → proposed → accepted | dismissed`.

## Configuration

`~/.tanuki/config.json` (all keys optional) < per-target `"defaults"` block in
the scenarios file < CLI flags. Defaults, in brief: `driver_model`
claude-sonnet-5 · `model_ceiling` sonnet (highest tier Tanuki may launch;
haiku < sonnet < opus, Fable/Mythos-class above every ceiling, unknown names
refused) · `max_scenarios` 6 (hard fan-out cap) · `max_turns` 40 ·
`timeout_s` 900 · `est_cost_per_scenario_usd` 1.5 · `min_recurrence` 3 ·
`min_scenarios` 2 · `compaction_unseen_runs` 3 · `brief_max_proposals` 10 ·
`issue_label_prefix` tanuki. Full table:
[docs/tanuki-spec.md](docs/tanuki-spec.md#configuration).

## Adding a target

Targets are resolved by config file, **never by your current directory** —
`cd`-ing into a repo does nothing (the registry only maps a cwd to a target
as a hint). To dogfood a new repo, `cd` into it and run `/tanuki init` — it
registers the repo, generates a scenario matrix behind a plan gate, and
writes `~/.tanuki/scenarios/<target>.scenarios.json`. Manual alternative:

1. Copy `templates/example.scenarios.json` to
   `~/.tanuki/scenarios/<new-target>.scenarios.json`; set `"target"` and
   `"plugin"` (the repo under test — also where /tanuki-loop's integration
   branch lives). Set `"host"` **only if the plugin operates on a
   repository's content** — a self-contained plugin omits it, and every
   scenario runs in a fabricated empty, isolated workspace instead.
2. Define scenarios as exploratory-testing **charters** (goal + persona + the
   branch being explored: host state × skill flow × purpose ×
   **intra-command decision points**). A scenario may pin an internal fork
   with `"decision_points": [{"point": "the framework question", "choice":
   "F2"}]` so two charters run the *same command* down *different internal
   branches*. Prefer breadth over repetition — re-running an unchanged
   scenario is redundant unless it also repeats the internal path (or you're
   establishing chronic-vs-flaky).
3. Optionally add the `"loop"` block (test_cmd, iterations, …) for
   `/tanuki-loop`; without it, the first loop run derives and offers to save
   one.

Then `/tanuki <new-target>` or `/tanuki-loop <new-target>` — or the bare
commands, which list every configured target and ask.

## Vocabulary (fixed)

**Event** — raw per-run fact, never judged, 0..n per run. **Finding** —
judged, deduplicated, ledger-tracked signal with a recurrence count.
**Proposal** — a finding promoted through the human gate. There are no
"observations."

## Status

Prototype (v0). Known deviations from the target architecture are listed in
the spec (no container isolation — dogfood only plugins you trust; the driver
runs headless with permissions skipped inside disposable clones).
