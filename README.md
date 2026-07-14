<p align="center">
  <img src="docs/assets/tanuki.png" alt="Tanuki" width="360">
</p>

# Tanuki 🦝

**Automated dogfooding for Claude Code plugins.**

You built a Claude Code plugin. It works when *you* use it — but you know
exactly which buttons to press. What happens when a first-time user follows
your README? When someone picks the option you never test? When the config
file is missing?

Tanuki finds out for you. It plays the role of a realistic user: it runs your
plugin through scripted user scenarios in disposable clones of your repo,
records everything that happens, and distills the recurring pain points into
a short, ranked list of concrete improvement proposals — with evidence
pointing back to the exact moment each problem occurred.

**You stay in control.** Tanuki never merges anything, never files issues on
its own, and never writes into the repo under test. Its output is a brief of
proposals; every decision after that is yours.

Full technical contract: [docs/tanuki-spec.md](docs/tanuki-spec.md).

## How it works, in one paragraph

Tanuki clones your plugin (and optionally a "host" repo it operates on) into
a throwaway workspace, then launches a **headless Claude session on a cheap
model** that acts out a scenario — "a new user tries the quickstart", "a user
picks the unusual option", "the config file is broken". The transcript is
mechanically normalized into **Events** (tool errors, retries, user choices,
friction notes). A mining pass turns events into **Findings** — deduplicated
problems with a recurrence count, so a problem seen in three runs counts as
chronic, not three separate complaints. Chronic findings get promoted into a
**brief**: at most 10 ranked **Proposals**, each written as
Problem → Proposed fix → Evidence. You then accept, dismiss, or defer each
one in-session.

Why a *cheap* model as the simulated user? Because real users include weak
agents. A frontier model quietly works around rough edges; a weaker one trips
over them — which is exactly the signal you want.

```
preflight (lint, code)          mechanical violations stop here — never
        │                       rediscovered by dogfooding
        ▼
plan gate (you approve)         scenarios × estimated time, from run history
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
                                a run ends in decisions, not a report
```

## Install

Tanuki is itself a Claude Code plugin; this repository doubles as its
marketplace. Inside Claude Code:

```
/plugin marketplace add tim-nish/tanuki
/plugin install tanuki@tanuki
```

Requirements: the `claude` CLI on PATH, `git`, Python 3. Nothing else — no
pip packages, no configuration files needed to start.

## Quickstart

```
cd ~/work/my-plugin
/tanuki init          # one-time onboarding: Tanuki reads your plugin's docs
                      # and proposes 4–6 test scenarios for your approval
/tanuki               # run it — shows the plan first, drives after you approve
```

That's the whole loop. After a run, useful entry points:

```
/tanuki my-plugin --status              # what's still waiting on my decision?
/tanuki my-plugin --brief               # reopen the latest brief, resume deciding
/tanuki my-plugin --ingest "…"          # log friction YOU hit, in plain words
/tanuki "try the export flow with a huge file"   # one-off ad-hoc scenario
/tanuki-loop                            # unattended overnight mode (below)
```

Every run ends with a **delta report** — which known problems recurred and
what's new — so you never need to remember previous runs.

## Reporting friction you found yourself (`--ingest`)

You'll keep using your own plugin, and you'll keep hitting things. Instead of
a TODO list you'll lose, hand the observation to Tanuki in plain language:

```
/tanuki my-plugin --ingest "The README says to look for the fact-sheet,
but I couldn't tell where it was written."
```

The one rule: **you never classify.** Don't decide whether it's a bug, a
papercut, or a duplicate — that's Tanuki's bookkeeping. Your words are stored
verbatim, then run through the same mining pass as an automated run: if it's
semantically a problem Tanuki already knows, that finding's recurrence count
goes up (your manual re-hit pushes it toward "chronic" like any other); if
it's new, a new finding is created. Either way you get the same bumped-vs-new
delta report. It's instant and costs nothing — use it the moment friction
bites.

## The decision pass (the human gate)

A run doesn't end with "here's a report." Tanuki walks you through each
promoted proposal — shown as **Problem → Proposed fix**, with evidence
collapsed to a pointer line you'll rarely need — and records one disposition
each:

- **accept** — optionally file the prepared GitHub issue right then (its own
  explicit confirmation; nothing is ever auto-filed). Filed issues carry a
  `tanuki` label plus `tanuki:<kind>` — the only machine-readable trace
  Tanuki leaves.
- **dismiss** — it never resurfaces as new (but stays deduplicated against).
- **defer** — stays pending; `--status` keeps it visible until you decide.

Accepted findings keep their recurrence tracking, which gives you a free
regression check: after you ship a fix, run Tanuki again — the finding's
*absence* verifies the fix landed.

## Unattended overnight mode (`/tanuki-loop`)

Once you trust the attended loop, `/tanuki-loop` runs the full cycle —
**drive → mine → classify → implement → test → commit** — repeatedly on a
dedicated integration branch in an isolated worktree, until two consecutive
quiet cycles (nothing new found, nothing fixed) or an iteration cap.

The human gate is *relocated, not removed*: the loop prepares a batch
overnight; in the morning you review the integration diff, the deferred
decisions, and the audit trail, and only you merge to `main`. No phase of the
loop ever merges, pushes, or files issues unattended. Full contract:
[spec-tanuki-loop](specs/spec-tanuki-loop/SPEC.md).

```
/tanuki-loop my-plugin                   # settings stored once in the scenarios file
/tanuki-loop my-plugin --iterations 2    # cautious first run
```

Watch it live with the dashboard (safe to run alongside the loop — it only
reads state files):

```
<plugin-root>/tools/tanuki-loop --target my-plugin dashboard --follow 10
```

The loop needs one thing from you: a `test_cmd` — your repo's regression
gate, run after every iteration. A failing gate stops the loop immediately.
On the first run without one configured, Tanuki derives a candidate from
your repo and asks you to confirm it.

## Vocabulary

Three words carry the whole design; everything flows one way through them:

| term | meaning |
|---|---|
| **Event** | a raw fact from one run ("tool X errored at turn 12"). Never judged, never edited. |
| **Finding** | a judged, deduplicated problem with a recurrence count across runs. |
| **Proposal** | a finding that crossed the promotion bar and now awaits your decision. |

Events → Findings → Proposals → (your call) labeled GitHub issues. Nothing
skips a step, including your own `--ingest` feedback.

## What's in this repo

| path | role |
|---|---|
| `commands/tanuki.md` | the `/tanuki` command — orchestrates the whole pipeline |
| `commands/tanuki-loop.md` | the `/tanuki-loop` command — unattended mode |
| `tools/tanuki-preflight` | mechanical lint of the plugin under test (runs before any scenario) |
| `tools/tanuki-drive` | the Driver: isolation, headless runs, event capture, live progress |
| `tools/tanuki-ledger` | the Findings ledger: ingest, dedupe support, promotion, status |
| `tools/tanuki-scheduler` | picks which scenarios each run should drive, adaptively |
| `tools/tanuki-loop` | deterministic safety substrate for the unattended loop |
| `tools/tests/` | test fixtures for the tools (`for t in tools/tests/test-*; do "$t"; done`) |
| `templates/example.scenarios.json` | starting point for your scenario file |
| `templates/config.example.json` | every config key with its default |

Everything Tanuki generates lives under `~/.tanuki/`, **never in your
repos**:

| path | what it is |
|---|---|
| `~/.tanuki/scenarios/<target>.scenarios.json` | your per-target scenario matrix (`/tanuki init` writes it) |
| `~/.tanuki/<target>/briefs/<run>.md` | **the deliverable** — the ranked proposal brief |
| `~/.tanuki/<target>/ledger.json` | persistent findings (don't hand-edit; use `tanuki-ledger set-status`) |
| `~/.tanuki/<target>/events/<run>/` | transcripts, normalized events, live `progress.json` |
| `~/.tanuki/<target>/ws/<run>/` | the disposable clones — delete freely |

## Configuration

Zero configuration is a supported state — every key has a sane default.
When you want to tune: `~/.tanuki/config.json` (global) < a `"defaults"`
block in the target's scenarios file < CLI flags.

Highlights: `driver_model` (default claude-sonnet-5 — the simulated user),
`model_ceiling` (default sonnet — the most capable model Tanuki is allowed to
launch; frontier-class models are always refused), `max_scenarios` (default
6 — hard cap on parallel scenarios per run), `max_turns` (default 40 per
scenario). The full table with all keys lives in
[docs/tanuki-spec.md](docs/tanuki-spec.md#configuration), mirrored in
[templates/config.example.json](templates/config.example.json).

## Writing good scenarios

`/tanuki init` generates your scenario file, but the ideas travel: each
scenario is an exploratory-testing **charter** — a goal plus a persona plus
the specific branch being explored ("a first-time user follows the
quickstart", "a user whose config is missing"). Two scenarios that run the
*same command* but pin *different answers* to a question the command asks
(via `"decision_points"`) are exploring different paths — that's breadth, and
breadth beats repetition. Re-run an unchanged scenario only to tell a chronic
problem from a flaky one.

If your plugin operates on a repository's content (say, a docs generator),
set `"host"` in the scenarios file and Tanuki clones that repo as the
workspace. Self-contained plugins omit it and get a fresh empty workspace per
scenario.

## Status

Prototype (v0). Known limitations, spelled out in
[the spec](docs/tanuki-spec.md#prototype-deviations-explicit-to-revisit-before-any-generalization):
isolation is git-clone-plus-pollution-check, not a container, and the driver
runs headless with permission prompts skipped inside its disposable clones —
so point Tanuki only at plugins you trust.
