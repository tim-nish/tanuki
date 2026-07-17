# Spec: doc/policy alignment fixes — publication boundary and internal consistency

Status: PROPOSED 2026-07-16 (alignment review of every tracked spec/doc against
the operator's recorded policy; produced on the operator's order). Touches
`README.md`, `docs/tanuki-spec.md`, `specs/spec-host-snapshot/SPEC.md`,
`specs/spec-tanuki-loop/SPEC.md`, `commands/tanuki-loop.md`. No behavior
changes — every item is a documentation or provenance fix; tools are untouched.

Scope of the review: `README.md`, `PRIVACY.md`, `CHANGELOG.md`,
`docs/tanuki-spec.md`, all five `specs/*/SPEC.md`, both `commands/*.md`,
`templates/*`, and the tracked `docs/stories/*`. The review checked (a) the
publication boundary — this is a public repository; provenance must use dated
generic decision-record references, never operator-private names or paths
(owner decision record 2026-07-16, public-spec provenance) — and (b) internal
consistency between the core pipeline contract and its extension specs.

## Findings (ranked)

### F1 — Private repository name in a public spec (boundary, must-fix)

`specs/spec-host-snapshot/SPEC.md`, Origin paragraph (line 10): the incident
description names the private repository the loop was running against. The
spec's own ratification note says the origin was genericized; the
genericization is incomplete — the name survived.

**Fix:** replace the repository name with a generic phrase — e.g.
"during loop-20260715-121003 (a private repo of the operator's, Phase 2)".
The run id is Tanuki's own and stays; only the target's identity is private.

### F2 — Core contract contradicted by the loop, unscoped (consistency, must-fix)

Three statements of the founding write-rule are mutually inconsistent:

- `docs/tanuki-spec.md` (Standing constraints): "Proposals-only: nothing
  merges, **nothing writes to the target repo**, findings never auto-update
  specs."
- `README.md` (intro): "Tanuki … **never writes into the repo under test**."
- `specs/spec-tanuki-loop/SPEC.md` + `README.md` (loop section): the loop
  implements fixes and **commits onto an integration branch of the target
  repo** (in a dedicated worktree), and the morning gate merges and pushes.

The loop's design is deliberate and gated (never `main`, never the operator's
working tree, merge only behind explicit approval), but the two blanket
statements are written as unscoped non-negotiables, so a reader hits a direct
contradiction ~100 lines later. Unscoped absolutes that the same repo's specs
override are how contracts drift.

**Fix:** scope the absolutes at their source:

- `docs/tanuki-spec.md`: "Proposals-only: nothing merges, nothing writes to
  the target repo **during the attended pipeline**; `/tanuki-loop` writes
  only to its own integration branch inside a dedicated worktree, never
  `main`, never the operator's working tree — merge happens only at the
  attended morning gate (spec-tanuki-loop)."
- `README.md` intro: "never writes into the repo under test — the overnight
  loop's only write surface is its own integration branch, and only you merge
  it (see Unattended overnight mode)."

### F3 — README contradicts the spec on issue labels (consistency, must-fix)

`README.md` (decision pass): "Filed issues carry a `tanuki` label **plus**
`tanuki:<kind>`." `docs/tanuki-spec.md` (Issue labels): "A filed issue carries
**exactly one** label … **no bare `<prefix>` marker label is applied**." The
label contract is the *only* machine-readable interface Tanuki emits;
downstream tooling keying on the README's two-label description would search
for a label that is never applied.

**Fix:** correct the README to the spec's contract: one label,
`tanuki:<kind>`; the prefix inside the kind label is the provenance marker.

### F4 — Operator-private command names as provenance (boundary, should-fix)

`specs/spec-tanuki-loop/SPEC.md` (Origin, steps 3 and 6) and
`commands/tanuki-loop.md` (lines 7–8, 142, 162) name the operator's private
attended commands whose rules were forked into the loop. The fork is real and
correctly declared ("copied, allowed to diverge — the judgment layer is not
reused"), so the names carry no contract weight: they are provenance, and
provenance in this repo is dated generic decision-record references, never
private tooling names a public reader cannot resolve.

**Fix:** replace the names with role descriptions — "the operator's attended
triage / implementation / commit-grouping commands (rules copied here,
allowed to diverge; owner decision record 2026-07-13, loop fork)" — in all
five locations. The fork semantics sentence stays; only the identifiers go.

### F5 — The loop's automation level is ahead of the operator's recorded policy (provenance gap, operator decision)

An operator decision record of 2026-07-11 (findings pipeline) states that
findings flow finding → proposal → human gate and that "the full closed loop
is never automated." `spec-tanuki-loop` (ratified repo-side 2026-07-13/16)
automates classify → implement → test → commit overnight for non-spec work,
relocating — not removing — the human gate (morning review; outward-facing
actions and design decisions stay manual). The spec is arguably consistent
with the *intent* of the earlier ruling (the gate that matters is preserved),
but the earlier blanket record was never explicitly superseded, so the two
readings coexist.

**Fix (in this repo):** add one line to `spec-tanuki-loop`'s status header:
"Supersedes the blanket never-automate reading of the operator's 2026-07-11
decision record for mechanical (non-spec, non-outward) work; the relocated
morning gate is the surviving human gate." **Fix (operator-side, out of this
repo's scope):** record the supersession in the operator's own decision log
so the two surfaces agree; until then this spec item stays open.

## Verified aligned (no action — recorded so the review is auditable)

- **Capture never judges / one-way flow:** `--ingest` verbatim, extraction
  downstream, no ingest-side policy filtering (spec-policy-advisory rejects
  it by design, not deferral).
- **Policy surface:** opt-in read-only pointer, code-enforced ≤4-file
  allowlist, pinned reads (`file:line@commit`), consulted-line audit,
  advisory-never-blocking at the gate only, enhancer-never-dependency,
  nothing persisted. Matches the operator's seam contract exactly.
- **Trajectory:** view-not-noun, mechanical marker lift, MCTS/value-backup
  explicitly rejected, plan-gated charter seeds, its own publication-boundary
  note (the convention F1/F4 items should follow).
- **Scheduler:** coverage-preserving adaptivity — never-delete, regression
  cadence, exploration quota, hysteresis; generation frontier-tier and
  plan-gated.
- **Host snapshot principle:** fixture-not-live-host, no auto-restore, the
  operator's repo is never the tool's jurisdiction.
- **Caps as architecture:** `max_scenarios`, `model_ceiling` (fail-closed on
  unknown models), turn/wall/token ceilings, `brief_max_proposals`.
- **Brief shape:** capped, ranked, Problem → Proposal → Evidence,
  claim–evidence binding, decision pass ends the run in dispositions.
- **Boundary sweep:** no private repository names, paths, or knowledge-base
  pointers anywhere else in the tracked files; `PRIVACY.md`, `CHANGELOG.md`,
  templates, and tracked stories are clean.

## Non-goals

- Any tool or behavior change (every finding above is documentation).
- Retrofitting untracked local files (BMAD scaffolding, local skills) — they
  are outside the publication boundary already.
- Relitigating ratified design (the loop's relocated gate, the fixture
  principle) — F5 asks for the *record* to be reconciled, not the design.

## Acceptance

- No tracked file names an operator-private repository, command, or knowledge
  path: `git grep` for the F1/F4 identifiers over tracked files returns only
  this spec's own file-location references (or nothing, once fixed there too).
- README, docs/tanuki-spec.md, and spec-tanuki-loop state the same write-rule
  and the same one-label contract.
- spec-tanuki-loop's header carries the supersession line; the operator-side
  record is reconciled (tracked as this spec's only open item).
