# Spec: gate-side policy advisory — opt-in policy_source consulted at the brief/decision pass

Status: RATIFIED 2026-07-15 by the operator, as written. Extends
`docs/tanuki-spec.md` (Consolidator/brief and decision-pass contracts only —
Driver, Miner, ingest, ledger, and scheduler are untouched).

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
  other dismissal (deduped-against; never resurfaces as new).
- **Audit:** any run whose brief carries policy lines appends a `consulted:`
  line (which allowlisted files/lines were read; which applied) to the brief.

## 5. Non-goals

- Ingest-side filtering, validation, or classification of human feedback
  against policy — rejected by design, not deferred.
- A served policy source (API, MCP, or any network transport) — the local
  file pointer is the transport. This spec takes no position on whether such
  a service should ever exist; if one does, it is the policy repo's concern,
  not Tanuki's.
- Reading anything outside the configured `files` allowlist.
- Auto-disposition, severity changes, or ranking changes driven by policy.
- Caching or mirroring policy content anywhere under `~/.tanuki/` beyond the
  single run's brief text.

## Success signal

With `policy_source` configured: a proposal that contradicts a recorded policy
line reaches the gate carrying its quoted, pinned tension line, and the
operator decides with the quote in view. With it unset or broken:
byte-identical behavior to today. A code test requesting any path outside the
allowlist through the reader is refused.
