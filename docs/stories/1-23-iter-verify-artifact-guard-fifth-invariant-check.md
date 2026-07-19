---
story: 1.23
epic: 1
title: "Story 1.23: iter-verify build-artifact guard as fifth integration-invariant check"
status: published
issue: 159
umbrella: 159
---

# Story 1.23: iter-verify build-artifact guard as fifth integration-invariant check

Umbrella: https://github.com/tim-nish/tanuki/issues/159 (spec: specs/spec-tanuki-loop/SPEC.md "One iteration", integration invariant AMENDED 2026-07-19) — on conflict, the amended spec wins.

As a tanuki-loop operator whose unattended run commits with a hand-run `git add -A`,
I want iter-verify to refuse the iteration when its commits introduce build-artifact paths (check (e) of the integration invariant), using the same pattern source as doctor's `test-cmd-artifacts` check,
So that `__pycache__`/`*.pyc` cruft can never ride through merge and gate-push onto origin even when Phase 1 skipped doctor.

**Acceptance Criteria:**

- Given an iteration whose commits add a path matching the artifact pattern list (e.g. `__pycache__/`, `*.pyc`), when `iter-verify` runs, then it fails as an immediate-stop breaker naming the offending path(s), same taxonomy/rendering as the existing four checks.
- Given an iteration whose commits introduce no artifact paths, when `iter-verify` runs, then behavior is unchanged from today.
- Given a run that must commit a legitimate binary, when the operator supplies the explicit per-run override, then check (e) passes for the declared paths only and the override is recorded in the run's audit trail — the guard never stands down silently.
- Given doctor and iter-verify, when either evaluates artifact patterns, then both read one shared pattern source (no second list to drift).
- Given the dashboard renders a run stopped by check (e), when the breaker is shown, then its NEXT guidance names the remedy (drop the artifact commit or declare the override), consistent with sibling breakers.
