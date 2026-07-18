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
- `<target> reconcile [branch…]`: **no driving** — audit the loop's own
  unmerged integration branches and produce a landing plan (see
  "Reconcile" below). A bare word because it does not drive
  (spec-short-command-surface D6). Read-only until an explicit gate.
- `<target> unresolved`: **no driving, read-only** — the reconcile pass's
  discovery, on its own: which integration branches never merged, how stale
  they are, and the mechanical signals about each commit. Run
  `${CLAUDE_PLUGIN_ROOT}/tools/tanuki-loop --target <target> unresolved
  [--json]` and show it. It emits **signals, never verdicts** — see
  "Reconcile". Also rewrites `~/.tanuki/<target>/unresolved.md`, the
  stable-path brief to open cold.

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
   never a bare "waiting" turn. Under `drive_concurrency > 1` the relay line
   reflects the aggregate from the progress file's `running` list — e.g.
   `[3/6] two running: s2 (80s), s5 (40s)` — always sourced from the
   substrate, never re-derived (story 1.22). In a supervised (Phase 1) run
   this is the operator's window into the loop; unattended, the same lines
   go to the audit trail.
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
  spec-tanuki-loop "Circuit breakers"): **detection is deferred** — the breaker
  fires at the *next* `iter-start` (which compares the worktree HEAD against
  the last verified `head_expected`), not immediately when the worktree is
  touched and not on `iter-verify` (F41). So a between-iterations external edit
  surfaces only when the loop tries to open the following iteration. Recovery:
  never `rollback` — it needs an open iteration and the breaker fires with all
  iterations closed. The operator
  runs `tanuki-loop recover` and explicitly chooses `--restore` (reset the
  worktree to the last verified `head_expected`, discarding external commits;
  audited, reflog-recoverable) or `--adopt` (re-baseline: accept the current
  HEAD as `head_expected`; adopted range audited and surfaced in the
  morning-gate diff). Closed iterations are never mutated; nothing recovers
  automatically.

## 2. Morning gate (attended — invariant in every phase)

**Every `main` in this section is a placeholder** (F100/F103): the base
branch's real name is `init`'s labeled `base` field (it may be `master` or
anything else) — take `base`, `base_sha`, and `integration_branch` from
`init`'s output before running any snippet below.

Do not end at "here is what ran." Present, for one review:
- the **integration branch diff** (the relocated Human Gate). Copy-pasteable,
  using the `base_sha` and `integration_branch` that `init` printed:
  ```
  git diff <base_sha>..<integration_branch>        # two dots — what the loop wrote
  git log --oneline <base_sha>..<integration_branch>
  ```
  Use **two dots**, not three: `A..B` is "what B has that A doesn't" — exactly
  the loop's work. `A...B` diffs against the merge-base instead, which silently
  hides anything that landed on `main` since the run started (F95).
  Run it from **either the loop worktree or the operator's normal checkout** —
  both share one object store and ref namespace, so the integration branch
  resolves identically from each; reading the diff from the normal checkout is
  safe and intended, and does **not** touch the worktree. (`git diff` here is
  read-only — the "never touch the operator's working tree" rule bars *writes*,
  not reads.)
- the **morning review queue** (deferred spec / judgment items, each with its
  reason), and
- the **audit artifact** (per-iteration SHAs, auto-decisions, convergence or
  breaker reason).
`init` and `finish` both print an `artifacts` block with the absolute paths to
`state.json` / `audit.md` / `queue.md` (under `~/.tanuki/<target>/loop/<run>/`)
— take them from there rather than hunting the run dir.

Then, behind the operator's single approval, run **merge-first and idempotent**
— nothing outward-facing until the merge is a fact:
1. **Final tests** on integration HEAD — `tanuki-loop test` (the same
   configured `test_cmd` each iteration runs); abort the gate on failure.
   **Running `test` here is expected even though the run is already closed**
   (F101): the overnight run finished with `cap` or `converged` before you sat
   down, and gate steps still work on a finished run — `test` echoes a
   `run_finished` block naming the close reason, and appends its result to the
   closed run's audit. That block is a confirmation, not a warning.
2. **Merge `integration → main`** — a plain `git merge --no-ff
   <integration-branch>` on `main` (there is no `tanuki-loop merge` subcommand;
   this one gate step is a hand-run git operation).
   **If the merge conflicts** — the base moved under the run and a hunk
   overlaps — this hand-run step has no tool recovery, unlike the others: abort
   with `git merge --abort` to return to a clean base, reconcile the divergence
   (rebase or re-run the loop on the fresh base), or resolve the conflict by
   hand and `git commit` the merge. Do **not** push a half-merged tree; the
   later `gate-push` divergence guard is not a substitute for a clean merge
   here (F152).
   **Check out the base branch first, and use its real name.** The merge runs
   in the operator's normal checkout and lands on whatever branch is currently
   checked out — confirm where you are with `git branch --show-current`, then
   `git checkout <base>` before merging, or the batch lands somewhere
   unintended (F137). The base is **not** assumed to be `main`: take the
   actual name from `init`'s `base` / `base_upstream` output (it may be
   `master` or anything else); the `main` in these snippets is a placeholder
   (F100, F103).
   **Look at `status`'s `warning` field before approving.** If the loop's
   commits swept build artifacts (`__pycache__`, `*.pyc`) onto the integration
   branch, `iter-verify` recorded them and `status` names them — they reach the
   remote at step 4 unless removed now (F102). Use `--no-ff` so the batch
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
5. **Materialize** issues — *skip steps 5–6 entirely if no issue tracker is
   configured (e.g. a hostless target); the merge commit and audit trail are
   already the record* — **one per resolved problem** (keyed by the lead
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
   *(Skipped along with step 5 when no tracker is configured.)*
7. Remove the loop worktree (`git worktree remove` — if it refuses with a
   modified-file error naming regenerable output, see the note at the end of
   this step). **This worktree removal is the terminal action of the attended
   gate — once the merge is pushed (step 4) and the worktree is gone, you are
   done.** **The one rule about `finish` (F136/F149/F151): it is never
   required here, and running it after the merge landed is harmless.** Nothing
   in this workflow invokes it; skipping it loses nothing; running it anyway
   simply deletes the merged integration branch immediately (a repo-state
   change — the branch would otherwise wait for the next `init` sweep) and
   returns `ok`. That is the whole decision — optional, not forbidden, not
   unsafe. Why no closing command is needed (owner ruling 2026-07-17): the
   merge's reachability from the base *is* the settlement, and the surfaces
   that need it derive it — `status` reports it, the next `init`'s
   merged-branch sweep deletes the integration branch (tip reachable from the
   base; an unmerged tip is never deleted), and `/repo-cleanup` owns any
   earlier branch/worktree removal. `finish --reason gate-ratified` remains
   accepted for compatibility only, and invoked without a verifiably landed
   delivery it refuses.
   **What a repeated `finish` reports** (F108): it echoes `previously_finished`
   with the earlier close's reason and timestamp rather than overwriting it
   silently, so a double close is visible and deliberate.
   Likewise `gate-check` after the branch is deleted reports `branch_deleted`
   with a `note`: a deleted branch is the *success* state at that point, not a
   missing one.
   **Artifacts already committed to the base can block the removal** (F111).
   The build-artifact guard above concerns output *the loop's own commits*
   swept in; this is the other direction. If regenerable output (`*.pyc`,
   `__pycache__/`) was committed to the base branch *before* the run, running
   `test_cmd` regenerates it in the worktree, the tracked content then differs
   from the index, and `git worktree remove` refuses — at the last gate step,
   naming a file that has nothing to do with the loop's work. It is not a
   breaker and nothing is wrong with the batch: use `git worktree remove
   --force`, or clean the paths first. The durable fix is to stop tracking
   them on the base (`git rm -r --cached <path>` + `.gitignore`), which also
   removes the `gate-push` artifact refusal for every later run.

### PR-protected targets (`"gate": "pr"` in the loop block)

When the base branch refuses direct pushes (required-check protection),
steps 2–4 above cannot run: there is no local merge to push. Set
`"gate": "pr"` in the scenarios `loop` block (`doctor` then validates `gh`
availability up front) and the gate reshapes:

- **Overnight close delivers the review material.** After the run finishes
  (`cap`/`converged`) and `tanuki-loop test` passes on the integration HEAD,
  run `tanuki-loop gate-pr`: it pushes the **integration branch** (never
  forced) and opens **one Draft PR** `integration → base`. That Draft PR *is*
  the morning-gate presentation — diff, grouped commits, run summary — moved
  onto the forge. It is delivery, **not ratification**; the loop never
  merges, in any phase.
- **The Human Gate is PR approval + merge.** Review the PR as you would the
  step-1 diff; the intended merge method is **"Create a merge commit"**, so
  the loop's intent-scoped commit groups stay visible in the base's history —
  don't squash.
- **`gate-pr` is idempotent and failure-safe.** A re-run reuses the existing
  PR (recorded in state, or found on the forge after a crash) and never
  duplicates; a failed push or PR create exits 3 with the local integration
  branch and run state intact — fix the cause and re-run.
- **The loop ends at delivery** (owner ruling 2026-07-17). `gate-pr` records
  the run as delivered — PR number, integration-tip SHA, base SHA — and that
  is the loop's terminal fact. It does not poll, merge, comment, or wait, and
  **you never run a closing command**: `status`, `init`, and `unresolved`
  derive the settlement from the forge and the current base when they need
  it — merged + reachable → `landed`; PR open → `pending`; closed without
  merge → `declined`; unverifiable → `unknown`, never an optimistic default.
  Offline, the last observed result is shown with its timestamp, marked
  stale.
- **Cleanup follows proof, not ceremony.** `/repo-cleanup` owns branch and
  worktree removal (the next `init`'s sweep is the backstop) and deletes only
  after reachability proves the work landed or an attended decision discards
  it — so nothing is deleted from under an open PR. Finding verification and
  recurrence accounting treat a change as landed **only** when its delivered
  SHA is reachable from the current base. Steps 5–6 (issues, where a tracker
  is configured) still follow the merge.

## Monitoring (the operator's window)

`${CLAUDE_PLUGIN_ROOT}/tools/tanuki-loop --target <target> dashboard` renders one screen
of **operational status, not internal state** — within seconds the operator
can answer: is the loop healthy, did anything unexpected happen, what did
this run change, is the scheduler behaving as intended, and what decision
comes next. Sections, in order — **always rendered** (a section whose
substrate is missing degrades to an explicit one-line reason, never a silent
omission; spec-tanuki-loop "Fixed skeleton").

**Empty states are enumerated, not written** (`specs/spec-tanuki-view/SPEC.md`
D3; issue #66). A section that renders empty resolves to exactly one member of
a closed set in `tools/tanuki-loop` (`EMPTY_STATES`), each carrying whether
the emptiness is **expected** for this target's state or a **GAP** the
operator should close, plus the command that changes it — and each with its
own fixture in `tools/tests/test-loop-empty-states`. The same absent
substrate can be either: no scenario matrix is *expected* on a fresh target
and a *GAP* once iterations have run. Adding an empty state means adding a
member (an unenumerated one raises); never write a new reason at a call site.

Sections:
- **health** — one OK / ATTENTION / DONE verdict with the reason (breaker,
  unmatched anomaly, cap-without-convergence, pending gate decisions).
- **latest drive** — live progress plus per-scenario results, every
  anomalous result classified against the ledger: `known: F2 deferred, …`
  (expected — already ruled on) vs `UNMATCHED` (new problem or mining not
  done — the only class that needs eyes mid-run).
- **this run** — run-scoped deltas (newly observed / status→accepted /
  recurred findings, deferred/frozen with their reasons inline); every label
  is a ledger fact, never a commit count (`fixed` was retired — it read as
  one); cumulative lifetime counts are a single pointer line to `--history`,
  where they belong.
- **scheduler decisions** — from the iteration's persisted plan: what is
  being verified (replaying accepted fixes), what was picked for exploration,
  the active rotation, what is waiting for future iterations, and whether the
  exploration quota was met (runs predating persisted plans degrade to
  per-state pools with a note).
- **convergence** — the definition spelled out (no new actionable finding +
  no patch commit produced + quota met), which conditions held last cycle,
  and what is still required (`one more quiet cycle ends the run`).
- **why stopped / NEXT** — two separated facts (spec-tanuki-loop "Stop
  reason vs delivery settlement"): the **computational stop reason**
  (cap | converged | breaker | cancelled — never a settlement) and, on its
  own line, the **delivery with its derived settlement** (landed | pending |
  declined | unknown, with the PR when one exists — the delivery-boundary
  ruling's derivation, stale-marked when served from cache). A breaker
  points at the audit trail; a converged/capped run lists the morning
  gate's concrete decisions (review the N-commit diff, rule on the
  deferred/frozen items, approve the merge).
The header's elapsed figure is **execution time**: it freezes at the run's
first close, and time waiting for PR review or merge never accrues to it.
Add `--follow 10` for a self-refreshing terminal view during an unattended
run — it reads only state files, so it's always safe to run alongside the
loop. Add `--live` (the substrate of `/tanuki <t> view live`) to render
only while the run is active: a closed run degrades to the typed empty
state plus one historical line, never the full dashboard.
The dashboard is the live view; for the cross-run long view (per-scenario
execution history, transitions, coverage, selection history) use
`tanuki-scheduler --target <t> history` (surfaced as `/tanuki <t> history`).

The deferred queue is handed back for a normal attended triage sitting —
the loop never decides a spec alternative on its own.

## Reconcile (`/tanuki-loop <target> reconcile [branch…]`)

The morning gate ends at a **merge**. When the operator declines it — or
merges some runs and not others — the integration branch survives with real
work on it and no destination. It then rots: `main` moves, the operator
re-derives the same fixes by hand, and the branch's version of a finding and
`main`'s version drift into **contradiction**. That is not hypothetical. On
2026-07-17 two loop branches held 25 unmerged commits; reconciling them found
that the loop and the attended sitting had resolved the same finding in
opposite directions **twice** (F102/artifacts, F25→F99/compacted), and both
had to be arbitrated in the spec lane after both had shipped.

`reconcile` is the sitting that pays that debt down, and it is **attended,
report-first, and hunk-level**.

### The unit of classification is a semantic change unit, not a commit

**A commit is not a verdict.** One commit routinely contains hunks with four
different verdicts, and its subject line describes at most one of them:

- `8f4d556` "render the module docstrings in `--help`" — its `tanuki-ledger`
  hunk was not a docstring at all. It carried the compacted-bucket filter
  from the commit beneath it, a design that had been **rejected**. Cherry-
  picking the commit on its title silently reintroduced it; only a test
  caught it.
- `76a6e4f` bundled three separable things: a rejected design, a fix that had
  already been re-implemented independently, and a genuinely good docs change
  that existed **nowhere else**. Skipping the commit lost the third; taking it
  smuggled the first.

So: **never infer intent from the subject line, the commit message, or the
finding ids it cites.** Read the diff. Classify each hunk.

Per-hunk verdicts (closed set):
- **already-landed** — `main` has this behavior. Semantic, never `git cherry`:
  on 2026-07-17 all 25 commits were patch-non-equivalent and several were
  already landed. Compare *behavior*, not patch text.
- **superseded** — `main` solves the same finding differently. Say which is
  better and why; sometimes `main`'s is (its F26 ignores a disagreeing
  override where the branch honored it — stricter, and the documented rule
  favors it). Sometimes the branch's is: that is what #87 concluded.
- **conflicting** — contradicts a ratified spec or decision. **Never resolved
  here.** Report it as a decision the operator owes, with the governing text
  quoted and the file pointer — a conflict is a decision, not a merge. Where
  that decision gets taken is the operator's business, not this command's
  (spec-short-command-surface, Non-goals: no downstream awareness).
- **still-applicable** — real, absent from `main`, no contract in the way.
- **unrelated-but-worth-preserving** — orphaned work worth keeping, no home
  yet. Report it; never land it silently.

### Steps

1. **Collect** (read-only) — **`tanuki-loop --target <t> unresolved --json`**,
   never re-derived by hand. It returns the branches with unique commits and
   no merged tip, their staleness (ahead / behind / days since last commit),
   and per commit: files touched, whether it **cherry-picks cleanly**, and the
   ledger findings its message cites **paired with their current status**.
   Explicit `branch…` arguments narrow the set.
   Those are **signals**. `cherry_picks_cleanly` is a fact about git and is
   **not** a synonym for applicable — `8f4d556` applied perfectly and carried
   a rejected design. A cited finding that is now accepted-and-tombstoned is a
   superseded *candidate*, not a verdict. The tool refuses to emit a verdict
   field precisely so this step cannot outsource step 2.
2. **Classify** every hunk of every commit against **current `main`**, into
   the verdicts above. This is judgment and stays at the command layer (the
   routing principle: deterministic work → tools, judgment → here). Cite the
   evidence per verdict — the code in `main` that makes it already-landed, the
   spec text that makes it conflicting.
3. **Report + landing plan** — the default output, and the whole point.
   Per branch: staleness; per commit: its hunks' verdicts and whether the
   commit is **atomic** (every hunk one verdict, applies cleanly) or
   **entangled**. Then the plan, in execution order, one row per action:
   cherry-pick / hand-port / route-to-spec-lane / skip-superseded / report.
   Stop here unless the operator opens the gate.
4. **Gate — one explicit confirmation, per plan, before any mutation.** No
   per-item questions, no partial auto-apply. Declining leaves the repository
   untouched; the report is still the deliverable.
5. **Execute**, in plan order:
   - **cherry-pick only an atomic, semantically-applicable commit** (`-x`, so
     provenance survives). Atomic means every hunk shares one verdict *and*
     it applies without conflict. Anything else is not a cherry-pick candidate
     — this rule exists because `8f4d556` looked like one.
   - **hand-port entangled changes**: apply the applicable hunks only, by
     hand, dropping the rest. Name in the commit message what was dropped and
     why — an entangled commit's good half is invisible otherwise.
   - **route conflicts**: nothing is landed. The plan row states the decision
     the operator owes, with the quoted contract text — never merge a
     conflicting hunk to "fix it later", and never take the decision here.
   - Run the suite after each landing group. A fixture that pins prose the
     branch legitimately improved is a fixture to update, not a reason to drop
     the change — say so in the plan.
6. **Report** what landed, what was hand-ported (and what was dropped from
   each), what was routed and where, what was skipped as superseded with the
   reason, and what remains report-only. A branch is **reconciled** when
   nothing unique remains; only then is it a `/repo-cleanup` deletion
   candidate — and that command, not this one, deletes it.

### Rules

- **Report-first.** Mutation happens only past the step-4 gate. There is no
  flag that skips it.
- **Never delete a branch or a worktree.** `/repo-cleanup` owns that, and it
  keeps a branch with unique commits as report-only precisely so this command
  can find it.
- **Never resolve a contract conflict here.** The spec lane owns that; this
  command's job is to *detect* it and hand it over with the quote.
- **Attended only.** No unattended phase reconciles: every verdict is
  judgment, and this command's whole purpose is to catch what an unattended
  run and an attended sitting decided differently.
- **The subject line is never evidence.** Neither are the finding ids in the
  message. Only the diff and `main` are.

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
  worktree only, recorded in the audit; prior successes are untouched. **A
  rolled-back iteration still consumes one of the `cap` slots** (F27) — the cap
  counts every *consumed* iteration, verified or rolled back, so a cap of N
  bounds total attempts, not just successful ones (visible in `status` /
  the dashboard's "N/N consumed" counter).
- Questions only in iteration 1 (recorded as run policy); iterations ≥ 2 never
  ask — unanswered judgment defers. A finding is re-fixed up to its attempt
  cap (default 4), then frozen — never a whole-loop stop.
- `integration → main` runs only after explicit approval, merge-first and
  idempotent, and is **pushed to the remote before any issue is materialized**
  (`gate-push`, refusing a diverged remote rather than force-pushing) — a
  local-only merge leaves dangling issue links and a diverged remote. On a
  PR-protected target (`"gate": "pr"`) the approval takes the form of PR
  approval + merge on the forge: the loop ends at delivering one Draft PR
  (`gate-pr` — review material, never ratification; owner ruling 2026-07-17),
  it never auto-merges, and settlement is derived by read-only surfaces from
  the forge and the current base — no human closing command. Phases
  differ only in supervision and cap — one code path, Phase-3-ready from the
  first run.
- Merged integration branches are deleted by `tanuki-loop finish` (backstop:
  the next `init`); an unmerged tip is never deleted.
