# Spec: short-command surface — no workflow requires raw tanuki-ledger

Status: PROPOSED 2026-07-17 (drafted on the operator's order, consolidating
the accepted ledger-UX findings F18, F20–F23, F33, F36, F60, F62, F66, F68,
F70, F72, F76 and the operator's stated principle). Touches
`commands/tanuki.md`, `tools/tanuki-ledger`, `docs/tanuki-spec.md`. No change
to the pipeline contract (events → findings → proposals → labeled issues) or
to any isolation/safety property.

## Problem

Finishing a run's findings today requires a series of raw `tanuki-ledger`
invocations with long options (`promote` that doesn't transition status, an
unhinted `set-status` per finding, hand-built `gh issue create` calls). The
operator's confusion is not one bad flag — it is that **the human workflow
leaks into the machine substrate**. The dogfood ledger agrees: 16 accepted
findings are `tanuki-ledger` UX complaints, and the most chronic one (F23,
recurrence 7) is exactly the promote/set-status seam.

## Principle (the one-line contract)

**Every human workflow completes through a short, option-free command; every
required input is resolved by configuration, state, or an interactive
question — never by memorized flags.** The `tools/tanuki-*` executables
remain the deterministic, fully-optioned machine substrate (the contract for
scripts, tests, and headless runs); the `commands/*.md` layer is the only
surface a human is expected to type. A workflow step that can only be
completed by a raw tool invocation is a UX defect of the same class as a
missing feature.

This is a restatement, not a new rule: the split "deterministic work →
small scripts; judgment work → agent prompts that call them" is a standing
owner decision (owner decision record 2026-07-06, routing principle). The
target resolution order in `commands/tanuki.md` (explicit arg → registry →
single target → picker) is the existing model; this spec extends it from
"which target" to "every required option".

**The one exception — `tanuki-loop` headless mode.** An unattended run has
no human to ask, so its contract stays explicit flags + config file, fully
specified up front (see `specs/spec-tanuki-loop/SPEC.md`). Interactive
resolution is a convenience of attended commands only; nothing in this spec
adds a prompt to the headless path.

## Deliverables (ranked)

### D1 — the decision pass fully wraps the mechanics (fixes F23)

`/tanuki <target> --brief` (and the end of a normal run) must be the
*complete* path from proposals to labeled issues, per the existing contract
("the human gate is part of the run, not homework", `docs/tanuki-spec.md`):

- Promotion moves status to `proposed` as the contract already states; the
  command layer performs the transition (the tool may keep a `--dry-run`
  preview). No separate unhinted `set-status` call.
- The pass walks each proposal top-down and asks for a disposition —
  **accept / dismiss / defer** — via the interactive question interface,
  small batches, no default disposition (a multi-outcome gate is never
  collapsed into a defaulted checkbox). Accept offers filing the issue right
  then, still explicitly confirmed; the command builds the `gh issue create`
  call itself (title = human-readable problem statement, one `<prefix>:<kind>`
  label, finding id in the body footer — the existing downstream boundary,
  unchanged).
- Below-bar ("watching") findings stay listed and remain acceptable at the
  gate — the promotion bar gates surfacing, not permission (owner decision
  record 2026-07-15, promotion-bar ruling). The pass must offer a way to
  accept a watching item without hand-running `set-status`.
- Dispositions are written back via the tool (`set-status` stays the
  substrate); the command reports what changed and what remains undecided.

Acceptance: an operator can go from "run finished" to "issues filed,
ledger updated" answering questions only — zero raw `tanuki-ledger` or
`gh` invocations typed by hand.

### D2 — `next` derived from enumerated ledger states (fixes F24, F36, F76)

The status "next:" hint has been fixed twice and regressed twice because it
is patched string logic. Replace it with a derivation over an explicit,
closed enumeration of ledger states — at minimum: no ledger / events but no
findings / findings all below bar / proposed awaiting decision / accepted
awaiting fix-verification / all findings tombstoned or decided — each
mapping to exactly one next command (a `/tanuki` form where one exists, the
tool form otherwise). One test fixture per state in `tools/tests/`, so a new
state or a regression fails visibly. `status` prints the derived line;
a `next` subcommand exposes the same derivation alone.

### D3 — help text carries the contract (fixes F20, F33, F60, F62, F66, F68, F72)

For operators (and models) that do touch the substrate directly, `--help`
must state the full contract in place:

- `upsert-finding --kind`: list the closed enum (`friction|papercut|gap`).
- `set-status --status`: list `open|proposed|accepted|dismissed`, and name
  the canonical path (`open` → `proposed` → `accepted`/`dismissed`) while
  stating that transitions are recorded, never rejected — the tool prints a
  one-line notice on a non-canonical transition (e.g. `open` → `accepted`,
  legitimate for a watching-list accept) and sequencing stays with the
  command layer (`docs/tanuki-spec.md`, Finding lifecycle).
- `upsert-finding --match`: state that it takes a finding id and bumps
  recurrence instead of creating a duplicate.
- `ingest`: include a one-line JSONL event example and the evidence-pointer
  format.
- `promote`: state the default `--min-recurrence` / `--min-scenarios`
  values, and print the active thresholds in the output so an empty result
  is unambiguous.
- `upsert-finding`: document recurrence semantics (counts upsert calls, not
  evidence pointers).
- `policy-surface`: one sentence on what the policy source is and why an
  operator would configure it (pointing at `specs/spec-policy-advisory/`).

### D4 — no silent state changes (fixes F18, F21, F70)

- `--target <slug>` that resolves to a directory that does not yet exist is
  announced ("creating new target namespace ~/.tanuki/<slug>/") on write
  commands, so a typo cannot silently fork a ledger.
- `ingest` either registers the run it ingests or prints that it did not
  ("events recorded; run not registered — pass --run-id to advance
  compact's unseen-runs counter"), so `runs: 0` after a successful ingest
  stops reading as a failure.
- The "N exact duplicate(s) collapsed" message names the collapsed event
  ids and what they collapsed into.

### D5 — the human never needs raw JSON

`findings`, `promote`, and `stats` keep JSON as the machine contract, but
the *command layer* renders every view a workflow needs (brief, status,
history, the decision pass). If a gap forces an operator to read
`tanuki-ledger findings` JSON by eye, that gap is a defect under the
Principle above — file it as a finding, not a habit.

## Non-goals (constraints folded in, so implementation needs no attachments)

- **No auto-filing, no unattended dispositions.** Issues are filed only
  inside an attended decision pass with explicit confirmation; the loop's
  headless phases never file, merge, or push (existing contract,
  `docs/tanuki-spec.md` standing constraints — unchanged by this spec).
- **No downstream awareness.** The pipeline still ends at the labeled
  issue. No tanuki command or tool invokes, names, or configures downstream
  triage/implementation tooling; downstream systems may rely only on the
  `<prefix>:<kind>` labels plus the issue body (existing downstream
  boundary). An operator wanting a one-shot "decide → file → start triage"
  chain composes it *outside* this repo, on their side of the label
  boundary; such a wrapper is expressly out of scope here.
- **No new prompts in headless mode.** `tanuki-loop` unattended runs keep
  their explicit-flags/config contract; interactive resolution never
  becomes a dependency of any unattended phase.
- **No second status vocabulary, no per-source bars.** The status set and
  the uniform promotion bar are unchanged; D1 changes *who executes the
  transition*, not what transitions exist (uniform-bar ruling, owner
  decision record 2026-07-15).
- **No ledger reads outside bounded views.** D2/D5 stay within the existing
  bounded-subcommand-view rule; nothing new loads `ledger.json` whole.

## Ordering

D1 and D2 first (they remove the daily confusion and the chronic F23/F24
class); D3 and D4 are mechanical and can land in one batch; D5 is a standing
acceptance rule rather than a one-time change. Each deliverable cites the
finding ids it closes so post-fix runs verify by absence (recurrence stops).
