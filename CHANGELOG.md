# Changelog

All notable changes to Tanuki are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions follow
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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
- `/tanuki <target> view [name]` — one option-free door to every read-only
  view (`specs/spec-tanuki-view/SPEC.md` D1/D2): bare `view` opens a picker
  over the closed catalog — `status`, `live`, `history`, `coverage`,
  `trajectory` — each with a state-derived hint; `view <name>` jumps
  straight there. `coverage` and `trajectory` gain named doors of their own
  (previously reachable only as a conditional block inside `--history` and
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

### Fixed
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

[Unreleased]: https://github.com/tim-nish/tanuki/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/tim-nish/tanuki/releases/tag/v0.1.0
