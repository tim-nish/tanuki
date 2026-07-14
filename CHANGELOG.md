# Changelog

All notable changes to Tanuki are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions follow
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
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
