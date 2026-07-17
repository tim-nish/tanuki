Unattended cumulative dogfooding: run the Tanuki cycle (drive → mine →
classify → implement → test → commit) over and over on a dedicated
integration branch, in an isolated worktree, until it converges or hits a
fixed iteration cap — then hand one batch to the operator's morning review.
The Human Gate is relocated, not removed: the loop **prepares**, the operator
**ratifies**. Contract: `${CLAUDE_PLUGIN_ROOT}/specs/spec-tanuki-loop/SPEC.md`. Reuses the Tanuki
pipeline (`${CLAUDE_PLUGIN_ROOT}/docs/tanuki-spec.md`) and the *logic* of the
operator's attended triage / implementation / commit-grouping commands (rules
copied here, allowed to diverge; owner decision record 2026-07-13, loop
fork) — never their GitHub-issue substrate.

Every `tanuki-drive` / `tanuki-ledger` here is the executable under
`${CLAUDE_PLUGIN_ROOT}/tools/` (not on PATH — invoke by full path).

Argument handling ($ARGUMENTS) — **the normal invocation is just
`/tanuki-loop` or `/tanuki-loop --iterations 5`**; repo-specific settings
live in the scenarios file's `"loop"` block, stored once, never retyped:
- *(empty target)*: **target picker**, exactly like /tanuki — list every
  `~/.tanuki/scenarios/*.scenarios.json` target with its one-line
  ledger status and ask which to run. One target → use it without asking.
- `<target>`: a slug with a scenario config at
  `~/.tanuki/scenarios/<target>.scenarios.json`. The config's `plugin` repo is where
  the integration branch and worktree live — findings about that plugin are
  fixed there.
- **Per-target loop config** comes from the scenarios file's `"loop"` block —
  `{"test_cmd", "iterations", "wall_time_s", "token_budget", "attempt_cap"}` —
  which `tanuki-loop init --scenarios <file>` reads directly. CLI flags
  override it per run. **First run of a target without a `"loop"` block:**
  derive a sensible `test_cmd` from the repo (its test runner / check
  scripts, minus checks that already fail on the base branch — those are
  baseline, not regressions), confirm it with the user once, and offer to
  save the block into the scenarios file so every later run is zero-config.
- `--iterations N`: overrides the config cap — a safety ceiling, not the
  success condition. Phase 1 commissioning uses a small cap (1–2).
- `--wall-time <dur>` / `--token-budget <N>`: override the config ceilings
  (breakers). Always ensure they're set (config or flag) for an unattended
  (Phase 2/3) run.
- `--phase 1|2|3` (default 1): supervision level only — same loop throughout.
  Phase 1 runs supervised; Phase 2/3 run headless
  (`--dangerously-skip-permissions`). **Questions are asked only in iteration 1**
  and recorded as the **run policy**; iterations ≥ 2 never ask and never stop
  for judgment. A headless run supplies the policy up front (a prior run's
  `policy.json` or `--policy <file>`); absent an answer the item is deferred,
  not asked. No phase files issues overnight or merges to `main`. Before any
  Phase 2/3 run, validate readiness with `tanuki-loop doctor` (Preflight
  step 2) — it enforces the ceilings this section only asks for.

## 0. Preflight (once)

1. Resolve the target scenarios config; expand `~` in its `plugin`/`host`
   paths. The `plugin` repo is the **loop repo**.
2. **Headless readiness (Phase 2/3 only) — `tanuki-loop doctor`.** Before an
   unattended run, validate the target **read-only** — no pipeline run, no
   state written: `${CLAUDE_PLUGIN_ROOT}/tools/tanuki-loop --target <target>
   doctor --loop-repo <plugin-repo> --scenarios <scenarios-file>
   [--policy <file>] [--allow-stale-base]`. It reuses init's own config
   resolution and guards and checks: the required `"loop"` ceilings
   (`test_cmd`, `wall_time_s`, `token_budget`, `attempt_cap`, `iterations` —
   the "always ensure they're set" rule, enforced), **base freshness** (the
   same measurement `init` fails closed on), that **`test_cmd` passes on the
   base tip** (in a throwaway detached worktree, removed afterward — a
   baseline failure would trip the test breaker on inherited state, not a
   regression), **policy coverage** (which actionable findings have no
   policy answer and would defer — informational, never fatal), and **host
   cloneable** (hosted targets: the live host path exists and a throwaway
   clone succeeds, per spec-host-snapshot; hostless targets pass with a
   note). Not-ready
   follows the breaker convention (`{"breaker": …}` + exit 3, nothing
   persisted); fix the named checks before `init`. Phase 1 runs may skip
   this step.
3. **Isolation + state — `tanuki-loop init`.** Run `${CLAUDE_PLUGIN_ROOT}/tools/tanuki-loop
   --target <target> init --loop-repo <plugin-repo> --scenarios
   <scenarios-file> --phase <P> [overrides: --cap/--test-cmd/--wall-time/
   --token-budget] [--policy <file>]` — the `"loop"` block supplies the
   repo-specific settings. It creates the
   integration branch `tanuki-loop/<target>/<ts>` in a **dedicated worktree
   outside the repo**, records the base SHA, and initializes `state.json`,
   `audit.md`, `policy.json`, and the morning queue. It never touches the
   operator's normal working tree; if the worktree cannot be created it stops
   (no shared-tree fallback). It prints the worktree path + integration branch;
   all iteration work happens in that worktree.
   **Freshness (never dogfood stale code).** `init` fetches the base branch's
   remote and **fails closed** when the local base is behind its upstream —
   otherwise every iteration would base on stale plugin code and surface
   already-fixed findings at the morning gate. Fast-forward the base (or pass
   `--base origin/<branch>`) and re-run; `--allow-stale-base` is the explicit
   override for a deliberately un-pushed local branch. The base is **never
   silently moved** to the upstream tip — the loop always dogfoods the ref the
   operator named, and `state.json`/the audit record the base tip and any
   behind-count. (A missing upstream or an offline fetch is tolerated with a
   note, not a stop.)
4. Run `${CLAUDE_PLUGIN_ROOT}/tools/tanuki-preflight <plugin-worktree>`; on failure stop
   and report (lint is not the loop's job). **`tanuki-preflight` assumes the
   loop-repo is a Claude plugin** — its `skills-dir` check hard-fails a repo
   with no `skills/` and no `commands/*.md`. For a non-plugin dogfooding target
   (a plain code repo), that check does not apply: skip preflight rather than
   fabricating placeholder plugin scaffolding to satisfy it (F39). `tanuki-ledger
   --target <target> init` (idempotent).

## 1. Iterate (until convergence or the cap or a breaker)

Each iteration is bracketed by `tanuki-loop`; the model does only steps 3–4
(judgment). Sequencing is guarded by the tool — an out-of-order subcommand
(e.g. a second `iter-start` on an unclosed iteration) is itself a breaker, so
a skipped check can never silently pass. Begin with `tanuki-loop --target
<target> iter-start` — it trips the cap / wall-time / external-modification
breakers, records the start SHA, and snapshots the ledger (exit 3 +
`{"breaker": …}` stops the run). Then:

1. **Drive** Tanuki against the **integration-branch state**: `tanuki-drive
   --plugin <loop-repo> --plugin-ref <integration-branch> … --run <run>-iterN`,
   so this iteration dogfoods the changes prior iterations landed (`--plugin-ref`
   clones the integration commit, not the repo's checked-out branch).
   **Host fixture (spec-host-snapshot).** For a hosted target, pass
   `--host <host_fixture_path from state.json>` — the run-scoped fixture `init`
   pinned — **never the scenarios file's live host path**. Every iteration
   then dogfoods the identical host, and operator activity in the real host
   during the run is out of scope by construction. Hostless targets pass no
   `--host` (issue #13). The live host is a clone source at `init` and nothing
   else, ever.
   **Scenario selection (deterministic — supersedes hand-derived rotation).**
   Each iteration's scenario set comes from `tanuki-scheduler --target <t>
   plan --scenarios <file> --run <run>-iterN` (`sync` first if the matrix
   changed; `--run` ties the persisted plan record to the run): verify set →
   exploration quota (unexplored branches — the anti-Goodhart guard) → active
   rotation → due regression members. Drive the planned subset via `--only`.
   **This plan is pre-approved — never present an execution-confirmation ("run
   all these scenarios?") gate.** The attended `/tanuki` plan gate
   (`${CLAUDE_PLUGIN_ROOT}/docs/tanuki-spec.md`) is the one piece of the reused
   pipeline that does **not** apply here: the scheduler already chose the set,
   so the drive starts immediately at every phase. If judgment surfaces, defer
   it to the morning gate — never prompt mid-run.
   After mining (step 2) run `tanuki-scheduler record-run --run <run>-iterN` —
   demotion/promotion is the tool's arithmetic. When the plan reports an
   empty unexplored pool across the whole run, note it in the audit: the
   morning gate should offer a generation pass (new charters, plan-gated —
   frontier work, but attended). Record the plan + its `quota_met` in the
   audit (`tanuki-loop log --audit "iterN plan: …"`).
   While the drive runs in the background, follow /tanuki's
   **background-liveness rule**: read `<run-dir>/progress.json` on a periodic
   check (~2–3 min) and relay one compact `[done/total] …` line per update —
   never a bare "waiting" turn. In a supervised (Phase 1) run this is the
   operator's window into the loop; unattended, the same lines go to the
   audit trail.
2. **Mine.** Ingest events; run extraction + frontier dedupe into the ledger
   (`${CLAUDE_PLUGIN_ROOT}/docs/tanuki-spec.md` §2). Recurrence updates as normal.
3. **Classify (internal — forked rules copied from the operator's attended
   triage command; no GitHub).**
   Altitude test: **spec** = alters an invariant others depend on; **story** =
   a plan across files; **direct** = small, self-evident. Bias **toward
   implementation** — `direct`/`story` → work list. **Defer only** when a fix
   is physically impossible, self-contradictory, or blocked by missing
   credentials/config — plus (iteration ≥ 2 / headless) a `spec` decision the
   run policy can't answer. Iteration 1 may ask the operator and MUST record
   each answer via `tanuki-loop policy --finding <Fid> --decision "…"` (the
   mechanism behind "questions only in iteration 1" — an unrecorded answer
   is a policy gap iteration 2 will defer on); iterations ≥ 2 never ask — an
   unanswered judgment defers. Record every defer with its reason.
4. **Implement** each work-list item on the integration worktree from the
   finding text (forked implement rules; no issue, no PR). Before implementing
   a finding call `tanuki-loop attempt --finding <Fid>`; if it returns
   `frozen: true` (past the attempt cap) skip it — already queued for the
   morning.
5. **Test / verify** — `tanuki-loop test` runs the configured `test_cmd` in
   the worktree; failure → immediate-stop breaker. If none is configured it
   reports skipped and the model verifies manually (always configure one for
   an unattended run).
6. **Commit** with forked commit-grouping logic — intent-scoped commits on the
   integration branch (no push, no PR). **Never `git add -A` blindly**: stage
   the work product, not whatever the runtime regenerated. Then
   `tanuki-loop iter-verify` (`--no-patch` when no change was expected) — the
   four-part integration invariant; it records the end SHA and breaks on any
   violation.
   **Build-artifact guard** (issue #71, F64/F102; arbitration 2026-07-17):
   regenerable output (`__pycache__/`, `.pytest_cache/`, `.mypy_cache/`,
   `.ruff_cache/`, `*.pyc`, `*.pyo`, `*.egg-info`) never counts as a dirty
   worktree — it is not work product, so it adds no friction to a clean
   bracket. When it is **committed**, `iter-verify` **reports** it: the paths
   and the cause appear in its output (`build_artifacts_committed`,
   `warning`), in the audit trail, and in `status` — which is what you read
   before approving the merge. It is **not a breaker**: halting here would add
   a fifth halt to the four-part integration invariant above and would stop an
   unattended run over a merely-incomplete `.gitignore` — a contract change
   that is the operator's call, not a headless run's. Enforcement lives at the
   gate instead: **`gate-push` refuses** to push a merge that adds artifact
   paths (`--allow-artifacts` overrides deliberately), so nothing regenerable
   reaches the remote while nothing overnight halts over it.
7. Append the iteration to the audit artifact (start/end SHA, findings
   bumped/new, items implemented, items deferred). Loop back to step 1.

If an iteration fails after step 4, run `tanuki-loop rollback` (`reset --hard`
to the start SHA + `git clean -fd` + clean-tree verify, removed paths logged),
so untracked files don't leak forward and prior successes are untouched, then
apply the breaker. **A rolled-back iteration still counts toward the cap** —
`iter-start` records it and `rollback` does not give the attempt back, so a run
with rollbacks reaches the "iteration cap reached" breaker sooner than the raw
count of *landed* iterations suggests (F27).

**Convergence (stop early — the cap is only a ceiling).** After each closed
iteration call `tanuki-loop record-cycle` (no counts — the tool computes
new-actionable from the `iter-start` ledger snapshot minus frozen/deferred,
and "patched" from the iteration SHAs; convergence never trusts a
model-supplied count). A cycle is **quiet** when there are no new actionable
findings and no accepted patch — **and it counts toward the streak only if
its scheduler plan reported `quota_met: true`**, which `record-cycle`
enforces itself by reading the iteration's persisted plan record (a run that
skipped exploration while unexplored branches remained cannot manufacture
convergence; spec-tanuki-scenario-lifecycle's loop amendment). **Two
consecutive quiet cycles** (the tool reports `converged: true` at streak ≥ 2)
end the run — one quiet drive can be luck. A finding may be re-fixed up to its attempt cap (default 4), then it is
frozen (via `tanuki-loop attempt`); a frozen finding is not a new actionable
one.

**Breakers.**
- *Immediate stop* (freeze integration branch, write audit, report): test
  failure · implementation-command failure · commit failure · integration
  invariant violation (the four-part check) · iteration cap reached ·
  wall-time exceeded · token budget exceeded · external modification of the
  integration worktree.
- *Defer / freeze, keep going*: a fix physically impossible,
  self-contradictory, or blocked by missing credentials/config → **defer**; a
  `spec` decision with no run-policy answer → **defer**; a finding past its
  attempt cap → **freeze** (that finding only). Keep processing other
  independent actionable findings; stop **only** if a deferred/frozen item
  blocks *all* remaining work.
- *Recovery from the external-modification breaker* (attended;
  spec-tanuki-loop "Circuit breakers"): never `rollback` — it needs an open
  iteration and the breaker fires with all iterations closed. The operator
  runs `tanuki-loop recover` and explicitly chooses `--restore` (reset the
  worktree to the last verified `head_expected`, discarding external commits;
  audited, reflog-recoverable) or `--adopt` (re-baseline: accept the current
  HEAD as `head_expected`; adopted range audited and surfaced in the
  morning-gate diff). Closed iterations are never mutated; nothing recovers
  automatically.

## 2. Morning gate (attended — invariant in every phase)

Do not end at "here is what ran." Present, for one review:
- the **integration branch diff** (the relocated Human Gate — `git diff
  <base SHA>..<integration HEAD>`),
- the **morning review queue** (deferred spec / judgment items, each with its
  reason), and
- the **audit artifact** (per-iteration SHAs, auto-decisions, convergence or
  breaker reason).

Then, behind the operator's single approval, run **merge-first and idempotent**
— nothing outward-facing until the merge is a fact:
1. **Final tests** on integration HEAD — `tanuki-loop test` (the same
   configured `test_cmd` each iteration runs); abort the gate on failure.
2. **Merge `integration → main`** — a plain `git merge --no-ff
   <integration-branch>` on `main` (there is no `tanuki-loop merge` subcommand;
   this one gate step is a hand-run git operation). Use `--no-ff` so the batch
   lands as one reviewable merge commit that `gate-check` can then confirm is
   reachable; executed by the gate only after approval, never unattended (F12).
3. **Verify** with `tanuki-loop gate-check` (integration HEAD reachable from
   base).
4. **Push the base branch** with `tanuki-loop gate-push` — the first
   outward-facing step, run **before** any issue is materialized. Until the
   merge commit exists on the remote, the commit links step 5 stamps onto
   issues 404 and a local-only `main` diverges the remote for the next
   workflow's `git pull --ff-only`. `gate-push` fetches first and, if the
   remote has moved, **refuses rather than force-pushing** (exit 3,
   `diverged: true`): reconcile the remote in, re-run the final tests, and
   re-run `gate-push`. It **also refuses when the merged diff adds build
   artifacts** (exit 3, `build_artifacts: […]`) — regenerable output must not
   reach the remote, and this is the last mechanical boundary before it does
   (F102). Remove them (`git rm -r --cached <path>`), add them to
   `.gitignore`, commit, and re-run; `--allow-artifacts` overrides
   deliberately. `status`'s `warning` field names these paths before you get
   here — read it when deciding the merge. Only past a successful push does
   anything else outward-facing run.
5. **Materialize** issues — **one per resolved problem** (keyed by the lead
   ledger finding id, or a `+`-joined set), describing **what landed**, each
   body stamped `tanuki-loop: <run-id>/<problem-key>`. Before creating,
   `tanuki-loop issue-get --key <problem-key>` (and a marker search) returns
   any existing issue; create only the missing ones and record each with
   `tanuki-loop issue-put --key <problem-key> --issue <n>` — a mid-gate death
   re-runs from here without duplicating. **Steps 5–6 apply only where an issue
   tracker is configured** (as with the board reconciliation in step 6): for a
   hostless/trackerless target the landed batch is already recorded in the merge
   commit and the audit trail, so skip issue materialization entirely rather
   than inventing a substitute (F15).
6. **Link** each to the (now pushed) merge commit, **close as completed**,
   reconcile the board to Done (where project-board tooling is configured).
7. Remove the loop worktree (`git worktree remove`), then close the run
   machine-readably: `tanuki-loop finish --reason gate-ratified` (likewise
   `converged` / `cap` / `aborted` when a run ends without a gate). **`finish`
   owns the branch cleanup** — its `branch_cleanup` report has three buckets:
   `deleted` (tip reachable from the base branch), `kept` (unmerged tips stay),
   and `skipped` (e.g. a branch still checked out in a worktree — "remove it
   first"). The next `init` for the target is the backstop. No branch is left
   to accumulate until an unrelated tool collects it.

## Monitoring (the operator's window)

`${CLAUDE_PLUGIN_ROOT}/tools/tanuki-loop --target <target> dashboard` renders one screen
of **operational status, not internal state** — within seconds the operator
can answer: is the loop healthy, did anything unexpected happen, what did
this run change, is the scheduler behaving as intended, and what decision
comes next. Sections, in order — **always rendered** (a section whose
substrate is missing degrades to an explicit one-line reason, never a silent
omission; spec-tanuki-loop "Fixed skeleton"):
- **health** — one OK / ATTENTION / DONE verdict with the reason (breaker,
  unmatched anomaly, cap-without-convergence, pending gate decisions).
- **latest drive** — live progress plus per-scenario results, every
  anomalous result classified against the ledger: `known: F2 deferred, …`
  (expected — already ruled on) vs `UNMATCHED` (new problem or mining not
  done — the only class that needs eyes mid-run).
- **this run** — run-scoped deltas (new / fixed / recurred findings,
  deferred/frozen with their reasons inline); cumulative lifetime counts are
  a single pointer line to `--history`, where they belong.
- **scheduler decisions** — from the iteration's persisted plan: what is
  being verified (replaying accepted fixes), what was picked for exploration,
  the active rotation, what is waiting for future iterations, and whether the
  exploration quota was met (runs predating persisted plans degrade to
  per-state pools with a note).
- **convergence** — the definition spelled out (no new actionable finding +
  no patch + quota met), which conditions held last cycle, and what is still
  required (`one more quiet cycle ends the run`).
- **why stopped / NEXT** — a breaker points at the audit trail; a
  converged/capped run lists the morning gate's concrete decisions (review
  the N-commit diff, rule on the deferred/frozen items, approve the merge).
Add `--follow 10` for a self-refreshing terminal view during an unattended
run — it reads only state files, so it's always safe to run alongside the
loop.
The dashboard is the live view; for the cross-run long view (per-scenario
execution history, transitions, coverage, selection history) use
`tanuki-scheduler --target <t> history` (surfaced as `/tanuki <t> history`).

The deferred queue is handed back for a normal attended triage sitting —
the loop never decides a spec alternative on its own.

## Rules

- Never write `main` during iterations; never touch the operator's normal
  working tree; never create GitHub issues / labels / PRs / story files during
  iterations. Outward-facing artifacts are written only at the morning gate,
  after the merge, describing what landed.
- **The live host is out of scope after `init` (spec-host-snapshot).** `init`
  pins a run-scoped host fixture; from then on the run reads, diffs, and
  drives only that fixture and the disposable clones under the run dir — the
  real host tree is **never** inspected, diffed, written, or "restored". If
  the loop believes the real host changed, that belief is out of its
  jurisdiction: the operator's repo is the operator's, and remediation of
  real trees is not a loop capability. Anomaly checks compare fixture against
  fixture (did the drive leave a footprint in *its* workspace?); live-host
  drift is a read-only informational note at `finish`, never a breaker. The
  skill never inspects or modifies any path outside the run dir and the loop
  worktree after init.
- The ledger is the source of truth overnight.
- Cumulative on one integration branch; never fan out per-iteration branches
  from `main`.
- Rollback is `reset --hard <start SHA>` **plus** `git clean -fd` in the loop
  worktree only, recorded in the audit; prior successes are untouched.
- Questions only in iteration 1 (recorded as run policy); iterations ≥ 2 never
  ask — unanswered judgment defers. A finding is re-fixed up to its attempt
  cap (default 4), then frozen — never a whole-loop stop.
- `integration → main` runs only after explicit approval, merge-first and
  idempotent, and is **pushed to the remote before any issue is materialized**
  (`gate-push`, refusing a diverged remote rather than force-pushing) — a
  local-only merge leaves dangling issue links and a diverged remote. Phases
  differ only in supervision and cap — one code path, Phase-3-ready from the
  first run.
- Merged integration branches are deleted by `tanuki-loop finish` (backstop:
  the next `init`); an unmerged tip is never deleted.
