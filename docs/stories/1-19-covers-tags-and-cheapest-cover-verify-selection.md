---
story: 1.19
epic: 1
title: "Story 1.19: covers: tags on scenarios, surface tags at mining, and cheapest-cover verify selection in plan"
status: published
issue: 146
umbrella: 137
---

# Story 1.19: covers: tags on scenarios, surface tags at mining, and cheapest-cover verify selection in plan

Umbrella: https://github.com/tim-nish/tanuki/issues/137 (spec: specs/spec-tanuki-scenario-lifecycle/SPEC.md "Cheapest covering verify") — on conflict, the amended spec wins.

As a tanuki operator with expensive end-to-end scenarios,
I want scenarios to declare `covers:` surface tags, mining to record a finding's surface tag, and `tanuki-scheduler plan` to fill each verify slot with the cheapest covering scenario (originating fallback; per-finding pin honored),
So that verify cost tracks what reproduces a finding, not where it happened to first appear.

**Acceptance Criteria:**

- Given scenarios with `covers:` tags and a finding carrying a surface tag, when `plan` builds the verify set, then the entry is the cheapest covering scenario (explicit cost key, else `max_turns`), and the plan record shows which finding each verify entry verifies and why that scenario was chosen.
- Given a finding with no surface tag or no cheaper cover, when `plan` runs, then the originating scenario is selected exactly as today.
- Given a finding pinned to its originating scenario, when `plan` runs, then the pin wins over any cheaper cover.
- Given the reserved exploration quota and LRU rotation, when covers-based selection is active, then slot counts and rotation order are unchanged — only the scenario filling a verify slot differs.
- Given `sync`, when a matrix declares `covers:` tags, then they are accepted (target-local vocabulary, no normalization) and visible in `status`/`history` output.
