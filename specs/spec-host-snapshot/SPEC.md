# Spec: host snapshot — the run observes a fixture, never the live host

Status: PROPOSED 2026-07-15. Touches `tools/tanuki-loop` (init, doctor,
state), `tools/tanuki-drive` (host resolution), and `commands/tanuki-loop.md`
(the zero-host-footprint rule). Depends on spec-tanuki-loop; refines the
host side of tanuki-drive's isolation contract.

Origin: during loop-20260715-121003 (writing-assistant, Phase 2) the operator
deleted three tracked files in the live host `~/work/product-lab` while the
loop was running — intentional, normal work on their own repo. The loop read
the live host tree, classified the operator's edits as a zero-host-footprint
violation, aborted the run, and **"restored" the files — reverting the
operator's deliberate work**. Meanwhile a parallel product-lab loop committed
to the same host mid-run, so later drives cloned a different host than earlier
ones. The tacit rule this created — *the operator may not touch the host repo
for the entire (multi-hour) run* — is an unacceptable operating burden, and
the auto-restore was a write into a repo the loop has no authority over.

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
  scenarios file's live path. Per-scenario disposable clones, `host_setup`
  (including destructive steps — now genuinely consequence-free), and
  `pollution_check` all keep working unchanged; only the clone *source*
  moves. Every iteration dogfoods the identical host.
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
  clone fails (a run that cannot pin its host must not start). Host base
  freshness at init follows the same rule as the plugin base — fetch and
  fail closed when behind upstream, `--allow-stale-base` to override.
- **`tanuki-loop doctor`**: add a fifth check — the host path resolves and is
  cloneable (a throwaway clone, removed afterward, mirroring the baseline-test
  worktree pattern). Fatal for hosted targets; skipped for hostless.
- **`tanuki-loop finish`**: the one permitted post-init read of the live host
  — `git rev-parse HEAD` for the drift note. Read-only, informational.
- **`tanuki-drive`**: no interface change; the loop passes
  `--host <run-dir>/host-base`. (Attended `/tanuki` single runs may keep
  cloning the live path directly — a one-shot attended run has no drift
  window; the fixture is a *loop* requirement.)
- **`commands/tanuki-loop.md`**: delete "treat any real-host working-tree
  change as a breaker" semantics; replace with the out-of-scope rule above.
  The skill never inspects or modifies any path outside the run dir and the
  loop worktree after init.

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
