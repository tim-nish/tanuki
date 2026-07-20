# Spec: /tanuki-loop — unattended cumulative dogfooding on an integration branch

Status: RATIFIED 2026-07-16 (header sync — proposed 2026-07-13, since
implemented and amended through 2026-07-15; the PROPOSED header had gone
stale). AMENDED 2026-07-18 (triage of issue #138): concurrent drive phase —
see "Concurrent drives within one iteration". Supersedes the blanket never-automate reading of the operator's
2026-07-11 decision record for mechanical (non-spec, non-outward) work
executed against snapshot-isolated state (integration worktree, host
fixture); the relocated morning gate is the surviving human gate
(owner decision record 2026-07-16, loop-automation supersession). Implemented in `commands/tanuki-loop.md`. Depends on the Tanuki pipeline
(`docs/tanuki-spec.md`) and the shared executables (`tanuki-loop`,
`tanuki-drive`, `tanuki-ledger`). Its classification and implementation
**judgment is an intentional fork** of the operator's attended triage and
implementation commands (rules copied here, allowed to diverge; owner
decision record 2026-07-13, loop fork) — the judgment layer is not reused.
Only the tools and the (issue-free) ledger are shared.

Origin: the attended cycle — tanuki, then the operator's attended triage /
publishing / implementation / commit-grouping commands, then merge, then
tanuki again — was
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
- **A declined gate leaves a debt, and the debt has an owner.** The gate's
  other outcome is that the branch is *not* merged — and that work does not
  evaporate: it sits on the integration branch while `main` moves, until the
  operator re-derives the same fixes by hand and the branch's answer and
  `main`'s answer contradict each other. On 2026-07-17 two branches held 25
  unmerged commits and had resolved two findings **opposite** to the way the
  attended sitting had, discovered only after both had shipped. So the loop
  owns a **reconcile** pass over its own unmerged branches — and reconcile is
  a **single terminal invocation** (#210, ratified 2026-07-19; this
  supersedes the prior attended/report-first contract — see the transition
  note in "Reconcile substrate" below). One invocation must:
  classify **all** unresolved runs, classifying **per semantic change unit**
  rather than per commit — a commit routinely mixes already-landed,
  superseded, conflicting and still-applicable hunks, and its subject line
  describes at most one of them; apply **every deterministic** port or
  disposition ("deterministic" = requires no owner semantic decision, NOT
  "algorithmically decidable" — agent-performed hunk isolation and adaptation
  may occur inside the invocation, and **missing behavioral evidence fails
  closed** to `needs-owner`, never guessed); **re-evaluate findings against
  current base behavior, per manifestation site** (a finding is resolved or
  superseded only when closed at **every** known site); iterate to a
  **fixpoint**; run **verification**; then present **one** gate containing
  **only genuine owner decisions** — contract conflicts, ambiguous
  supersession, spec-lane routing, `needs-owner` items — nothing a tool
  could have closed. After approval it performs the terminal actions with no
  further commands: **merge → push → update state → clean up** (branch and
  worktree cleanup that `/repo-cleanup` formerly owned moves here). On a
  PR-protected target (`gate: "pr"`) the terminal "merge" is expressed as PR
  mechanics, and the reconcile gate approval IS the explicit human decision —
  the loop still never auto-merges (the 2026-07-17 ruling's substance is
  preserved: no merge happens without the owner's explicit approval).
  Reconcile never resolves a contract conflict silently — it presents the
  decision the operator owes, with the governing text quoted.
  **Identifiers are evidence, never required inputs**: the normal flow
  defaults to all unresolved runs; commit hashes and branch names appear
  only as evidence in reports. Its **discovery and mechanical signals** come
  from `tanuki-loop unresolved` (below), never re-derived in the sitting;
  its **verdicts** remain semantic judgment, now executed inside the
  invocation and gated once.
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
- **Never dogfood stale code — base freshness is guarded at `init`.** The
  integration branch is cut from the local base tip, so a local base lagging
  its remote makes every iteration dogfood outdated plugin code and surface
  already-fixed findings at the morning gate (issue #2: a real run wasted
  triage effort confirming remote-fixed findings were non-issues). `init`
  therefore fetches the base's remote and **fails closed** when the local base
  is behind its upstream; the operator fast-forwards, names an explicit
  `--base`, or passes `--allow-stale-base` to dogfood the local tip
  deliberately. The base is **never silently retargeted** to the upstream tip —
  what runs is always the ref the operator named, and the base tip + any
  behind-count are recorded in `state.json` and the audit. A missing upstream
  or an offline fetch degrades to a recorded note, not a stop.

## Issue-free by construction

The overnight loop is a **fork**: the classification and implementation rules
are copied from the attended commands into this command (see "One iteration",
steps 3–4/6) and run over the **ledger**, never their GitHub-issue plumbing.
The tool layer and the ledger are genuinely shared; the judgment layer is
copied and may diverge — a change to the attended commands does not
propagate here.
- Classification (direct / story / spec) runs against ledger findings — no
  `triage:*` labels, no story files, no issues.
- Implementation runs from the finding text directly onto the integration
  branch — no bound issue, no PR.
- Intent-scoped commits land on the integration branch — no push, no PR.

GitHub artifacts exist only *after* the morning approval (see "Morning gate").
This honors the "ledger-only overnight" decision: the ledger is authoritative
during the run; the outward-facing record is written once, describing what
actually landed. One scoped exception: on a PR-protected target the run's
close delivers a **Draft PR** (see "PR-protected targets") — review material
awaiting the approval, not a record asserting its outcome; issues and the
merge itself still exist only after the operator ratifies.

## Deterministic substrate (`tools/tanuki-loop`) vs. judgment

Every scriptable safety check, invariant, and rollback lives in the
`tanuki-loop` executable so the model is left responsible **only** for
judgment (classify, implement) and running the target's tests. The tool owns:
worktree/branch/base-SHA setup (`init` — which also **guards base
freshness**, below), the per-iteration start gate —
cap / wall-time / external-modification breakers + start-SHA capture + a
ledger snapshot (`iter-start`), the four-part integration invariant + end-SHA
capture (`iter-verify`), rollback (`rollback` = `reset --hard` + `clean -fd` +
verify), the deterministic test runner (`test` — runs the per-target
`test_cmd` configured at init inside the worktree; failure is a breaker),
per-finding attempt counting and freezing (`attempt`), the two-quiet
convergence counter (`record-cycle`), the post-merge reachability check
(`gate-check`), and the materialization idempotency map (`issue-get`/
`issue-put`). Breakers are `tanuki-loop` exit code 3 with `{"breaker": …}`.
(`gate-check`/`gate-push` also exit 3 on their enumerated stop conditions
but emit their own payloads — `{"reachable": false}` / `{"pushed": false,
…}` — not a `breaker` key; scripted callers must not key on `.breaker` for
the gate commands.)

The tool also owns the operator surface: `doctor` (the read-only Phase 2/3
headless-readiness validator — see "Headless readiness" below), `policy`
(records iteration-1
judgment answers — the mechanism behind "questions only in iteration 1"),
`recover` (the attended external-modification recovery path — see "Circuit
breakers"), `finish` (machine-readable terminal reason: converged | cap | aborted —
gate-ratified survives as compatibility only per the 2026-07-17
delivery-boundary ruling; breakers persist `last_breaker` the same way), and `dashboard`
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

## Headless readiness (`tanuki-loop doctor`) — ADDED 2026-07-15

Phase 2/3 readiness is validated **up front, read-only, without running the
pipeline**: `tanuki-loop --target <t> doctor --loop-repo <repo> [--scenarios
F] [--policy F] [--phase 2|3] [--allow-stale-base] [init's override flags]`.
Rationale: this spec requires ceilings for an unattended run ("always ensure
they're set") and a policy supplied up front, but `init` does not enforce
either — a forgotten `"loop"` block previously surfaced only mid-run (an
unbounded loop, a baseline test failure overnight, or a night of deferrals
discovered at the morning gate). The validator is a preflight gate, **not
another execution mode**: it runs no drive, no mining, no iteration.

It reuses `init`'s own resolution and guards (`resolve_loop_config`,
`resolve_base`, `check_base_freshness`) — never parallel logic — so it cannot
disagree with the run it validates. Five checks:

1. **Required headless config** — `test_cmd`, `wall_time_s`, `token_budget`,
   `attempt_cap`, and `iterations` all resolved (scenarios `"loop"` block or
   CLI override); any absent ceiling is named. Fatal.
2. **Base freshness** — the same measurement `init` fails closed on
   (issue #2); `--allow-stale-base` mirrors init's explicit override. Fatal
   without the override.
3. **Baseline `test_cmd`** — run once against the base tip in a **throwaway
   detached worktree, removed afterward** (the operator's tree is never
   touched; the repo nets zero change). A base-tip failure means every
   iteration would trip the test breaker on **inherited state, not a
   regression** — better learned in seconds than overnight. Fatal.
4. **Policy coverage** — which actionable ledger findings (status
   open/proposed, not tombstoned) have no answer in the supplied policy
   (`--policy`, else the current run's `policy.json`, else empty) and would
   therefore **defer** under "questions only in iteration 1".
   **Informational, never fatal** — deferral is the designed-safe headless
   outcome; the check exists so a night of it is a known cost, not a morning
   surprise. A malformed policy file, by contrast, is a failed check.
5. **Host cloneable** (hosted targets; mandated by
   `specs/spec-host-snapshot/SPEC.md`) — the scenarios file's live host path
   exists, is a git repo, and a throwaway clone of it succeeds (removed
   afterward), so `init`'s fixture pinning cannot fail overnight. Hostless
   targets report the check as passing with a "no host fixture to pin" note.
   Fatal for a hosted target.

**Read-only contract:** `doctor` writes no repo, ledger, loop-state, or policy
data (check 2's fetch refreshes remote-tracking refs only, exactly like
`init`'s guard). A failed check follows the breaker convention
(`{"breaker": …}` + exit 3) but persists nothing — there is no run state to
write. Out of scope, deliberately: a stub-drive rehearsal mode and runtime
token-budget enforcement (the budget remains a stored, harness-tripped
ceiling).

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
   *this run* changed — newly observed / status→accepted / recurred
   findings, deferred/frozen with reasons inline — computed from the
   run-start ledger snapshot (`iter-start` now snapshots
   `{id: {status, recurrence}}`, not bare ids). Lifetime totals are one
   pointer line to the history view (`tanuki-scheduler history`), which owns
   the long view. (AMENDED 2026-07-18: the delta previously labeled `fixed`
   is a ledger status transition, open/proposed → accepted — it is now
   labeled as exactly that, because `fixed 0` beside the convergence line's
   patch fact read as a contradiction.)
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

### Fixed skeleton (REVISED 2026-07-15 — section structure is harness-owned)

The six points above govern *content*; this revision pins *structure*, closing
the gap dogfooding surfaced (finding F5: sections silently absent, leaving a
cold reader unable to distinguish "no data yet" from "missing data").

1. **Six sections, fixed order, unconditional presence.** Every render emits,
   in this order: (1) header + health verdict, (2) latest drive, (3) this run,
   (4) scheduler decisions, (5) convergence, (6) why stopped / NEXT. Section
   presence is enforced by the rendering code, never by the availability of a
   substrate file — the skeleton is the contract a cold reader learns once.
2. **A missing substrate degrades to a stated reason, never to absence.**
   When a section's input does not exist, the section renders its heading plus
   one explicit line naming what is missing and why that is expected, e.g.
   `latest drive: none yet — iteration 1 has not started a drive
   (no progress.json)`, `scheduler decisions: no scheduler state for this
   target — run tanuki-scheduler sync first`, `this run: no ledger movement
   yet`. This generalizes point 4's "degrade … with an explicit note — never
   silently" from the plan record to **every** section.
   **Run provenance flips expected-vs-gap (issue #236 / F166).** An empty
   `latest drive` / `scheduler decisions` section is a **gap** only for a
   *scheduled* run that should have driven and did not; for a run that drove
   *outside* the scheduler — a manual, toy, or ad-hoc run with no persisted
   plan — the same emptiness is **expected**, and the section's reason names
   that provenance rather than reading as a defect. Provenance is derived
   from the presence/absence of a persisted plan record for the run, never
   guessed. This mirrors `specs/spec-tanuki-view/SPEC.md` D3's typed
   `expected: true|false` enumeration; the two surfaces share one rule.
3. **Counters carry their definitions inline, and their labels are the
   substrate's facts.** Any number whose meaning is not derivable from the
   line it appears on states its definition next to its first use — e.g.
   `status→accepted` = ledger status transitions this run (not patch
   commits); the iteration counter counts *consumed* iterations, including
   rolled-back ones (so `cap hit` next to `1/3` cannot read as a
   contradiction). A label may not imply a fact the substrate does not
   record (AMENDED 2026-07-18: `fixed` retired for this reason; the
   convergence line says patch commits were *produced* on the integration
   branch, never "landed"). A counter needing a paragraph belongs in the
   audit, not the dashboard.
4. **No model-supplied text except quoted reasons.** The only free text on
   the dashboard is operator/model-authored reason strings (deferral/freeze
   reasons). These render truncated to one line with an explicit ellipsis
   marker **and** a pointer to the full text (`… — full: queue.md`); a
   truncation that hides the justification a reviewer needs is a contract
   violation, not a cosmetic choice.

The skeleton is verifiable mechanically: a dashboard render against an
`init`-only run (no drive, no scheduler state, no findings) must still emit
all six section headings with their degradation lines.

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
3. **Classify (internal — forked rules, copied from the operator's attended
   triage command).** For each
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
6. **Commit.** Intent-scoped commits (forked commit-grouping rules) onto the
   integration branch, then `tanuki-loop iter-verify` — the **integration
   invariant**, all five or an immediate-stop breaker (AMENDED 2026-07-19 —
   triage of issue #159): (a) HEAD attached to the
   expected integration branch; (b) the iteration start SHA is an ancestor of
   the new HEAD; (c) HEAD differs from the start SHA when a patch was expected;
   (d) the worktree is clean; (e) the iteration's commits introduce no
   build-artifact paths (pattern source shared with doctor's
   `test-cmd-artifacts` check — one list, two surfaces; a run needing to
   commit a legitimate binary declares it via an explicit per-run override,
   never by the guard silently standing down). It records the end SHA.
   Check (e) closes the write path a hand-run `git add -A` opens: doctor's
   artifact check runs only at doctor time, which Phase 1 may skip, so
   without (e) nothing between the commit and the remote objects.
7. **Ready for the next drive.** Loop back to step 1 until convergence or the
   cap or a breaker.

## Concurrent drives within one iteration (AMENDED 2026-07-18 — triage of issue #138)

The cumulative rule ("one integration branch, never fan out") governs
**implementation** — code must stack. The **drive phase** carries no such
dependency: scenarios already run against snapshot-isolated fixture clones,
so within one iteration the planned scenario set may drive concurrently.

- **`drive_concurrency` loop-block key (default 1).** 1 is exactly today's
  serial behavior and remains the default; N>1 lets `tanuki-drive` run up to
  N planned scenarios in parallel, one disposable host-fixture clone per
  scenario (the fixture is already a clone — this is N clones instead of 1).
  Resource use is explicit, never inferred.
- **The iteration bracket is unchanged.** `iter-start → drives → mine →
  classify/implement → iter-verify` — only the drives overlap. Mining is a
  **barrier**: one mining pass over the combined transcripts after all
  drives finish. Ledger writes happen only at mining, so concurrency
  introduces no concurrent ledger writes.
- **Progress contract generalizes.** With N>1 there is no single "current
  scenario": the progress substrate records **per-scenario progress
  entries** (stage, elapsed, KB, result), and the dashboard's latest-drive
  section renders the aggregation — running scenarios listed with their
  stages, done/total across the set. At `drive_concurrency: 1` the rendered
  output is today's (one current scenario) — the serial view is the N=1
  case of the aggregate, not a second format.
- **Budget accounting sums at the barrier.** Token/cost accounting for the
  iteration is the **sum across concurrent drives**, folded in at the
  mining barrier — before the next `iter-start` breaker evaluation, so the
  token-budget breaker sees the true total. The budget remains a stored,
  harness-tripped ceiling (unchanged posture).
- **Anomaly classification is unchanged.** Results stay per-scenario;
  drive concurrency is invisible to the ledger, the scheduler, and the
  morning gate.

Rejected at the same triage: keeping drives serial as a simplicity invariant
(wall-clock addressed only by stage-entry fixtures and cheap verify covers) —
declined because concurrency is the one lever that raises **coverage per
wall-clock hour**, and the isolation model already paid for it.

## Convergence and stop conditions

The cap is a ceiling; **convergence is the success condition**, and the loop
stops on **convergence or cap, whichever comes first.**

**Canonical quiet-cycle definition** (every other statement of it — dashboard,
command doc — references this one): a cycle is **quiet** when all three hold:
**no new actionable finding** (computed from the `iter-start` ledger snapshot —
a finding is *not* new-actionable when it is a duplicate bump, deferred, or
frozen), **no patch landed** (start SHA == end SHA), and — per the
spec-tanuki-scenario-lifecycle loop amendment — **the iteration's persisted
scheduler plan reported `quota_met: true`**; `record-cycle` computes and gates
all three itself (see "Dashboard revision", enforcement note).
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

**Recovery from the external-modification breaker (attended — ADDED
2026-07-15, resolves ledger finding F4).** The external-modification breaker
fires when every iteration is already closed (`iter-start` is the detector),
so `rollback` — which requires an *open* iteration — is structurally the wrong
tool and must never be the answer (misusing it also overwrites `last_breaker`
with a less-informative sequencing breaker; ledger F17). Recovery is a
dedicated attended subcommand, `tanuki-loop recover`, with these guarantees:

- **Preconditions (guarded, breaker on violation):** `last_breaker.reason` is
  an external-modification breaker, and no iteration is open. `recover` is an
  operator command — the loop never invokes it, and nothing recovers
  automatically.
- **Two explicit modes — the human chooses; there is no default:**
  - `recover --restore` — restore the loop-owned state: `git reset --hard
    <head_expected>` + `git clean -fd` in the loop worktree only. The
    externally-introduced commits/files are discarded from the worktree
    (recoverable via reflog), and the audit records the discarded range and
    removed paths.
  - `recover --adopt` — re-baseline around the external change: require a
    clean tree, then set `head_expected` to the current worktree HEAD. The
    audit records old → new and the adopted commit range (flagged loudly when
    the old `head_expected` is not an ancestor — history was rewritten, not
    extended). Adopted commits become part of the integration branch the
    morning-gate diff presents.
- **Closed iterations are immutable:** neither mode edits any recorded
  iteration (`start`/`end`/`phase`); recovery happens *between* iterations,
  adjusting only `head_expected`, the worktree, and the audit. The next
  `iter-start` then passes its own guards normally.

## Reconcile substrate (`tanuki-loop unresolved`) + drift notification — ADDED 2026-07-17

> **Transition note — terminal contract (#210, ratified 2026-07-19).** The
> reconcile pass was originally "attended judgment by contract":
> report-first, never merging without a gate, never deleting a branch
> (`/repo-cleanup` owned deletion). Dogfooding (writing-assistant run
> `loop-20260719-173009`) showed that contract cost ~5 human turns per
> sitting and shipped an incorrect per-finding verdict (F77 marked
> superseded while live at a second site). The pass is now a **single
> terminal invocation** (Non-negotiables, "A declined gate leaves a debt"):
> deterministic dispositions are applied inside the invocation, findings are
> re-evaluated **per manifestation site**, and post-approval
> merge/push/state-update and branch/worktree cleanup are terminal steps of
> reconcile itself — `/repo-cleanup` no longer owns reconcile-branch
> deletion. **Ratified definitions:** "deterministic" = requires no owner
> semantic decision (missing behavioral evidence fails closed to
> `needs-owner`); identifiers are evidence, never required inputs (default
> is all unresolved runs); one gate, genuine decisions only; per-site
> finding closure with stored, re-derivable closure evidence.

Judgment stays in the pass; two things around it are **not** judgment, and
leaving them to the sitting is why reconciling felt improvised and needed a
model for work a script should do:

- **Discovery.** Nobody knew the debt existed. Two branches accrued 25
  unmerged commits and were found only because someone went looking, by which
  time each had resolved a finding **opposite** to the way the attended
  sitting had (F102, F25→F99). Drift is silent by construction: a declined
  gate produces no artifact, and `git branch` is not a place anyone looks.
- **Mechanical signals.** Ahead/behind, files touched, applies-cleanly, the
  finding ids a message cites and whether they are still open — all
  deterministic, all re-derived by hand today.

Per the routing principle (deterministic → scripts, judgment → the command
layer), those become a tool; the verdicts stay in the pass.

### `tanuki-loop [--target <t>] unresolved [--json]` — read-only

Enumerates the loop's own unmerged integration branches. It runs no drive, no
iteration, writes no state, and **prompts never** — the
`announce_new_target` rule applies (*"never a prompt; unattended callers must
keep working"*). Per branch:

- identity: branch, target, run id, tip SHA, created/last-commit dates;
- **staleness**: commits ahead of the default branch, commits behind it, and
  days since its last commit — behind-count is the decay signal (the two 2026
  branches reached 63 and 116 behind);
- **merge state**: whether any PR from it merged, and whether its tip is
  reachable from the default branch;
- per commit: subject, files touched, whether it cherry-picks cleanly onto
  the default branch, and the ledger finding ids its message cites **paired
  with each finding's current status** — a commit citing a finding that is
  now `accepted` and tombstoned is a superseded *candidate*;
- worktree: path and whether it is clean.

**What it must not do — the load-bearing constraint.** It emits **signals,
never verdicts**. It must not print, return, or imply already-landed /
superseded / conflicting / still-applicable. Those are semantic: on
2026-07-17 every one of 25 commits was patch-non-equivalent (`git cherry`
called them all unmerged) while several were already landed, and a commit
subjected "render the module docstrings in `--help`" carried a **rejected
design** in its other hunk. A tool that guessed a verdict from patch identity
or a subject line would be confidently wrong, and confidently wrong is worse
than silent — it is the failure the pass exists to prevent. `cherry-picks
cleanly` is a fact about git; it is **not** a synonym for applicable, and the
field name must not suggest it is.

Ownership: `unresolved` reuses the same branch/worktree resolution `init` and
`branch_cleanup` already use — never parallel logic — so it cannot disagree
with the run it describes.

### The notification — pull-based, never a prompt

Drift is reported through the surfaces the operator already opens, and
through one stable path they can open cold:

1. **A `note:` line on stderr** from `/tanuki-loop` and `/tanuki-loop
   <t> reconcile` when unresolved branches exist: how many, how stale the
   worst is, and the one command that acts on it. One line, never blocking,
   never a prompt — an unattended phase that hits it keeps running.
2. **`~/.tanuki/<target>/unresolved.md`** — a stable-path brief, rewritten by
   `unresolved`, that reads cold: the branches, their staleness, and what the
   reconcile sitting will ask of the operator.
   **It lives under `~/.tanuki/`, never in the target repo** — the
   non-negotiable is absolute ("never write any configuration into the target
   repository"; the brief's canonical home is already `~/.tanuki/` for this
   reason). A sibling project's equivalent brief lives in its own repo; that
   shape is not available here and the difference is deliberate, not an
   oversight.
3. **Empty state is enumerated**, per `spec-tanuki-view` D3: "no unresolved
   branches" is an `expected` state with its reason, not silence — silence is
   what let 25 commits rot.

The notification **states, never acts**: it never merges, never deletes, and
never opens the gate. It exists so the operator learns the debt exists on a
day they were not looking for it.

### Acceptance

- A repo with unresolved branches: `/tanuki-loop` prints the note; the note
  names the count, the worst staleness, and `reconcile`. Exit code and every
  other output are unchanged.
- A repo with none: no note; `unresolved` reports the enumerated empty state;
  `unresolved.md` says so rather than being absent.
- An unattended (Phase 2/3) run with unresolved branches present completes
  normally — the note is stderr, and nothing waits.
- `unresolved --json` emits signals only; a test asserts the payload contains
  **no verdict field** (no already-landed/superseded/conflicting/applicable),
  because a future contributor's instinct will be to add one.
- `unresolved` writes nothing under the loop repo (a fixture asserts the
  target repo nets zero change), and nothing outside `~/.tanuki/<target>/`.
- One fixture per enumerated empty state (D3's rule).

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
   <integration HEAD> main`).
4. **Push `main` to its remote** (`tanuki-loop gate-push`) — the first
   outward-facing step, before any issue is materialized. A merge that stays
   local is not done: the commit links step 5 stamps onto issues reference a
   commit that does not exist on the remote (they 404), and a local-only `main`
   diverges the remote for any concurrent workflow's `git pull --ff-only`
   (observed 2026-07-14, run `loop-20260714-154413`: a local-only gate merge
   forced a manual reconcile after story PRs landed on the remote). A diverged
   remote is reconciled **explicitly** — `gate-push` fetches and, if the remote
   moved, refuses rather than force-pushing; the operator merges the remote in,
   re-runs the final tests, and re-runs `gate-push`. Only past a successful
   push does anything else outward-facing run.
5. **Materialize** the converged work as GitHub issues — **one issue per
   resolved problem** (collapsing the intermediate findings), describing **what
   landed**, each stamped `tanuki-loop: <run-id>/<problem-key>` (the
   Materialization key above — one format, everywhere).
6. **Link** each issue to the (now pushed) merge commit, **close it as
   completed**, and reconcile the board to Done.
7. **Remove the loop worktree.** No closing command follows (owner ruling
   2026-07-17): the merge's reachability from the base *is* the settlement,
   derived by whatever next needs it — the next `init`'s merged-branch sweep
   deletes the integration branch, and `/repo-cleanup` owns any earlier
   cleanup. `finish --reason gate-ratified` survives as a compatibility
   mechanism only; no workflow instructs it.

**Idempotent retry:** before creating an issue, search for its `<run-id>`
marker (plus a per-problem key); an existing one is reused, never duplicated.
If the gate dies after the push, re-running resumes at step 5 and creates only
the missing issues. The GitHub history is a clean record of resolved problems,
not a transcript of the overnight search.

## PR-protected targets (`gate: "pr"`)

Some base branches refuse direct pushes (a required-check branch protection);
there the local-merge + `gate-push` sequence above cannot run at all. For such
a target the scenarios `loop` block sets `"gate": "pr"` (default `"merge"` —
everything above unchanged), and the gate reshapes around the forge:

**Delivery is the loop's boundary (owner ruling 2026-07-17).** After a
successful unattended run closes (`finish --reason cap|converged`) and the
final tests pass on the integration HEAD, `tanuki-loop gate-pr` pushes the
**integration branch** (never the base, never forced) and opens **one Draft
PR** `integration → base`. **The loop ends when that Draft PR opens.** The
run is recorded as **delivered** — PR number, integration-tip SHA, base SHA —
and that is the loop's terminal fact. The loop does not poll, merge, comment,
or wait for the PR outcome; whether the PR is ultimately accepted is not the
loop's to know, and **no human runs a machine-level closing command** to tell
it. The Human Gate is **PR approval + merge on the forge**; the loop **never
auto-merges** — `gate-pr` contains no merge call, and no phase adds one.

Because the Draft PR is review material rather than a decision, opening it
does not violate "no outward-facing artifacts overnight": that rule's target
was always *records that assert conclusions* (issues stating what landed,
merges into `main`). A draft marked as awaiting review asserts nothing.

**Settlement is derived, never declared.** Later **read-only** operations
(`status`, `init`'s cleanup backstop, `unresolved`) derive the outcome from
the forge and the current base, at the moment they actually need it:
- **merged and reachable from the current base** → `landed` (the
  merged-on-forge-but-base-not-pulled window is `landed` with
  `local_synced: false` and a sync hint);
- **PR still open** → `pending` — awaiting review, not rot;
- **closed without merging** → `declined` — the branch holds unlanded work,
  and what to do with it is the reconcile pass's judgment;
- **forge unavailable or unverifiable** → `unknown`, **never an optimistic
  default**. Offline views may serve the **last observed** result with its
  timestamp, always marked stale — a cache is never presented as current
  truth, and no view requires a human to restate the forge's decision.

**The direct-merge (non-PR) gate settles the same way — by reachability, not
by a forge PR.** On a non-PR-protected target the gate merges `integration →
base` locally rather than opening a Draft PR, so there is no PR state to read;
the settlement is nonetheless derived, from the **same strict test** — the
integration tip's reachability from the current base (the "the merge's
reachability from the base *is* the settlement" rule stated above):
- **integration tip reachable from the current base** → `landed`, with
  `delivered` populated from the local merge (integration-tip SHA, base SHA,
  merge commit) exactly as `gate-pr` populates it from the PR;
- **not yet reachable** (merge not run, or base moved past it) → `pending`;
- **base unreadable** → `unknown`, never an optimistic default.
So `status.delivered`/`settlement` populate symmetrically on both gate paths —
the PR path sources reachability alongside the forge PR, the direct-merge path
sources it from the base alone. A direct-merge run whose merge has landed must
never show `settlement: null`, which a reader would misread as a lost delivery.

**Accounting uses the strict test.** Finding verification and recurrence
accounting may treat a delivered change as landed **only** when its delivered
SHA is reachable from the current base — never from the forge's word alone,
and never optimistically.

**Cleanup ownership.** Branch and worktree removal belong to `/repo-cleanup`
(with the next `init`'s merged-branch sweep as the mechanical backstop), and
may delete **only** after reachability proves the work landed, or after an
attended decision explicitly discards it. An unmerged tip is never deleted —
unchanged. `finish --reason gate-ratified` survives **as a compatibility
mechanism only**: no workflow instructs it; invoked anyway, it derives the
same settlement and refuses unless the delivery verifiably landed.

Mechanics, each mechanically enforced by the tool (exit 3 + remedy, nothing
mutated, on violation):
- **Preconditions.** `gate-pr` refuses unless the run is finished
  (`cap`/`converged`) and `tanuki-loop test` recorded a pass against the
  *current* integration HEAD (the recorded SHA must match — call order is not
  trusted). `doctor` validates `gh` availability up front for `pr`-mode
  targets (`gate-pr-tooling` check).
- **Idempotent.** An existing PR for the head branch — recorded in
  `state.json` or found on the forge (covering a crash between create and
  record) — is reused and reported, never duplicated.
- **Failure leaves everything intact.** A failed push (never forced) or a
  failed PR create exits 3 with the local integration branch, worktree, and
  run state untouched; re-running `gate-pr` resumes.
- **Merge method.** The intended method is a merge commit, so the loop's
  intent-scoped commit groups stay visible in the base's history; the PR body
  states this. (Settlement derivation tolerates squash/rebase via the forge's
  merge-commit oid, but the grouped history is then lost — an operator choice,
  not a loop one.)

Issue materialization (steps 5–6) is unchanged where a tracker is configured;
the PR replaces only the merge/push steps (2–4).

**Branch cleanup has an owner** (amended by the 2026-07-17 delivery-boundary
ruling): `/repo-cleanup` owns branch and worktree removal, with the next
`init`'s merged-branch sweep as the mechanical backstop — both delete an
integration branch **only** when its tip is reachable from the base
(reachability proves the work landed) or an attended decision explicitly
discards it; an **unmerged** tip is never deleted. `finish`'s own sweep (on
`cap`/`converged`/`aborted` closes) remains as an additional backstop with
the same reachability rule. This closes a leak where merged
`tanuki-loop/<target>/<ts>` branches accumulated until an unrelated tool
happened to collect them.

### Stop reason vs delivery settlement (ADDED 2026-07-18)

The delivery-boundary ruling above already fixes what settlement *is* —
derived, never declared, with its closed result set (`landed | pending |
declined | unknown`). This section adds the fact that lives on the other
side of the boundary, because `finish --reason gate-ratified` used to
overwrite the run's close and the dashboard then presented `gate-ratified`
as why the run stopped — a category error the operator had to undo by
reading audit.md:

- **Stop reason** — why loop *execution* ended. A closed set: `cap` |
  `converged` | `breaker` | `cancelled` (the lifecycle name for a finish
  reason of `aborted`). Recorded (`state.json` `stop`) at the **first
  computational close** and never overwritten; a settlement — including the
  compatibility `gate-ratified` — is not a stop reason and may never render
  as one. A legacy run whose close was overwritten before this separation
  reports `unknown` — never a guess.
- **Settlement in views.** A read-only view never polls the forge
  (spec-tanuki-view): it derives **offline** — local reachability still
  answers `landed`; otherwise the last cached observation, always marked
  stale with its timestamp, or `unknown`. The same never-optimistic rule:
  offline derivation can only *confirm* landed from reachability, never
  assume it.
- **Execution time** ends at the first close (`state.json` `wall_end`);
  waiting for PR review or merge never accrues to it. Runs closed before
  `wall_end` existed freeze at their recorded close timestamp.

`tanuki-loop status` emits this as one typed `lifecycle` block (`active`,
`breaker_open`, `stop_reason`, `stopped_at`, `execution_s`, `delivered`,
and the offline-derived `settlement`; the live-derived settlement remains
the top-level `settlement` field). The dashboard renders stop reason and
settlement as separate lines and freezes its elapsed figure. Views derive
from this block and never re-classify (spec-tanuki-view D4).

## Phases (supervision + cap, not separate code paths)

Because no phase files issues overnight and no phase auto-merges to `main`, the
phases share one implementation and differ only in supervision and cap:
- **Phase 1 — commissioning.** Small cap (1–2), operator watching, used to
  validate that the breakers fire and the reachability/rollback invariants
  hold.
- **Phase 2 / 3 — unattended.** Identical loop, higher cap, run headless
  (`--dangerously-skip-permissions` or a pre-approved allowlist), auto-advancing
  the mechanical gates and stopping at any breaker. Readiness is validated
  up front by `tanuki-loop doctor` (see "Headless readiness") before `init`.

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
  idempotent; the base is **pushed to its remote before any issue is
  materialized** (`gate-push`, never force over a diverged remote); deferred
  judgment waits for the morning, never auto-decided. On a PR-protected
  target (`gate: "pr"`) the approval takes the form of **PR approval + merge
  on the forge**: the loop ends at delivering one Draft PR (`gate-pr` —
  review material, not ratification; owner ruling 2026-07-17), it **never
  auto-merges**, and settlement (landed / pending / declined / unknown) is
  **derived** by read-only surfaces from the forge and the current base — no
  human closing command restates the forge's decision. Reconcile's terminal
  merge (#210) does not breach this: on a `gate: "pr"` target the terminal
  action is expressed as PR mechanics, and the reconcile gate approval is
  the owner's explicit decision — no merge ever happens without it.
- Merged integration branches are cleaned up by `finish` (backstop: the next
  `init`); outside reconcile, an unmerged tip is never deleted. An unmerged
  tip is therefore a **debt with an owner**: `unresolved` makes it visible
  and `reconcile` pays it down — and once the reconcile gate is approved,
  reconcile's own terminal step deletes the now-settled branch and worktree
  (#210; this cleanup formerly belonged to `/repo-cleanup`). Never-deleted
  must not mean never-looked-at — that is precisely how 25 commits rotted
  into contradiction.
- `unresolved` emits **signals, never verdicts**, and prompts never. Whether a
  change is already-landed, superseded, conflicting or still-applicable is
  semantic judgment that belongs to the reconcile pass — executed inside its
  single terminal invocation, gated once (#210); patch identity and
  subject lines are both known-wrong proxies for it.
- Stop on convergence or cap, whichever first; stop immediately on any
  immediate-stop breaker.
