# Spec: /tanuki-solve — consolidate-then-decide pass from ledger to labeled issues

Status: IMPLEMENTED 2026-07-17; **AMENDED 2026-07-17 — folded into the
decision pass** (issue #72, see Non-goals). D1/D3 stay normative as the
consolidation contract, but their consuming surface is now
`/tanuki <t> decide` per `specs/spec-short-command-surface/SPEC.md` D1, not
a separate `/tanuki-solve` command; that command is retired once the fold
lands, its capability absorbed rather than dropped. D2 in
`tools/tanuki-ledger` (`consolidate` subcommand) is unaffected; D4 in
`tools/tests/test-ledger-consolidate` (drafted and implemented on the
operator's order, same sitting) now applies to the folded pass. Extends
`specs/spec-short-command-surface/SPEC.md` D1 (the wrapped decision pass);
changes nothing in the pipeline contract (events → findings → proposals →
labeled issues) or any isolation/safety property. Origin: a live incident —
two findings carrying **contradictory proposals** for the same defect
(reject-invalid-input vs document-the-accident, issues #68/#70) were filed
as separate issues in one sitting, because the decision pass presented
findings one at a time and the conflict was never surfaced at the approval
screen. Downstream, the contradiction had to be reconciled by hand across
two issue threads (owner decision record 2026-07-17, coordinated
disposition).

## Problem

The decision pass walks proposals **item by item**. That shape silently
assumes findings are independent. They are not: a run family produces
overlapping findings (same defect seen by different scenarios), contradictory
findings (two incompatible proposals for one behavior), and dependent
findings (one disposition reframes another). Presenting each as an isolated
accept/dismiss/defer question collapses what is really a multi-outcome
arbitration into a sequence of defaulted-feeling binary gates — the exact
failure the gate design forbids (a multi-outcome gate is never collapsed
into independent checkboxes; owner decision record 2026-07-16,
single-axis-gate ruling). The result is filed issues that conflict with each
other, discovered only downstream where reconciliation is most expensive.

## Principle (the one-line contract)

**No finding reaches the approval screen unanalyzed: the solve pass first
consolidates — merge overlaps, surface contradictions as one decision,
order by dependency — and only then asks for dispositions.** Filing an
issue whose conflict with another filed issue was detectable from the ledger
is a defect of this command, not of the operator's attention.

## The command

`/tanuki-solve <target>` (empty target resolves like `/tanuki`: explicit arg
→ registry → single target → picker). Attended only. It is the complete
path from "run finished" to "issues filed, ledger updated":

1. **Orient** — `status` + `next` counts; if `next` says accepted fixes
   await re-verification, say so and continue with what is decidable now
   (this command decides and files; it never drives runs).
2. **Consolidate** (new — see Deliverables) — build the presentation plan:
   merge groups, conflict groups, dependency order.
3. **Decide** — the D1 decision-pass contract, unchanged in its mechanics
   (interactive questions, small batches, accept/dismiss/defer, no default
   disposition, dispositions written back through the tool), but walked
   **over the consolidated plan, not the raw finding list**.
4. **File** — per accepted item, explicit confirmation, `gh issue create`
   with exactly one `<prefix>:<kind>` label and every constituent finding id
   in the body footer (existing downstream boundary, unchanged).
5. **Watching list** — below-bar findings presented once, acceptable at the
   gate (the bar gates surfacing, not permission; owner decision record
   2026-07-15, promotion-bar ruling). Consolidation applies here too: a
   watching item that conflicts or merges with a proposal above the bar is
   shown *inside* that group, not in the tail list.
6. **Summary** — per finding: id → disposition → issue URL if filed; merged
   and conflict groups reported as groups; what remains open/proposed.

## Deliverables (ranked)

### D1 — the consolidation stage

Between promotion and presentation, the pass classifies every pair/group of
candidate items (proposals + watching) into:

- **Merge group** — same defect surfaced by different scenarios or runs
  (overlapping evidence pointers, same subsystem + same behavior). Presented
  as ONE item with combined evidence and the union of finding ids; one
  disposition; if accepted and filed, one issue whose footer lists every
  constituent finding id, and every constituent finding gets the same
  disposition and issue URL written back. Ledger entries are never rewritten
  or deleted — merging is a presentation and filing act, recorded via
  status + shared issue URL, not history surgery.
- **Conflict group** — two or more findings whose proposals cannot both
  hold (mutually exclusive fixes for the same behavior). Presented as ONE
  multi-outcome question naming the branches explicitly ("A: reject
  path-like targets; B: document the mode — choose one, or defer the
  group"); the chosen branch becomes the disposition context for every
  member (winner accepted, loser dismissed-with-reason or absorbed into the
  winner's issue body as a rejected alternative). A conflict group is never
  split back into independent yes/no questions.
- **Dependency edge** — a finding whose disposition reframes another
  (e.g. a lifecycle-enforcement decision that changes what counts as valid
  for a status-display finding). The reframing item is asked first; the
  dependent item's presentation states the dependency ("given the choice on
  F63, this proposal now reads…"). Ordering rule: most recontextualizing
  first (owner decision record 2026-07-16, gate-ordering ruling).
- **Independent** — everything else; walked as today.

### D2 — deterministic pre-clustering in the substrate

The mechanical share of D1 lives in the tool layer, per the standing
routing principle (deterministic work → small scripts; judgment work → the
command layer; owner decision record 2026-07-06): a read-only
`tanuki-ledger consolidate` subcommand emits candidate groups as JSON —
clustering by evidence-pointer overlap, shared scenario family, shared
subsystem path, and proposal-text similarity — with a reason per candidate
group. It proposes, never decides: the command layer confirms or discards
each candidate group by judgment (semantic contradiction is not reliably
mechanical), and may add groups the clustering missed. `consolidate` reads
bounded views only (no whole-ledger load beyond the existing bounded-view
rule) and writes nothing.

### D3 — conflict evidence survives into the issues

When a conflict group is resolved, the filed issue's body records the
arbitration: the branches considered, the chosen branch, and the constituent
finding ids of the losing branch. This keeps the downstream record
self-contained (a reader of the single filed issue sees that an alternative
existed and was rejected) without any downstream tooling awareness.

### D4 — acceptance test for the incident class

A fixture ledger containing two findings with contradictory proposals for
the same behavior must produce: one conflict-group question, and — under
every possible answer — at most one filed issue for that behavior. The
#68/#70 shape becoming unrepresentable is the acceptance criterion for this
spec.

## Non-goals (constraints folded in, so implementation needs no attachments)

- **No downstream awareness.** The pipeline still ends at the labeled
  issue. This command does not invoke, name, or configure triage or
  implementation tooling; downstream systems may rely only on the
  `<prefix>:<kind>` labels plus the issue body. An operator wanting a
  one-shot "decide → file → start triage" chain composes it outside this
  repo, on their side of the label boundary (existing boundary, restated
  from `specs/spec-short-command-surface/SPEC.md` Non-goals — a private
  wrapper that calls this command and then the operator's own downstream
  tooling is expressly anticipated and expressly out of scope here).
- **No auto-filing, no unattended dispositions, no headless mode.** The
  loop's unattended phases never consolidate, decide, or file; this command
  is attended-only and every disposition is a human answer in the session.
- **No ledger history surgery.** Consolidation never rewrites, merges, or
  deletes ledger entries; groups exist at presentation/filing time and are
  recorded through the existing status vocabulary plus shared issue URLs.
  No second status vocabulary; the uniform promotion bar is unchanged.
- **No LLM-inner-loop clustering.** `consolidate`'s clustering is
  deterministic (pointer/path/scenario overlap + text similarity); judgment
  is applied once per candidate group at the command layer, never in a
  generate-and-filter loop.
- **~~`/tanuki <t> --brief` keeps working~~ — RESOLVED 2026-07-17, the fold
  fired.** The rule this bullet recorded ("if maintaining both surfaces
  diverges, the resolution is folding solve's consolidation stage into the
  brief pass, not a second contract") met its trigger: issue #72 observed
  that the brief pass walks the raw finding list with no consolidation, so
  any sitting deciding through `--brief` or the end of a normal run could
  reproduce the #68/#70 incident this spec exists to prevent. Per the rule,
  the consolidation stage is folded into the decision pass —
  `specs/spec-short-command-surface/SPEC.md` D1 now requires it, cites D1/D3
  below as the normative taxonomy, and carries the ledger-anchored entry the
  workflow needs — surfaced as `/tanuki <t> decide`, with `--brief`
  retained as an alias (the flag names the workflow, not the artifact, since
  the pass no longer requires a brief). `/tanuki-solve` is consequently retired as a separate
  surface: its capability (one attended command from open findings to
  labeled issues, no raw ledger invocations) is not lost but absorbed —
  retirement is conditional on that capability being wholly reachable
  through `/tanuki <t> --brief`, which the amended D1 requires. This spec's
  D1/D3 remain normative as the consolidation contract; only the consuming
  surface changed. D2 (`tanuki-ledger consolidate`) is untouched.

## Ordering

D2 first (the substrate emitter is independently testable), then D1/D3 in
one batch (the command is the consumer), D4 alongside D1. Each deliverable
cites the incident it closes; post-fix solve sittings verify by absence —
no cross-issue contradiction discovered downstream.
