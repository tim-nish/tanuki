# ARCHIVED — tanuki-spec paraphrase layer (moved 2026-07-27)

> **Non-normative.** This file is development history (docs/README.md kind
> 3/4): the command-grammar, decision-pass, and file-layout sections that
> lived in `docs/tanuki-spec.md` until story 5.14 (issue #345) archived
> them. They paraphrased contracts owned elsewhere and are **not** current
> governing text. Authoritative today:
> - argument grammar, init, views, ingest — `commands/tanuki.md`
> - command shape (bare word / flags / drive) — `specs/spec-short-command-surface/SPEC.md` D6
> - read-only views — `specs/spec-tanuki-view/SPEC.md`
> - decision pass — `specs/spec-short-command-surface/SPEC.md` D1 + `specs/spec-tanuki-solve/SPEC.md` D1/D3
> - repo/state file layout — `README.md` §"What's in this repo" tables
>
> Kept verbatim below for provenance; never edit this file.

## Command — `/tanuki [target] [scenarios… | "<ad-hoc free text>" | init | decide | status | history | view | mine <run> | ingest "<feedback>"]`

**Command shape (the rule):** a bare word selects a mode that **does not
drive**; flags modify a drive; the bare default is driving. Ratified in
`specs/spec-short-command-surface/SPEC.md` D6 (issue #74) and applied
surface-wide. The previously documented flag spellings are retained as
aliases: `--brief` → `decide`, `--status`, `--history`, `--mine-only` →
`mine`, `--ingest` → `ingest`. `view` is the **option-free door to every
read-only view** (`specs/spec-tanuki-view/SPEC.md`): bare `view` opens the
view picker; `view <name>` jumps to one of the closed catalog — `status`,
`live`, `history`, `trajectory`. The surface reads and renders;
the tools stay the computing substrate, and no view ever writes.

(`commands/tanuki.md` is authoritative for the full argument grammar —
including `init` onboarding, `--history`, and ad-hoc free-text scenarios from
spec-tanuki-scenario-lifecycle; this section states the pipeline contract.)

`commands/tanuki.md` orchestrates: resolve target config
(`~/.tanuki/scenarios/<target>.scenarios.json`) → preflight (stop on failure) →
plan gate → drive (subset of scenarios if named) → mine → consolidate →
decision pass. UX rules, from dogfooding Tanuki itself:

- **No-argument invocation resolves, then falls back to a picker — never an
  error** (resolution order per commands/tanuki.md and
  spec-tanuki-scenario-lifecycle): explicit argument → registry match for the
  cwd (`~/.tanuki/registry.json`) → single configured target → **picker**
  listing every `~/.tanuki/scenarios/*.scenarios.json` target with its
  one-line `status` summary (AskUserQuestion — selection beats typing names
  without completion). Adding a target = copying an existing scenarios file;
  no config edit is needed to switch targets.
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
- **`/tanuki <target> decide`** is the decision pass: the complete,
  **ledger-anchored** path from whatever the ledger holds decidable to
  labeled issues — consolidation before presentation, then promote → decide →
  file (spec-short-command-surface D1). It orients off `status`/`next` and
  needs no recent run or brief; a brief is reprinted as context when one
  exists. **`--brief`** is an alias that reprints the latest brief and
  continues into the same pass — the artifact is what its name promises,
  never the gate. **`/tanuki <target> status`** runs the ledger's `status`
  view and stays read-only.
- **`/tanuki <target> --ingest "<feedback>"`** records human feedback in
  natural language (see "Human feedback ingest") and runs extraction + dedupe
  on it immediately, reporting the delta (bumped vs new) like any run.
- **All intermediates live under `~/.tanuki/<target>/events/<run>/`** —
  never the session temp dir, never the target repos. The brief's canonical
  home stays `~/.tanuki` (writing it into a repo would pollute it); the
  in-session presentation + `decide` are the access path.

## The decision pass — `/tanuki [target] decide` (alias `--brief`)

(`specs/spec-short-command-surface/SPEC.md` D1 is authoritative for the pass
and its entry points; `specs/spec-tanuki-solve/SPEC.md` D1/D3 remain
authoritative for the consolidation taxonomy; `commands/tanuki.md`
implements both. Folded 2026-07-17, issue #72 — this was `/tanuki-solve`, a
separate command, until consolidation moved into the pass itself and the
separate surface was retired; its capability, one attended command from open
findings to labeled issues with no raw ledger invocations, is preserved
here.) The consolidate-then-decide pass: promote → **consolidate**
(deterministic candidate groups from `tanuki-ledger consolidate`, judged and
classified as merge / conflict / dependency at the command layer) → decision
pass over the consolidated plan → confirmed filing → watching list. A
conflict group is presented as ONE multi-outcome question naming the
branches; a filed issue from a resolved conflict records the rejected
alternative in its body. With fewer than two candidate items there is nothing
to consolidate and the stage is skipped. Disposition mechanics, the promotion
bar, and the downstream boundary are unchanged by the fold — the pipeline
still ends at the labeled issue, and no downstream tooling is invoked, named,
or configured (an operator's decide→file→triage chain composes outside this
repo, on the far side of the label boundary).

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

