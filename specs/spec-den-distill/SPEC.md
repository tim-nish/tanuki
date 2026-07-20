# Spec: den-distill — proposal-only contribute-back into the knowledge hub's staging intake

Status: RATIFIED 2026-07-19 (triage of issues #178/#179; the hub-side half
was ratified 2026-07-13 by owner decision — staging intake accepts
machine-proposed items behind the hub's own human gate; the den stays raw
history and never becomes a second knowledge store). Amends
`docs/tanuki-spec.md` in two places: the "Prototype deviations" note on
lesson candidates (the "a human moves them" promise this spec replaces with
a gated mechanism) and the decision-pass / acceptance-rule sections (lesson
candidates become first-class walked items). Design records: #178
(mechanism), #179 (workflow surface).

Problem: the consolidator already produces lesson-shaped conclusions, but
they dead-end as prose in the brief — no output artifact, no schema, no
disposition tracking, no dedupe against candidates already moved. The most
durable output of the pipeline (patterns, not single fixes) depends on
unassisted clerical work outside any command, while the hub's ratified
staging intake sits waiting for exactly these proposals.

## 1. The boundary (the ratified core)

Contribute-back is **proposal-only, schema-conforming,
staging-directory-only**:

- Tanuki writes **only** into the configured staging directory — never the
  hub's recall/policy surface, never any other hub path. A test fixture
  asserts the write-path allowlist.
- Acceptance into the hub's actual knowledge surface remains entirely the
  hub's own gated process; dismissal at the hub-side gate needs no
  back-channel.
- The den (`~/.tanuki/` ledger + briefs) stays raw history — emitting a
  staging proposal copies *out*; nothing turns the den into a knowledge
  store consumers read.
- **Attended gate on every write.** Files are written only on an explicit
  per-item accept in-session, mirroring "issue-shaped, ready to paste — but
  never auto-filed". `/tanuki-loop` renders candidates in its brief but
  never walks the gate and never emits (attended-gate work, same class as
  issue filing).

## 2. Configuration — opt-in `contribute_back`

Sibling of `policy_source` and equally narrow in the other direction:
`{"path": <hub staging directory>, "schema": <intake template id or file>}`.
Unset ⇒ the feature does not exist (byte-identical behavior to today).
Doctor validates the block — path exists and is a directory, schema template
resolves — before any drive consumes it. Once the declared-input
configuration surface (#172, stories 1.27–1.29) lands, the block is
declared/edited there; until then, a doctor-validated hand edit.

**Scope is per-target, never machine-wide (issue #241 / F198).** `contribute_back`
is resolved **only** from a target's own configuration — its scenarios-file
`defaults` block, or the per-target declared-input surface once it lands —
**never** from the machine-wide `~/.tanuki/config.json`. A `contribute_back`
block found at global scope is **ignored** for resolution and **flagged by
doctor** (a global-scope contribution target is almost always a leftover from
another target's session). Rationale: the staging `path` names one target's
knowledge hub; resolving it globally lets one target — or a stale scratch run
— silently route *every* target's lessons into that hub, mutating shared
machine state. This is the one exception to the general
`~/.tanuki/config.json < target defaults < CLI` precedence
(`docs/tanuki-spec.md`): for `contribute_back` the global tier does not
participate at all.

## 3. The emit tool (deterministic, zero-dep per the routing rule)

`tanuki-ledger distill` (or equivalent subcommand — no new top-level
command):

- **Selection:** lesson-shaped candidates that cleared the **existing**
  promotion bar (chronic ≥3 / breadth ≥2 — no new thresholds). Judgment
  already happened at promotion; rendering is mechanical.
- **Rendering:** one staging-intake file per accepted candidate, conforming
  to the configured schema template — frontmatter: slug, created date,
  `source_repo` (the tanuki target), perishable flag, tags; body: the
  candidate stated in full sentences.
- **Pinned provenance, compact:** finding id(s), recurrence/breadth counts,
  run-manifest pointers — the same claim–evidence binding the brief already
  enforces, so the hub-side gate audits without re-deriving.
- **Ledger transition `contributed`** (analogous to `proposed` →
  issue-filed): an emitted candidate is recorded so re-runs bump recurrence
  but never re-emit a duplicate staging file. A candidate dismissed at the
  gate is never emitted and stays deduped-against.

## 4. The workflow surface (one attended command completes the loop)

- **Lesson candidates join the decision pass.** After the proposal walk, the
  pass walks lesson candidates the same way — accept / dismiss / defer per
  item, never a pre-selected default. **Accept writes the staging file right
  then** (via the emit tool), exactly parallel to "accept → optionally file
  the issue right then". One attended `/tanuki <target>` run goes
  drive → mine → brief → decisions → contributed staging proposals, zero
  steps outside the command.
- **"Distill den" action in the existing picker, no new top-level command:**
  runs the same candidate walk over the *existing* ledger without driving
  anything — contributing past history never requires a new run.
  `/tanuki <target> distill` is acceptable as direct syntax only if the
  grammar requires it.
- **Discoverability when unconfigured:** candidates present but
  `contribute_back` unset ⇒ the brief and the decision pass say so in one
  line ("N lesson candidates; contribute-back not configured — …") instead
  of silently rendering prose. Nothing is written.
- **Receipts in-session:** each accept prints the written file path; the run
  summary counts contributed items alongside filed issues.
- Dispositions are written back by the command (`set-status`), never typed
  by the user; deferred candidates stay visible in `status`.

## Acceptance

- `contribute_back` unset: byte-identical to today.
- Set: an accepted chronic candidate produces exactly one schema-valid file
  in the staging directory, carrying pins; ledger records `contributed`; a
  later run re-observing the pattern bumps recurrence, emits nothing.
- A dismissed candidate is never emitted and stays deduped-against.
- No write ever lands outside the configured staging directory (fixture-
  asserted allowlist).
- The picker's "distill den" walk over a pre-existing ledger behaves
  identically to the in-run pass.

## Decomposition

Stories 1.34 (umbrella #178 — config + doctor + emit tool + `contributed`
status), 1.35 (umbrella #179 — decision-pass walk + accept-writes + receipts),
1.36 (umbrella #179 — picker action + unconfigured notice) — created
`ready`, flowing through /publish-issues → /implement-story.
