# Spec: /tanuki-loop — unattended cumulative dogfooding on an integration branch

Status: PROPOSED 2026-07-13. Implemented in
`commands/tanuki-loop.md`. Depends on the Tanuki pipeline
(`docs/tanuki-spec.md`) and the shared executables (`tanuki-loop`,
`tanuki-drive`, `tanuki-ledger`). Its classification and implementation
**judgment is an intentional fork** of `/triage-gh` and `/implement-story`:
the rules are copied into this command and allowed to diverge — the judgment
layer is not reused. Only the tools and the (issue-free) ledger are shared.

Origin: the attended cycle `tanuki → triage-gh → publish-issues →
implement-direct → implement-story → merge → commit-groups → tanuki` was
approved-at-every-gate for two days straight. The gates that mattered were the
few requiring judgment (a spec alternative, a missing project config, a botched
stacked merge); the rest were rubber-stamps on mechanical work. This command
automates the mechanical majority overnight so dogfooding iterates while the
operator sleeps.

## The relocated gate (the whole idea)

The Human Gate is **not removed — it is relocated.** It moves from N inline
"Approve" clicks to **one morning review** of everything the loop produced.
This keeps Tanuki's founding principle intact (`docs/tanuki-spec.md`:
proposals-only, decision support not autonomy): the loop **prepares**, the
human **ratifies**. Two things are therefore *never* automated, in any phase:
irreversible / outward-facing actions (GitHub issues, the merge to `main`) and
genuine design decisions (spec-touching work). Everything mechanical and
fully reversible runs unattended.

## Non-negotiables

- **Integration branch = the loop's temporary main.** A dedicated branch
  (`tanuki-loop/<target>/<YYYYMMDD-hhmmss>`) accumulates every iteration's
  commits. `main` is never written by the loop.
- **Cumulative, never fan-out.** Iterations stack on the *same* integration
  branch — iteration B builds on A's result. The loop never creates
  independent per-iteration branches from `main`.
- **Dedicated worktree.** The loop operates in its own `git worktree` (created
  outside the repo, e.g. under the session scratch or a sibling dir). The
  operator's normal working tree is never touched, read or written.
- **Issue-free overnight.** During the run the loop creates **no** GitHub
  issues, labels, projects, PRs, or story artifacts. The ledger is the single
  source of truth (see below).
- **`integration → main` happens only behind the operator's explicit
  approval — every phase, Phase 3 included.** The loop never merges
  unattended; once approved, the morning gate performs the merge, merge-first
  (below). No iteration writes `main`.
- **The scheduler plan is pre-approved — no plan gate in the loop.** The
  attended `/tanuki` "confirm execution before anything runs" plan gate
  (`docs/tanuki-spec.md`) **does not apply at any phase of the loop**. Once the
  target is initialized and `tanuki-scheduler plan` has produced the scenario
  set, that set is authoritative and the drive starts automatically — the
  operator never re-approves it. This is a corollary of the relocated gate:
  scenario selection is deterministic (spec-tanuki-scenario-lifecycle), so an
  execution-confirmation prompt is redundant and would block the unattended
  run. If genuine human judgment surfaces, it **defers to the morning gate**
  (per "Questions only in iteration 1" and the defer rules) — never an inline
  prompt during the run.
- **Questions only in iteration 1.** Iteration 1 may ask judgment questions and
  records the answers as the **run policy**. From iteration 2 on, the loop
  never asks and never stops for judgment — it applies the policy and
  auto-approves the mechanical path. A headless run supplies the policy up
  front (args, or a prior run's policy); absent a policy answer, the judgment
  item is deferred, never asked.
- **Fixed iteration cap, set before the run.** A safety ceiling, not the
  success condition (convergence is — see below).

## Issue-free by construction

The overnight loop is a **fork**: the classification and implementation rules
are copied from the attended commands into this command (see "One iteration",
steps 3–4/6) and run over the **ledger**, never their GitHub-issue plumbing.
The tool layer and the ledger are genuinely shared; the judgment layer is
copied and may diverge — a change to `/triage-gh` does not propagate here.
- Classification (direct / story / spec) runs against ledger findings — no
  `triage:*` labels, no story files, no issues.
- Implementation runs from the finding text directly onto the integration
  branch — no bound issue, no PR.
- Intent-scoped commits land on the integration branch — no push, no PR.

GitHub artifacts exist only *after* the morning approval (see "Morning gate").
This honors the "ledger-only overnight" decision: the ledger is authoritative
during the run; the outward-facing record is written once, describing what
actually landed.

## Deterministic substrate (`tools/tanuki-loop`) vs. judgment

Every scriptable safety check, invariant, and rollback lives in the
`tanuki-loop` executable so the model is left responsible **only** for
judgment (classify, implement) and running the target's tests. The tool owns:
worktree/branch/base-SHA setup (`init`), the per-iteration start gate —
cap / wall-time / external-modification breakers + start-SHA capture + a
ledger snapshot (`iter-start`), the four-part integration invariant + end-SHA
capture (`iter-verify`), rollback (`rollback` = `reset --hard` + `clean -fd` +
verify), the deterministic test runner (`test` — runs the per-target
`test_cmd` configured at init inside the worktree; failure is a breaker),
per-finding attempt counting and freezing (`attempt`), the two-quiet
convergence counter (`record-cycle`), the post-merge reachability check
(`gate-check`), and the materialization idempotency map (`issue-get`/
`issue-put`). Breakers are `tanuki-loop` exit code 3 with `{"breaker": …}`.

The tool also owns the operator surface: `policy` (records iteration-1
judgment answers — the mechanism behind "questions only in iteration 1"),
`finish` (machine-readable terminal reason: converged | cap | gate-ratified |
aborted; breakers persist `last_breaker` the same way), and `dashboard`
(one-screen **operational status** — see "Dashboard revision" below;
`--follow N` self-refreshes). Per-target run settings — `test_cmd`,
`iterations`, `wall_time_s`, `token_budget`, `attempt_cap` — live in the
scenarios file's `"loop"` block, read by `init --scenarios`, so the normal
invocation is just `/tanuki-loop` (CLI flags remain per-run overrides).

Two properties make the substrate trustworthy without trusting the model:
- **Sequencing guards.** Each iteration must move `started → (verified |
  rolled_back)` before the next `iter-start`, and `record-cycle` refuses an
  open iteration — a skipped or out-of-order check is itself a breaker,
  never a silent pass.
- **Computed accounting.** `record-cycle` derives new-actionable from the
  `iter-start` ledger snapshot (new ids minus frozen/deferred) and "patched"
  from the iteration's start/end SHAs; explicit flags exist as overrides but
  convergence never depends on a model-supplied count.

**Run policy (`policy.json`).** Iteration-1 judgment answers, keyed by **ledger
finding id** (stable across iterations because dedupe bumps the same id):
`{"run": "<run-id>", "decisions": {"<Fid>": {"classification":
"spec|story|direct", "disposition": "defer" | "<chosen resolution>",
"iter": 1}}}`. From iteration 2 the loop consults this map and never asks; an
absent answer for a genuine `spec` decision is a defer, not a question.

## Dashboard revision (REVISED 2026-07-14 — operational status, not internal state)

Operator feedback from the first unattended run: the dashboard exposed
internal state (cumulative counters, raw scenario statuses, `quiet streak
1/2`, `NEXT: morning gate`) that required too much interpretation — it could
not answer, within seconds, *is the loop healthy, did anything unexpected
happen, is the scheduler behaving as intended, what will I have to decide?*
The dashboard's contract is therefore restated: **it classifies and
explains; it never presents a raw number or status the operator must join
against other state by hand.** Concretely:

1. **Health verdict first.** One line — `OK` / `ATTENTION` / `DONE` — with
   the reason (breaker, unmatched anomaly, cap-without-convergence, pending
   gate decisions). Everything below it is detail.
2. **Anomalies are classified, never bare.** A non-ok scenario result (e.g.
   `timeout after 900s`) is joined against the ledger's `scenarios` links
   and the run's deferred/frozen sets and tagged `known: F2 deferred, …`
   (already ruled on — expected) or `UNMATCHED` (no ledger finding yet — the
   only class that warrants mid-run attention, and an ATTENTION trigger).
3. **Run-scoped deltas, not cumulative counters.** The live view reports what
   *this run* changed — new / fixed / recurred findings, deferred/frozen with
   reasons inline — computed from the run-start ledger snapshot (`iter-start`
   now snapshots `{id: {status, recurrence}}`, not bare ids). Lifetime totals
   are one pointer line to the history view (`tanuki-scheduler history`),
   which owns the long view.
4. **Scheduler decisions, not pool sizes.** From the iteration's persisted
   plan record (spec-tanuki-scenario-lifecycle): the verify set, the
   exploration pick, the active rotation, what waits for future iterations,
   and `quota_met`. Runs predating persisted plans degrade to per-state pools
   with an explicit note — never silently.
5. **Convergence is explained.** The quiet-cycle definition is spelled out
   (no new actionable finding + no patch + exploration quota met), with which
   conditions held last cycle (`record-cycle` persists `last_cycle`) and what
   is still required.
6. **NEXT is a decision list.** A stopped run enumerates the morning gate's
   actual asks — review the N-commit diff, rule on each deferred/frozen item
   (reason shown), approve the merge — not a location name.

Enforcement note (same revision): `record-cycle` now *enforces* the loop
amendment itself — quiet requires the iteration's persisted plan to report
`quota_met: true` (absent plan record ⇒ ungated, for pre-scheduler runs).
Previously the amendment was documented but the tool did not check it.

**Materialization key.** The morning gate stamps each issue body with
`tanuki-loop: <run-id>/<problem-key>`, where `<problem-key>` is the lead ledger
finding id for the resolved problem (or a `+`-joined sorted set when several
findings collapse into one). `tanuki-loop`'s `materialized` map records
`{problem-key → issue}`; a retry checks it (and the on-issue marker) before
creating, so a mid-gate death never duplicates an issue.

## One iteration

Each iteration is bracketed by `tanuki-loop`: `iter-start` records the start
SHA and trips the cap / wall-time / external-modification breakers before any
work; `iter-verify` runs the integration invariant and records the end SHA
after the commit; `rollback` (on failure) restores the start SHA — `git reset
--hard` **and** `git clean -fd`, removed paths recorded, tree re-verified clean
— **inside the loop worktree only**, so untracked files cannot leak forward and
prior successful iterations are untouched.

1. **Drive.** Run Tanuki against the **integration-branch state** of the
   target plugin — `tanuki-drive --plugin <loop-repo> --plugin-ref
   <integration-branch> …`, which checks out the integration commit in the
   clone rather than the plugin repo's currently-checked-out branch. This is
   what closes the feedback loop: iteration N+1 dogfoods the changes iteration
   N landed. (Without `--plugin-ref` a local clone follows the source's checked
   out branch — a general Tanuki correctness fix, not loop-specific.)
   **Charter rotation:** from iteration 2 on, the drive uses a derived variant
   of the base matrix — regression scenarios kept unchanged (their silence
   verifies the landed fixes), flow scenarios' `decision_points` varied to
   internal branches this run hasn't walked. Re-driving an identical matrix
   only re-confirms known findings (repetition is redundant unless
   it repeats the internal path — and here it does, so vary it).
2. **Mine.** Extraction + frontier dedupe → findings in the ledger, recurrence
   updated (`docs/tanuki-spec.md` §2, unchanged).
3. **Classify (internal — forked rules, copied from `/triage-gh`).** For each
   new actionable finding apply the altitude test copied here: **spec** =
   changes an invariant other components depend on; **story** = a plan across
   files; **direct** = small and self-evident. Bias is **toward
   implementation** — `direct`/`story` → the work list. **Defer only when a
   fix is physically impossible, self-contradictory, or blocked by missing
   credentials/configuration**, plus (iteration ≥ 2 or headless) a genuine
   `spec` decision for which the run policy has no answer. Deferred items go to
   the morning queue; the loop keeps going.
4. **Implement.** Implement each work-list item on the integration branch from
   the finding text (forked implement rules; no issue, no PR), incrementing
   that finding's **attempt count** (tracked per finding for the run).
5. **Test / verify.** `tanuki-loop test` runs the per-target `test_cmd`
   (configured at init) deterministically in the worktree; a failure is an
   immediate-stop breaker. Unconfigured, it reports skipped and the model
   verifies manually — unattended runs must configure one.
6. **Commit.** Intent-scoped commits (forked commit-groups rules) onto the
   integration branch, then `tanuki-loop iter-verify` — the **integration
   invariant**, all four or an immediate-stop breaker: (a) HEAD attached to the
   expected integration branch; (b) the iteration start SHA is an ancestor of
   the new HEAD; (c) HEAD differs from the start SHA when a patch was expected;
   (d) the worktree is clean. It records the end SHA.
7. **Ready for the next drive.** Loop back to step 1 until convergence or the
   cap or a breaker.

## Convergence and stop conditions

The cap is a ceiling; **convergence is the success condition**, and the loop
stops on **convergence or cap, whichever comes first.**

A drive → mine is **quiet** when all three hold: **no new actionable
findings**, **no accepted patch** was generated, and every remaining finding is
a **duplicate**, **deferred**, or **frozen**. Per the
spec-tanuki-scenario-lifecycle loop amendment, a cycle additionally counts as
quiet **only if its scheduler plan met the exploration quota** —
`record-cycle` reads the iteration's persisted plan record and gates on
`quota_met` itself (see "Dashboard revision", enforcement note).
**Convergence requires two consecutive quiet cycles** — a single quiet drive
can be luck given the deliberately weak driver. `tanuki-loop record-cycle`
tracks the streak and reports `converged` at streak ≥ 2 (a non-quiet cycle
resets it to 0).

Zero findings is *not* required. A finding may be **fixed repeatedly**: track
attempts per finding (default cap **4**). Do not treat the first recurrence as
failure or convergence — re-fix it. On the recurrence *after* the attempt cap,
**freeze that finding only** (stop re-fixing it, record it for the morning) and
continue the loop; a frozen finding no longer counts as a new actionable one.

## Circuit breakers

**Immediate stop** (freeze the integration branch, write the audit artifact,
report): test failure · implementation-command failure · commit failure ·
integration-invariant violation (the four-part check in step 6) · iteration
cap reached · wall-time limit · token budget exceeded · **external
modification of the integration worktree** (detected by comparing the recorded
end SHA / worktree cleanliness at the top of each iteration).

**Defer / freeze, do not stop** (record and keep going): a fix that is
physically impossible, self-contradictory, or blocked by missing
credentials/configuration → **defer** to the morning queue; a genuine `spec`
decision with no run-policy answer → **defer**; a finding past its attempt cap
→ **freeze** (that finding only). The loop keeps processing other independent
actionable findings and stops **only** if a deferred/frozen item blocks *all*
remaining work.

## Morning gate (attended — invariant across all phases)

The loop ends by presenting, for one review:
- the integration branch diff (the relocated Human Gate),
- the morning review queue (deferred spec / judgment items), and
- the audit artifact (every auto-decision the loop made in lieu of the human,
  plus per-iteration start/end SHAs and the convergence/breaker reason).

On the operator's approval the gate runs **merge-first and idempotent** — no
outward-facing record is written until the merge is a fact:
1. **Approve + final tests.** Re-run the target's full test/verify on the
   integration HEAD; abort the gate on failure.
2. **Merge `integration → main`.** Behind the approval, perform the merge.
3. **Verify reachability from `main`** (`git merge-base --is-ancestor
   <integration HEAD> main`). Only past this does anything outward-facing run.
4. **Materialize** the converged work as GitHub issues — **one issue per
   resolved problem** (collapsing the intermediate findings), describing **what
   landed**, each stamped with a stable `tanuki-loop-run: <run-id>` marker.
5. **Link** each issue to the merge commit, **close it as completed**, and
   reconcile the board to Done.

**Idempotent retry:** before creating an issue, search for its `<run-id>`
marker (plus a per-problem key); an existing one is reused, never duplicated.
If the gate dies after the merge, re-running resumes at step 4 and creates only
the missing issues. The GitHub history is a clean record of resolved problems,
not a transcript of the overnight search.

## Phases (supervision + cap, not separate code paths)

Because no phase files issues overnight and no phase auto-merges to `main`, the
phases share one implementation and differ only in supervision and cap:
- **Phase 1 — commissioning.** Small cap (1–2), operator watching, used to
  validate that the breakers fire and the reachability/rollback invariants
  hold.
- **Phase 2 / 3 — unattended.** Identical loop, higher cap, run headless
  (`--dangerously-skip-permissions` or a pre-approved allowlist), auto-advancing
  the mechanical gates and stopping at any breaker.

The architecture is **Phase-3-ready from the first commit**: raising the cap
and removing supervision is the only change between phases. The morning
`integration → main` gate is invariant in all of them.

## Rules

- Never write `main`; never touch the operator's normal working tree; never
  create GitHub issues/labels/PRs/story files during iterations.
- The ledger is the source of truth overnight; GitHub artifacts are written
  only at the morning gate, describing what landed.
- Every iteration is bracketed by start/end SHAs; rollback is `reset --hard`
  **plus** `git clean -fd` in the loop worktree only, recorded in the audit.
- Questions only in iteration 1 (recorded as run policy); iterations ≥ 2 are
  non-interactive — judgment without a policy answer is deferred, never asked.
- A finding may be re-fixed up to its attempt cap (default 4), then it is
  **frozen** — never a whole-loop stop.
- `integration → main` runs only after explicit approval, merge-first and
  idempotent; deferred judgment waits for the morning, never auto-decided.
- Stop on convergence or cap, whichever first; stop immediately on any
  immediate-stop breaker.
