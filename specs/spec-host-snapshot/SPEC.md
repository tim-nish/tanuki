# Spec: host snapshot — the run observes a fixture, never the live host

Status: RATIFIED 2026-07-15 (triage of issue #15; amended at ratification —
genericized origin, additive skill rule, host_setup escape guard, read-only
freshness). Touches `tools/tanuki-loop` (init, doctor, state),
`tools/tanuki-drive` (host resolution + touched-path guard), and
`commands/tanuki-loop.md` (the out-of-scope rule). Depends on
spec-tanuki-loop; refines the host side of tanuki-drive's isolation contract.

Origin: during loop-20260715-121003 (writing-assistant, Phase 2) the operator
deleted three tracked files in the live host — a repo of their own they were
concurrently working in — while the loop was running: intentional, normal
work. The orchestrating session read the live host tree, classified the
operator's edits as a footprint violation, aborted the run, and **"restored"
the files — reverting the operator's deliberate work**. Meanwhile a parallel
loop on another target committed to the same host mid-run, so later drives
cloned a different host than earlier ones. The tacit rule this created — *the
operator may not touch the host repo for the entire (multi-hour) run* — is an
unacceptable operating burden, and the auto-restore was a write into a repo
the loop has no authority over.

## The principle

**The live host is a clone source at `init`, and nothing else, ever.** A run
observes a *fixture* — an immutable, run-scoped snapshot of the host pinned at
init time. After init the real host tree is out of scope: never read, never
diffed, never written, never "restored". Operator activity in the real host
during a run is invisible **by construction** — not tolerated, unobservable.
This is the same move the loop already makes for the plugin side (dedicated
worktree, base SHA recorded at init); the host gets the symmetric treatment.

## What exists today, and the two actual gaps

`tanuki-drive` already never executes against the real host: every scenario
gets a fresh disposable clone (`prepare_workspace`), `host_setup` runs inside
the clone, and `pollution_check` inspects the clone. Two gaps remain:

1. **Snapshot timing.** Each drive clones from the **live path at drive
   time**, so the host state drifts across a run: operator edits and
   parallel-loop commits land in later iterations' clones. Iterations of one
   run dogfood different hosts; findings are non-reproducible and
   misattributable.
2. **The orchestrating session.** The loop skill's zero-host-footprint rule
   sends the *judgment layer* to look at the **real** host tree for
   anomalies. Any concurrent operator activity reads as a violation, and the
   skill's remediation (`git checkout --`) writes into the operator's repo.
   The drive was never the leak — the skill was.

## Non-negotiables

- **Pin once, at `init`.** `tanuki-loop init` materializes the host fixture:
  `git clone --quiet --no-hardlinks <live-host-path> <run-dir>/host-base`
  and records `host_base_sha` (the fixture's HEAD) in `state.json` next to
  the plugin base SHA. Git-native and effectively atomic — no rsync, no raw
  `.git` copy of a repo another process may be writing (racy, non-portable).
- **Committed state only, by design.** A clone carries the committed tip; the
  operator's uncommitted host changes are deliberately excluded from the
  fixture. A run is a controlled experiment against a declared state, and
  "the host's committed tip at init" is the only state that is reproducible,
  attributable, and cheap to pin. (This is already true of today's per-drive
  clones — the spec makes it a contract instead of an accident.)
- **Drives clone from the fixture.** For the whole run, every
  `tanuki-drive --host` resolves to `<run-dir>/host-base`, never the
  scenarios file's live path. Per-scenario disposable clones, `host_setup`,
  and `pollution_check` all keep working unchanged; only the clone *source*
  moves. Every iteration dogfoods the identical host. Destructive
  `host_setup` steps are consequence-free **only when sandbox-relative** —
  see the next non-negotiable.
- **`host_setup` and the scenario must never touch the live host path.**
  `host_setup` runs `shell=True` with the clone as cwd, so an absolute path
  still reaches the real tree — the incident's own `host_setup` did exactly
  that. Sandbox-relative paths are the contract, and the drive enforces it:
  assert *what was touched* — fail the scenario if `host_setup` or any
  scenario command referenced the clone-source path (the same
  assert-what-was-run posture as the hostless execution-escape check,
  issue #13).
- **The real host is never written. No auto-restore, any phase.** If the
  loop believes the real host changed, that belief is *out of its
  jurisdiction*: the operator's repo is the operator's. Remediation of real
  trees is not a loop capability.
- **Anomaly checks compare fixture against fixture.** The zero-host-footprint
  question becomes "did the drive leave a footprint in *its* workspace?" —
  answered entirely inside `host-base` + the disposable clones. The skill
  text must stop referencing the real host path after init.
- **Drift is informational, never a breaker.** The morning report may note
  "live host moved during the run (`<sha>` → `<sha>`, N commits)" — one
  read-only `git rev-parse` at `finish`, so the operator knows findings were
  mined against the init-time view. It never blocks, aborts, or reconciles.
- **One fixture per run, not per iteration.** Iterations stay comparable
  against the same base. A scenario needing a pristine host each drive gets
  it for free — its disposable clone is already fresh from `host-base`.
- **Hostless scenarios are unaffected.** A plugin with no host keeps the
  fabricated-workspace path (issue #13); this spec touches only hosted runs.

## Component changes

- **`tanuki-loop init`**: clone the host fixture into the run dir; record
  `host_base_sha` and `host_fixture_path` in `state.json`; fail closed if the
  clone fails (a run that cannot pin its host must not start). Host
  freshness is **informational only, and read-only**: a `git ls-remote`
  compare may note the host is behind its upstream, but init never fetches
  in the live host (`git fetch` writes its `.git` — remote-tracking refs)
  and never fails closed on host staleness. The plugin base earns
  fail-closed freshness because it is the code under test; the host is an
  observation fixture, and its staleness is the operator's business.
- **`tanuki-loop doctor`**: add a fifth check — the host path resolves and is
  cloneable (a throwaway clone, removed afterward, mirroring the baseline-test
  worktree pattern). Fatal for hosted targets; skipped for hostless.
- **`tanuki-loop finish`**: the one permitted post-init read of the live host
  — `git rev-parse HEAD` for the drift note. Read-only, informational.
- **`tanuki-drive`**: no interface change; the loop passes
  `--host <run-dir>/host-base`. (Attended `/tanuki` single runs may keep
  cloning the live path directly — a one-shot attended run has no drift
  window; the fixture is a *loop* requirement.)
- **`commands/tanuki-loop.md`**: **add** the out-of-scope rule above — no
  real-host-breaker text exists today; the incident behavior was improvised
  by the orchestrating session, which is exactly why the rule must be
  written down. The skill never inspects or modifies any path outside the
  run dir and the loop worktree after init.

## Alternatives rejected

- **rsync/cp snapshot including `.git`** — captures dirty state, but copying
  a live `.git` mid-write is racy, the tool is platform-dependent, and dirty
  state is exactly what a reproducible fixture should exclude.
- **Per-drive live-path clone (status quo)** — mid-run drift; iterations
  incomparable; operator activity leaks into findings.
- **Watch-and-pause (detect operator activity, pause the loop)** — keeps the
  live host in scope, inverts the burden again, and races the operator.
  Unobservability beats detection.
- **Remote-pinned fixture (`url@sha`) / fixture-builder scripts** — the right
  long-run generalization for CI and third-party contributors, but out of
  scope here; the local-path pin is the contract, and a `host:` source with
  modes can extend it later without breaking `state.json`.
