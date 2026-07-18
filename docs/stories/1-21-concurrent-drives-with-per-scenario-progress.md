---
story: 1.21
epic: 1
title: "Story 1.21: tanuki-drive runs planned scenarios concurrently under a drive_concurrency cap, with per-scenario progress records"
status: published
issue: 148
umbrella: 138
---

# Story 1.21: tanuki-drive runs planned scenarios concurrently under a drive_concurrency cap, with per-scenario progress records

Umbrella: https://github.com/tim-nish/tanuki/issues/138 (spec: specs/spec-tanuki-loop/SPEC.md "Concurrent drives within one iteration") — on conflict, the amended spec wins.

As a tanuki operator with a large scenario matrix,
I want a `drive_concurrency` loop-block key (default 1) that lets `tanuki-drive` run up to N planned scenarios in parallel, each against its own fixture clone, with the progress substrate recording per-scenario entries,
So that an iteration's drive phase stops being serial wall-clock while isolation, mining, and the iteration bracket stay exactly as specified.

**Acceptance Criteria:**

- Given `drive_concurrency: 1` or an absent key, when a drive runs, then behavior and progress output are byte-compatible with today (serial is the N=1 case, not a second format).
- Given `drive_concurrency: N>1`, when the planned set drives, then up to N scenarios run in parallel, one disposable clone each, and every scenario's result in the manifest is identical in shape to today's.
- Given concurrent drives, when progress is read mid-run, then per-scenario entries (stage, elapsed, KB, result) are present and atomically updated — no interleaved corruption.
- Given the mining step, when drives finish, then mining runs once over the combined transcripts after all drives complete (barrier), and ledger writes happen only there.
- Given a scenario failure under concurrency, when other drives are still running, then the failure is recorded per-scenario without aborting the siblings.
