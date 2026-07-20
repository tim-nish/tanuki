# Changelog

All notable changes to Tanuki are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions follow
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.3.0] — 2026-07-20

### Added
- **Reconcile substrate — one terminal invocation** (stories 1.38–1.40,
  spec-tanuki-loop #210): `tanuki-loop dispose` (deterministic disposition +
  port engine — per-hunk evidence: reverse-apply / forward-apply / postimage
  containment; anything unevidenced fails closed), `site-record` +
  `reevaluate` (per-site finding closure with stored deterministic probes —
  resolved/superseded only when closed at every known site), and
  `reconcile` — fixpoint of dispose + reevaluate, verification, ONE gate of
  genuine owner decisions, then merge → push → state → cleanup as a
  persisted, resumable, idempotent transaction.
- **Charter probe coverage** (stories 1.24–1.26): charters may declare a
  `probe` block (one `required` evidence predicate plus named checkpoints,
  fail-closed validation, authored at the plan gate); `tanuki-drive`
  evaluates the predicates into an independent probe axis in the manifest;
  report, run summary, and dashboard render the probe axis next to status.
- **Declared-input configuration** (stories 1.27–1.28, 1.37): a typed
  `inputs` block in the scenarios file; `tanuki-config` substrate
  (show / check / set, dry-run preview, per-field doctor hooks); the
  `/tanuki <target> configure` mode; and a built-in input catalog
  (`drive_model`, `drive_concurrency`) so configure works with no `inputs`
  block. Config gains an `int` type with optional min/max bounds (#205).
- **Host bindings & per-host scheduling** (stories 1.29–1.33): declared
  bindings with placeholder resolution in prompts and `host_setup`;
  scheduler state keyed `(scenario_id, host_binding_id)`; host-coupling
  detection (portable / bound / host-coupled); plan-time compatibility check
  with `compat_skipped`; and a one-shot per-run override (consume-once,
  off-host tagging).
- **Den distill — contribute-back** (stories 1.34–1.36, spec-den-distill):
  `contribute_back` config with doctor validation and staging-directory-only
  writes; lesson candidates join the decision pass (accept emits, verbatim
  receipts); the `/tanuki <target> distill` mode walks the existing ledger
  without a new run.
- **Generation surfacing** (#168, #170, #171, #174): `tanuki-scheduler
  candidates` enumerates the full deterministic generation pool
  (trajectory-observed unexplored branches + uncovered findings); the
  generate pass persists its membership record at the gate; `status`
  surfaces generation-trigger advisories.
- **Loop hardening**: committed build artifacts are check (e) of the
  five-part integration invariant — an immediate-stop breaker at the
  offending commit, with per-path `--allow-artifact` overrides (story 1.23,
  #159); concurrent drives with per-scenario progress and summed budget
  accounting (stories 1.21–1.22); `covers` tags with cheapest-cover verify
  selection and covering-absence compaction with a repin guard (stories
  1.19–1.20); estimate/entry-fixture visibility at the plan gate (#204).

### Changed
- `ingest` note run ids gain a time suffix (`manual-<YYYYMMDD>-<HHMMSS>`) so
  same-day notes stay distinct runs (F184; operator-approved contract
  change).
- Morning-gate documentation: re-run/crash-retry outputs
  (`previously_finished`, `branch_deleted`/`note`) documented at the steps
  that produce them (#217); the external-modification breaker's deferred
  detection (next iter-start, never on touch or at iter-verify) stated in
  --help and iter-verify output (#219).
- Dashboard: the two "new finding" counts are labeled with their scopes —
  run-delta vs last cycle's snapshot (#220); every `tanuki-loop` subcommand
  argument now carries a `help=` description (#221).
- Docs information architecture: `docs/README.md` maps normative (user docs,
  specs) vs non-normative (development history, archived planning) material;
  the story registry is marked as history (#229). README rebuilt around the
  two command families: basic-sequence Mermaid diagram, complete linked
  command index, configure as the primary configuration path, and
  write/push boundaries stated per the spec (#228).

### Fixed
- `recover --restore` reports discarded uncommitted edits accurately: the
  `git()` helper preserves porcelain leading columns (no truncated first
  filename) and discarded edits are listed separately from external commits
  (F168/F186, #218; regression-tested).
- Ledger, scheduler, and loop fixes from the overnight dogfooding runs —
  including doctor readiness distinctions, gate-push divergence handling,
  and settlement derivation edge cases (see the audit trail of merged loop
  batches #190, #222 for the itemized list).

## [0.2.0] — 2026-07-18

### Removed
- The **coverage function**, end to end (operator decision, 2026-07-18 — no
  demand: in four days of operation no target ever declared `axes`, and the
  observed-only diff accumulated 99 driver-improvised axis names with
  nothing to compare against). Gone: the `coverage` view
  (`spec-tanuki-view` D2 amended — the catalog is four views: `status`,
  `live`, `history`, `trajectory`), `tanuki-scheduler history --scenarios`
  and its axis-coverage / exploration-debt / recommendations / coverage-diff
  computation and output (`spec-tanuki-trajectory` §2 retired), and the
  generation pass's `axes`/`covers` declaration obligation
  (`spec-tanuki-scenario-lifecycle` "Exploration axes" retired; a matrix may
  still carry the keys — tools ignore them). **Kept:** typed trajectory
  events and `decision_points` capture (the `trajectory` view consumes
  them), repo-wide scenario states (unexplored / long-unrun / productive),
  and the plan's exploration quota. Side effect: a four-view catalog fits
  the interactive picker's four-option limit, dissolving issue #103's
  paging problem.

### Added
- PR-protected targets: `"gate": "pr"` in the scenarios `"loop"` block
  reshapes the morning gate for a base branch that refuses direct pushes —
  the overnight close delivers **one Draft PR** via `tanuki-loop gate-pr`
  (pushes the integration branch, never forced; idempotent and
  failure-safe), and the Human Gate becomes PR approval + merge on the
  forge. The loop ends at delivery (owner ruling 2026-07-17): no closing
  command; `status`, `init`, and `unresolved` **derive** the settlement
  (landed | pending | declined | unknown) from the forge and the current
  base, offline-safe with a stale-marked cache.
- `tanuki-loop unresolved` + the `/tanuki-loop <target> reconcile` sitting —
  the debt-paydown path for integration branches that outlived their merge
  window: `unresolved` reports which loop branches never merged, their
  staleness, and per-commit mechanical signals (files touched, clean
  cherry-pick, cited findings with current status) — **signals, never
  verdicts**; `reconcile` classifies every hunk against the current base
  (already-landed / superseded / conflicting / still-applicable /
  worth-preserving), produces a landing plan, and mutates nothing until one
  explicit gate. `~/.tanuki/<target>/unresolved.md` is the stable-path
  brief.
- `/tanuki <target> view [name]` — one option-free door to every read-only
  view (`specs/spec-tanuki-view/SPEC.md` D1/D2): bare `view` opens a picker
  over the closed catalog — `status`, `live`, `history`, `trajectory` (the
  catalog shipped with `coverage` as a fifth view; removed in this same
  release, see Removed above) — each with a state-derived hint; `view <name>`
  jumps straight there. `trajectory` gains a named door of its own
  (previously reachable only through
  the `--history <scenario>` flag overload). The surface reads and renders
  only; the `tanuki-*` tools remain the fully-optioned computing substrate,
  and no view writes. View vocabulary is target-local in v1: axis names
  render exactly as the substrate emits them, with no normalization and no
  cross-target comparability claim (spec OPEN-2, resolved by operator
  ruling).
- `tanuki-loop recover` — attended recovery from the external-modification
  breaker once the last iteration is closed: `--restore` resets the loop
  worktree to `head_expected` (discarded range audited), `--adopt`
  re-baselines `head_expected` onto the current worktree HEAD (adopted range
  audited, non-ancestor rewrites warned). No default mode; any other state is
  refused with the breaker convention, and closed iterations are never
  mutated. (#33)
- Host-snapshot fixture: `tanuki-loop init` pins a run-scoped clone of the
  live host (`<run-dir>/host-base`, `host_base_sha`/`host_fixture_path` in
  `state.json`, fail-closed on clone failure); every drive receives the
  fixture, `doctor` gains a fifth host-cloneable check (fatal for hosted
  targets), and `finish` reports live-host drift as an informational note.
  After init the live host is out of scope by construction. (#15, #17, #18)
- `tanuki-loop doctor` — a read-only Phase 2/3 headless-readiness validator:
  required loop ceilings (`test_cmd`, `wall_time_s`, `token_budget`,
  `attempt_cap`, `iterations`), the base-freshness guard, a baseline
  `test_cmd` run on the base tip (throwaway worktree, removed afterward), and
  policy coverage (which findings would defer, informational). Reuses `init`'s
  config resolution and guards; not-ready follows the breaker convention
  (exit 3) without persisting anything.
- Continuous integration: the tool test suites run on every push and pull
  request (GitHub Actions).
- `CHANGELOG.md` and a tests/license badge row in the README.
- `tanuki-loop gate-push` — the morning gate now pushes `main` to its remote
  after `gate-check` and before materializing issues, so issue commit links
  resolve and the remote never diverges under a concurrent workflow. A diverged
  remote is refused (reconcile + re-run), never force-pushed. (#5)

### Changed
- The README documents the operator-facing loop surfaces that existed only
  in `--help` and the command docs: `doctor` (with its required
  `--loop-repo`/`--scenarios`), the unattended-run ceilings
  (`--wall-time`/`--token-budget`/`--phase`), the `"gate": "pr"` Draft-PR
  flow, `dashboard --live` vs `--follow`, and a standing pointer that each
  tool's `--help` is the complete surface — loop-internal subcommands the
  skill drives (`iter-start`, `record-cycle`, …) are expected, not
  undocumented.

### Fixed
- `tanuki-ledger compact` now works on hand-ingested ledgers: `ingest` and
  `ingest-note` write a minimal manual run manifest (`results: []`, never
  overwriting a driven one), so the elapsed-run verify-by-absence fallback
  the spec already promised can actually advance — previously an accepted
  finding on a hand-built ledger reported "nothing to compact" forever.
  A manual run drives no scenario, so it can never manufacture false
  driven-absence for a real one. (#106)
- The `tanuki-loop` dashboard surfaces the committed build-artifact warning
  that `status` and `iter-verify` already report (same computation, shared),
  so an operator watching only the live screen sees what `gate-push` will
  later refuse. (F129)
- `tanuki-loop status` `verdict` honors a recovered breaker: after
  `recover --restore`/`--adopt` (and the successful `iter-start` that
  follows) the verdict no longer reads "stopped" with the stale breaker
  reason while `lifecycle.active` is true. (F118)
- The no-patch `iter-verify` breaker ("HEAD == start SHA but a patch was
  expected") gives a concrete remedy — commit the work, or `rollback` —
  instead of the generic four-part-failure hint. (F131)
- `view live`'s empty states explain the pre-`init` window: a just-launched
  loop is invisible until `init` persists state (target resolution,
  `doctor`, and preflight are deliberately stateless, and doctor can take
  minutes), so the `live:no-active-run` / `live:never-initialized` reasons
  now say so instead of misleading the operator who just started a loop.
  (#102)
- The bare `view` picker is pinned to **all four** catalog views as named
  options — the four-view catalog fits the interactive picker's four-option
  cap exactly, so a view is never demoted behind the free-text/`Other`
  affordance and never dropped because it was already viewed this session.
  (#103)
- `view live` means an actively running loop, never the final dashboard of
  the latest completed run (`specs/spec-tanuki-view/SPEC.md` D2, amended):
  `tanuki-loop dashboard --live` renders the dashboard only while the run is
  active and degrades a closed run to a typed empty state
  (`live:no-active-run` / `live:never-initialized`) plus one historical line.
  Stop reason and settlement are two separated, typed facts (spec-tanuki-loop
  "Stop reason vs delivery settlement", over the delivery-boundary ruling):
  the first computational close (cap | converged | breaker | cancelled) is
  preserved in `state.json` `stop` and a settlement — including the
  compatibility `gate-ratified` — never renders as why the run stopped.
  Views derive settlement **offline** (`derive_settlement(offline=True)`:
  local reachability or the stale-marked cache — a view never polls the
  forge; unknown never reads landed). Execution time freezes at the first
  close (`wall_end`) — review/merge waiting never accrues. Dashboard
  counters are ledger facts: `fixed` retired for `status→accepted`, and the
  convergence line says patch commits were *produced*, not "LANDED".
  `tanuki-loop status` gains a typed `lifecycle` block carrying all of it.
- Hostless self-dogfood scenarios now run the disposable plugin clone, never
  the operator's real checkout: the driver's sandbox preamble pins the clone
  path, and a post-run execution-escape check asserts nothing outside the
  workspace/clone was run (assert what *ran*, not just what was left dirty).
  Cumulative fixes are actually dogfooded again. (#13)
- The `tanuki-loop` dashboard always renders its six fixed sections (header/
  health, latest drive, this run, scheduler decisions, convergence, why
  stopped/NEXT), each degrading to a stated reason instead of disappearing;
  the no-mode breaker path recovers instead of crashing the dashboard. (#36)
- `tanuki-loop init` no longer dogfoods stale code: it fetches the base
  branch's remote and **fails closed** when the local base is behind its
  upstream (`--allow-stale-base` overrides; the base is never silently moved to
  the upstream tip). The base tip and any behind-count are recorded in
  `state.json` and the audit. (#2)
- Scenarios config location is now enforced as canonical: the loaders
  (`tanuki-drive`, `tanuki-loop init`, `tanuki-scheduler`) fail closed naming
  `~/.tanuki/scenarios/<target>.scenarios.json` when the config is absent,
  instead of a raw traceback or silently reading a stale copy from an
  undocumented path. (#4)
- Merged integration branches now have a cleanup owner: `tanuki-loop finish`
  (backstop: the next `init`) deletes integration branches whose tip is
  reachable from the base branch and reports any it kept — unmerged tips are
  never deleted. Previously they accumulated until an unrelated tool collected
  them. (#6)

## [0.1.0] — 2026-07-14

First public release — Tanuki packaged as a Claude Code plugin.

### Added
- `/tanuki` — drive branching user scenarios in disposable clones, mine the
  recorded Events into deduplicated Findings with recurrence tracking, and
  consolidate chronic ones into a capped, ranked Proposal brief. Proposals
  only: nothing merges, nothing writes to the target repos.
- `/tanuki-loop` — unattended cumulative dogfooding on an isolated integration
  branch until convergence or a fixed iteration cap, then a morning review
  gate where the operator ratifies one batch.
- Adaptive scenario scheduler with cross-run history, exploration coverage,
  and exploration-debt reporting.
- Operational-status loop dashboard: health verdict, anomalies classified
  against the ledger, run-scoped deltas, scheduler decisions, and a spelled-out
  convergence explanation.
- Deterministic tools (zero runtime dependencies): `tanuki-drive`,
  `tanuki-ledger`, `tanuki-scheduler`, `tanuki-loop`, `tanuki-preflight`.

[Unreleased]: https://github.com/tim-nish/tanuki/compare/v0.3.0...HEAD
[0.3.0]: https://github.com/tim-nish/tanuki/compare/v0.2.0...v0.3.0
[0.2.0]: https://github.com/tim-nish/tanuki/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/tim-nish/tanuki/releases/tag/v0.1.0
