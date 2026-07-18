---
story: 1.20
epic: 1
title: "Story 1.20: covering-absence compaction and the under-reproduction re-pin guard"
status: published
depends_on: ["1.19"]
issue: 147
umbrella: 137
---

# Story 1.20: covering-absence compaction and the under-reproduction re-pin guard

Umbrella: https://github.com/tim-nish/tanuki/issues/137 (spec: specs/spec-tanuki-scenario-lifecycle/SPEC.md "Cheapest covering verify") — on conflict, the amended spec wins.

As a tanuki operator,
I want driven-absence compaction to count runs where any covering scenario was driven (finding absent), and a recurrence before tombstoning to reset absence AND re-pin the finding's verify entry to its originating scenario,
So that cheap covers can verify fixes without ever manufacturing a false "fixed" tombstone from under-reproduction.

**Acceptance Criteria:**

- Given an accepted finding with a surface tag, when a run drives a covering scenario and the finding does not recur, then that run counts toward the finding's driven-absence.
- Given a finding accruing cover-driven absence, when it recurs anywhere before tombstoning, then the absence count resets (existing rule) and the finding's verify entry is re-pinned to its originating scenario — visible in ledger/plan output, arithmetic only.
- Given a finding with no `covers` match, when compaction evaluates it, then behavior is exactly today's originating-scenario driven-absence.
- Given fixtures pinning existing behavior, when the suite runs, then the pre-covers compaction fixtures still pass unchanged (legacy path intact).
