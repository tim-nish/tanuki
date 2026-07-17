Tanuki solve — the consolidate-then-decide pass: take a target's candidate
findings from the ledger to labeled GitHub issues in one attended sitting,
with consolidation (merge / conflict / dependency analysis) BEFORE any
approval question. Contract: `${CLAUDE_PLUGIN_ROOT}/specs/spec-tanuki-solve/SPEC.md`;
the decision-pass mechanics are `commands/tanuki.md` §"the decision pass"
(spec-short-command-surface D1) — this command changes what is *presented*,
never the disposition mechanics.

Every `tanuki-*` tool named below is the executable under
`${CLAUDE_PLUGIN_ROOT}/tools/` (not on PATH — invoke by full path).

Boundaries (the contract, restated):
- **Attended only.** Never run from a headless/unattended context; every
  disposition and every issue filed is a human answer in this session.
- **No default disposition.** Accept / dismiss / defer are always explicit;
  a conflict group is ONE multi-outcome question, never a sequence of
  independent yes/no gates.
- **Pipeline ends at the labeled issue.** No downstream tooling is invoked,
  named, or configured here (spec-short-command-surface, Non-goals).
- **Ledger writes only through `tanuki-ledger`**; never edit ledger.json.
  Consolidation never rewrites ledger entries — groups live in presentation
  and filing, recorded via statuses and shared issue URLs.

Argument handling ($ARGUMENTS):
- `<target>`: the target namespace slug (e.g. `my-plugin`).
- *(empty)*: resolve like /tanuki — cwd's registered target
  (`tanuki-scheduler resolve --cwd $PWD`) → single configured target →
  interactive picker over `~/.tanuki/scenarios/*.scenarios.json`.

## Steps

1. **Orient.** `tanuki-ledger --target <t> status` and `... next`. Show the
   one-line counts (open/proposed/accepted/dismissed) and the derived next
   step. If `next` says accepted fixes await re-verification, say so — this
   command decides and files; it never drives runs. Offer `/tanuki <t>` for
   that and continue with what IS decidable now.
2. **Promote.** `tanuki-ledger --target <t> promote` — qualifying open
   findings move to `proposed` and are listed.
3. **Consolidate (spec D1 — before anything is presented).**
   a. `tanuki-ledger --target <t> consolidate` — the deterministic
      candidate groups (proposed + open/watching) with mechanical reasons.
   b. Judge each candidate group — confirm or discard, and classify the
      confirmed ones; also add any group the clustering missed (the tool
      proposes, this layer decides):
      - **merge** — same defect, different sightings. One presentation
        item: combined problem statement, union of evidence pointers and
        finding ids.
      - **conflict** — proposals that cannot both hold. One multi-outcome
        question naming each branch explicitly ("A: …; B: …; defer group").
      - **dependency** — one disposition reframes another. Order the
        reframing item first; state the dependency when presenting the
        dependent one.
   c. Build the presentation plan: conflict/dependency groups first (most
      recontextualizing first), then merges, then independents, each lane
      ordered by the ledger's computed priority.
4. **Decide — the decision-pass contract over the plan, not the raw list.**
   Present each plan item (Problem → Proposal, evidence as a pointer line)
   via AskUserQuestion, ≤4 per round:
   - merge group → one disposition; on accept, every constituent finding id
     gets the same disposition (`set-status` per id).
   - conflict group → one question, options = the branches (+ defer). The
     winning branch's finding is accepted; the losing branch's finding is
     dismissed (note the chosen branch via `upsert-finding --match <id>
     --note "superseded by <winner>: <branch>"`) — or, if the human says
     both should be absorbed into one issue, treat as a merge with the
     winning proposal text.
   - independent → accept / dismiss / defer as today.
   Write every disposition back via `tanuki-ledger set-status`; the operator
   types nothing.
5. **File (spec D3 — conflict evidence survives).** On accept, offer —
   separate, explicit confirmation, batch confirmation fine — `gh issue
   create` in the target repo: title = plain problem statement, exactly one
   `<prefix>:<kind>` label (create if missing), and a body footer listing
   EVERY constituent finding id. For a resolved conflict group the body also
   records the arbitration: branches considered, branch chosen, rejected
   alternative(s) with their finding ids — the filed issue is
   self-contained; a reader sees the alternative existed and was rejected.
6. **Watching list.** Present the below-bar findings once, sorted by
   priority — the operator may accept any right there (the bar gates
   surfacing, not permission); same set-status + confirmed-filing path.
   A watching item already inside a confirmed group was presented with its
   group in step 4, not here. Act only on picks; never walk every item.
7. **Summary.** Per finding: id → disposition → issue URL if filed; groups
   reported as groups (merged ids together, conflict branches with the
   chosen one marked); what remains proposed/open; the derived next step
   (`tanuki-ledger next`). The sitting is complete only if the operator
   never had to type a `tanuki-ledger` or `gh` command themselves.
