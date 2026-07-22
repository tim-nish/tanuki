# Spec: den-distill — proposal-only contribute-back, emitted into a directory this repo owns

Status: RATIFIED 2026-07-19 (triage of issues #178/#179; the consuming-side
half was ratified 2026-07-13 by owner decision — a consuming intake accepts
machine-proposed items behind its own human gate; the den stays raw
history and never becomes a second knowledge store). AMENDED 2026-07-22
(triage of #278, owner decision record 2026-07-22 (contribute-back emission
target)): the configured path is a directory **this repo owns**, swept by
the consuming side, not the consuming side's own intake directory — see §1.
Amends
`docs/tanuki-spec.md` in two places: the "Prototype deviations" note on
lesson candidates (the "a human moves them" promise this spec replaces with
a gated mechanism) and the decision-pass / acceptance-rule sections (lesson
candidates become first-class walked items). Design records: #178
(mechanism), #179 (workflow surface).

Problem: the consolidator already produces lesson-shaped conclusions, but
they dead-end as prose in the brief — no output artifact, no schema, no
disposition tracking, no dedupe against candidates already moved. The most
durable output of the pipeline (patterns, not single fixes) depends on
unassisted clerical work outside any command, while a ratified consuming
intake sits waiting for exactly these proposals.

## 1. The boundary (the ratified core)

Contribute-back is **proposal-only, schema-conforming,
emission-directory-only**:

- Tanuki writes **only** into the configured emission directory — **a
  directory this repo owns** (a run workspace, or this repo's own working
  tree), which the consuming side's intake sweeps on its own schedule.
  Tanuki does not write into the consuming side's tree at all. A test
  fixture asserts the write-path allowlist.
- The emitted file is identical either way; only the destination and how it
  is described change. The write-path invariant reads "no write outside the
  configured emission directory" — a self-contained claim about this repo,
  not an allowlist naming a foreign directory.
- Acceptance into the consuming side's actual knowledge surface remains
  entirely its own gated process; dismissal at that gate needs no
  back-channel. Collection latency between emission and arrival belongs to
  the consuming side, not to this repo.
- The den (`~/.tanuki/` ledger + briefs) stays raw history — emitting a
  proposal copies *out*; nothing turns the den into a knowledge store
  consumers read.
- **Attended gate on every write; emission is the completing action.** Files
  are written only on an explicit per-item accept in-session, and the accept
  *completes* when the file is emitted into this repo's own directory —
  arrival on the consuming side is not this repo's event and never gates the
  decision pass. Mirrors "issue-shaped, ready to paste — but
  never auto-filed". `/tanuki-loop` renders candidates in its brief but
  never walks the gate and never emits (attended-gate work, same class as
  issue filing).

## 2. Configuration — opt-in `contribute_back`

Sibling of `policy_source` and equally narrow in the other direction:
`{"path": <emission directory owned by this repo>, "schema": <intake
template id or file>}`.
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
another target's session). Rationale: `path` names **one target's emission
workspace**; resolving it globally lets one target — or a stale scratch run
— silently route *every* target's lessons into that one workspace, mutating
shared machine state. This is the one exception to the general
`~/.tanuki/config.json < target defaults < CLI` precedence
(`docs/tanuki-spec.md`): for `contribute_back` the global tier does not
participate at all.

## 3. The emit tool (deterministic, zero-dep per the routing rule)

`tanuki-ledger distill` (or equivalent subcommand — no new top-level
command):

- **Selection:** lesson-shaped candidates that cleared the **existing**
  promotion bar (chronic ≥3 / breadth ≥2 — no new thresholds). Judgment
  already happened at promotion; rendering is mechanical.
- **Rendering:** one intake-shaped file per accepted candidate, conforming
  to the configured schema template — frontmatter: slug, created date,
  `source_repo` (the tanuki target), perishable flag, tags; body: the
  candidate stated in full sentences.
- **Pinned provenance, compact:** finding id(s), recurrence/breadth counts,
  run-manifest pointers — the same claim–evidence binding the brief already
  enforces, so the consuming-side gate audits without re-deriving.
- **Ledger transition `contributed`** (analogous to `proposed` →
  issue-filed): an emitted candidate is recorded so re-runs bump recurrence
  but never re-emit a duplicate file. A candidate dismissed at the
  gate is never emitted and stays deduped-against.

## 4. The workflow surface (one attended command completes the loop)

- **Lesson candidates join the decision pass.** After the proposal walk, the
  pass walks lesson candidates the same way — accept / dismiss / defer per
  item, never a pre-selected default. **Accept writes the file right then**
  (via the emit tool) into this repo's emission directory, exactly parallel
  to "accept → optionally file the issue right then". The emission is the
  completing action of the accept — nothing waits on the consuming side.
  One attended `/tanuki <target>` run goes
  drive → mine → brief → decisions → emitted proposals, zero
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
  in the emission directory, carrying pins; ledger records `contributed`; a
  later run re-observing the pattern bumps recurrence, emits nothing.
- A dismissed candidate is never emitted and stays deduped-against.
- No write ever lands outside the configured emission directory (fixture-
  asserted allowlist).
- The picker's "distill den" walk over a pre-existing ledger behaves
  identically to the in-run pass.

## Decomposition

Stories 1.34 (umbrella #178 — config + doctor + emit tool + `contributed`
status), 1.35 (umbrella #179 — decision-pass walk + accept-writes + receipts),
1.36 (umbrella #179 — picker action + unconfigured notice) — created
`ready`, flowing through /publish-issues → /implement-story.

The 2026-07-22 amendment (#278) decomposes to **no stories**: the shipped
write-path allowlist is already "inside the configured directory"
(`tools/tanuki-ledger:1768-1773`), so the reframing needs no code change and
the existing fixture (`tools/tests/test-ledger-distill:123`) still asserts
the invariant as now stated. Known residue, deliberately left: the doctor
error string at `tools/tanuki-ledger:1536-1537` still says "the hub staging
directory" — stale wording, not an invariant.
