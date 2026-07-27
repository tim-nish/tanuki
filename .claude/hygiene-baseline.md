# Hygiene baseline — tim-nish/tanuki

Written by `/hygiene-sweep`. One row per measured file/read-set; incremental
sweeps diff against these numbers. Dispositions are durable: a `rejected`
finding never re-surfaces.

## Sweep log

| Date | Scope | Findings | Filed | Rejected |
|------|-------|----------|-------|----------|
| 2026-07-27 | full (first run) | 10 | #339 #340 #341 #342 #343 #344 #345 #346 #347 #348 | — |

## Dispositions

| Finding | Class | Disposition |
|---------|-------|-------------|
| Retired merge-first gate steps still printed in commands/tanuki-loop.md | redundancy/contradiction | filed #339 |
| Config-key surface ×5, divergent (reach_min_turns, driven_env_passthrough, drive_model/driver_model, drive_concurrency) | redundancy/drifted | filed #340 |
| Spec↔command drift: four-part vs five-part invariant; queue-discharge paragraph ×2 | redundancy/drifted | filed #341 |
| Cross-tool helper duplication, already diverged (slug_target, open_scenarios, load_config, host-binding, priority) | redundancy/code | filed #342 |
| ~200+ lines of spec text restated in command files | redundancy/structural | filed #343 |
| commands/* exceed ~10k tokens, no declared cap (cap proposal) | bloat/proposal | filed #344 |
| docs/tanuki-spec.md §398–500 paraphrases per-topic specs; file layout ×2 | redundancy/superseded | filed #345 |
| spec-doc-policy-alignment: completed audit report in specs/ | mis-filed archive | filed #346 |
| tools/tanuki-loop single flat 6,602-line module (structure-decision proposal) | bloat/code | filed #347 |
| 1 MB logo PNG; manifest description strings ×3 differ | bloat/asset + drift | filed #348 |

Noted, not findings (suppressed by design): `docs/stories/` (~50k tok) is
fenced non-normative archive off every hot path; the loop command's fork from
the attended commands is a documented owner decision (2026-07-13);
`docs/README.md` and `docs/journey.md` are clean pointer surfaces.

## Measurements — 2026-07-27

Approx tokens = chars/4. Tracked files: 196.

| File / read-set | Lines | ~Tokens |
|-----------------|------:|--------:|
| tools/tanuki-loop | 6,602 | 90,051 |
| tools/tanuki-ledger | 2,659 | 35,039 |
| tools/tanuki-drive | 1,544 | 18,895 |
| tools/tanuki-scheduler | 1,479 | 18,507 |
| tools/tanuki-config | 594 | 6,835 |
| specs/spec-tanuki-loop/SPEC.md | 1,037 | 16,414 |
| commands/tanuki-loop.md | 762 | 12,612 |
| commands/tanuki.md | 798 | 12,405 |
| specs/spec-tanuki-scenario-lifecycle/SPEC.md | 627 | 9,459 |
| docs/tanuki-spec.md | 514 | 8,251 |
| README.md | 485 | 6,508 |
| specs/spec-tanuki-view/SPEC.md | 370 | 5,462 |
| CHANGELOG.md | 299 | 4,578 |
| specs/spec-tanuki-trajectory/SPEC.md | 284 | 4,086 |
| specs/spec-short-command-surface/SPEC.md | 275 | 4,034 |
| specs/spec-tanuki-solve/SPEC.md | 194 | 2,931 |
| specs/spec-host-snapshot/SPEC.md | 181 | 2,807 |
| specs/spec-den-distill/SPEC.md | 171 | 2,517 |
| specs/spec-policy-advisory/SPEC.md | 161 | 2,351 |
| specs/spec-doc-policy-alignment/SPEC.md | 146 | 2,054 |
| docs/stories/* (92 files, archive) | 4,065 | 49,778 |
| docs/assets/tanuki.png (binary) | — | 1,000 KB |

Read-sets (per-sitting context load; both command files are self-contained —
they cite specs by section but instruct no upfront spec reads):

| Read-set | ~Tokens |
|----------|--------:|
| /tanuki invocation (commands/tanuki.md) | 12,405 |
| /tanuki-loop invocation (commands/tanuki-loop.md) | 12,612 |

No repo-declared budget exists for either surface (see #344).
