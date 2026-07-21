# Spec: gate-side policy advisory — opt-in policy_source consulted at the brief/decision pass

Status: RATIFIED 2026-07-15 by the operator, as written. Extends
`docs/tanuki-spec.md` (Consolidator/brief and decision-pass contracts only —
Driver, Miner, ingest, ledger, and scheduler are untouched). AMENDED
2026-07-21 (triage of #271, owner decision record — policy-advisory extension
adoption): two opt-in features added — **consult-first at the gate** (§6) and
**policy-divergence flagging** (§7). Both are active only when `policy_source`
is configured and are behaviorally absent otherwise; §4 and §8 are amended to carve
the single bounded exception consult-first requires. No new config surface, and
the §1–§3 boundary (Driver/Miner/ingest/ledger/scheduler untouched) is
preserved.

Problem: findings and their proposed fixes can conflict with decisions the
operator has already made and recorded elsewhere — a proposal can be
individually sound yet contradict the product's ratified scope or an earlier
architectural ruling — and today the operator must carry all of that recorded
policy in their head at the decision pass. An input-side guardrail was
considered and **rejected**: filtering or classifying feedback at ingest
deletes the recurrence signal the ledger exists to accumulate, and violates
the capture-records-raw rule (`--ingest` stores the note verbatim, unjudged).
Policy binds *proposals*, not observations, so the correct attachment point is
the one place judgment already happens: the brief and the human gate.

## 1. Configuration (opt-in, Tanuki-owned)

A `policy_source` block in `~/.tanuki/config.json` (global) or a target file's
`"defaults"` block (per-target override, normal precedence):

```json
"policy_source": {
  "path": "~/path/to/policy-repo",
  "files": ["GLOSSARY.md", "LESSONS.md", "topics/decisions.md"]
}
```

- `path` — a local git checkout of a policy repository the operator declares:
  any repo holding the operator's recorded decisions, conventions, or product
  policy as plain markdown. Required if the block is present.
- `files` — the read allowlist: ≤4 repo-relative markdown paths. This is the
  *entire* readable surface; Tanuki assumes nothing about the policy repo's
  layout beyond this list (repo-specific structure belongs in the operator's
  config, never in Tanuki's code).
- Block absent → the feature does not exist: zero behavior change anywhere.
  **Nothing is ever written into the policy repo or any target repo** — the
  existing non-negotiable holds.

## 2. Reader (deterministic tool, code-enforced bounds)

`tanuki-ledger` gains a `policy-surface` subcommand (or a small sibling tool):
resolves `path`, reads **only** the `files` allowlist (passed in code — every
other path under the policy repo is structurally unreadable), records the pin
`<policy-repo>@<commit>` (`git rev-parse HEAD`), and emits the lines with
`file:line@commit` pointers. Read-only. Degradation: path absent, unreadable,
or not a git repo → one log line, feature silently off for that run — the
policy source is an enhancer, never a dependency. Stdout only — policy text is
never persisted into the ledger, events, or scheduler state.

## 3. Where it runs (and where it must not)

- **Runs at:** step 3 (Consolidate/brief) and step 4 (Decide) — the frontier
  tier, per the model-tier table. Tension detection is frontier judgment,
  never delegated to cheap models and never reduced to code matching; the
  *reader* is deterministic code.
- **Never at:** `--ingest` (the human's note is recorded verbatim, unjudged —
  hard rule), the Driver, the extraction subagent, or scheduler arithmetic.
  The policy surface is a judgment-time input to the gate, not an event
  source: nothing read from it may enter the ledger as an event or finding.

## 4. Output (advisory, never blocking)

- **Brief:** a proposal whose problem or proposed fix tensions with a policy
  line gains one advisory line, placed with the evidence line:
  `policy: tension — <one clause> vs "<policy quote>" (file:line@commit)`.
  No tension → no line. The brief's ranking is unaffected by policy flags.
- **Gate:** the same line is shown as context when the proposal is walked at
  the decision pass. The flag never auto-dismisses, downgrades, or reorders a
  proposal, and never pre-selects a disposition — accept/dismiss/defer remain
  entirely the human's. A dismissal motivated by policy is recorded like any
  other dismissal (deduped-against; never resurfaces as new). **The one bounded
  exception is consult-first fork resolution (§6):** a *decision-point fork*
  (not a proposal's disposition) whose answer the policy source covers may be
  auto-resolved to an **overrideable FYI** — never machine-final, always the
  human's to override, and only when `policy_source` is configured. Proposal
  dispositions themselves remain entirely the human's, unchanged.
- **Audit:** any run whose brief carries policy lines appends a `consulted:`
  line (which allowlisted files/lines were read; which applied) to the brief.

## 6. Consult-first at the gate (`--brief`, opt-in — ADDED 2026-07-21, triage of #271)

At the decision pass, before surfacing a **decision-point fork** (a
policy/architecture/prior-decision fork the flow would otherwise ask the human
to resolve), the gate consults the configured policy source via the §2 reader:

- **Covered fork → overrideable FYI.** When the policy source covers the fork,
  the gate **auto-resolves** it and demotes it to an FYI carrying the **chosen
  option** and a **pinned verbatim quote** (`file:line@commit`). The FYI is
  always **overrideable** — the human may reopen and re-decide it — and is
  **never machine-final**: the auto-resolution is a labor-saving default, not a
  decision the tool owns. This is the single bounded relaxation of §4's
  "never pre-selects" rule, and it applies to *forks*, never to a proposal's
  accept/dismiss/defer disposition.
- **Miss → escalate with candidates.** When the policy source does not cover
  the fork, it escalates to the human as today, carrying **up to 3 pinned
  candidate answers** drawn from the allowlisted reads (each `file:line@commit`)
  — never a machine-final choice, only ranked material for the human's decision.
- **Attribution.** Every auto-resolved FYI and every candidate set is attributed
  in the run output as **policy advisory** and recorded in the `consulted:`
  audit line (§4), naming which allowlisted lines drove it.
- **Opt-in and silent-when-unconfigured.** `--brief` consult-first is active
  **only** when `policy_source` is configured; a run without it is
  byte-identical to today (no forks auto-resolved, no candidates attached). No
  new config key: activation rides the existing `policy_source` block.

## 7. Policy-divergence flagging (ADDED 2026-07-21, triage of #271)

At the existing pinned policy reads (§4 brief/gate), the gate additionally flags
**implementation decisions that contradict or have outgrown** a quoted policy
line — a divergence between what the code/plan does and what the recorded policy
says:

- Flags are **divergence candidates**, emitted as **proposal-only** items —
  never auto-applied, never authoritative, never a disposition. Each carries the
  divergence clause and the pinned quote (`file:line@commit`), attributed as
  policy advisory.
- The human decides what to do with a divergence candidate exactly as with any
  proposal; nothing about the code, the plan, or the policy repo is written.
- Active only when `policy_source` is configured; silent (byte-identical to
  today) otherwise.

**Generic vocabulary (both §6 and §7).** Zero policy-source-specific identity in
code, UX, or docs — the policy source is any operator-declared markdown repo
(§1), and the consult-first / divergence surfaces name it only generically.
**Publication-boundary lint** (the existing check that policy text never leaks
into the ledger, events, or scheduler state) **extends to these new surfaces**:
FYI quotes, candidate answers, and divergence flags are run-output/audit only,
never persisted as events, findings, or scheduler state.

## 8. Non-goals

- Ingest-side filtering, validation, or classification of human feedback
  against policy — rejected by design, not deferred.
- A served policy source (API, MCP, or any network transport) — the local
  file pointer is the transport. This spec takes no position on whether such
  a service should ever exist; if one does, it is the policy repo's concern,
  not Tanuki's.
- Reading anything outside the configured `files` allowlist.
- Auto-disposition of **proposals** (accept/dismiss/defer), severity changes, or
  ranking changes driven by policy. (The §6 consult-first exception resolves
  **decision-point forks** to an *overrideable, never-machine-final* FYI — it
  does not dispose of proposals, and the human owns every final call.)
- Caching or mirroring policy content anywhere under `~/.tanuki/` beyond the
  single run's brief text.

## Success signal

With `policy_source` configured: a proposal that contradicts a recorded policy
line reaches the gate carrying its quoted, pinned tension line, and the
operator decides with the quote in view. With it unset or broken:
byte-identical behavior to today. A code test requesting any path outside the
allowlist through the reader is refused.
