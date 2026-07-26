---
story: 2.1
epic: 2
title: "Story 2.1: driven session escaped its charter sandbox and created a real GitHub repo — gh credentials are fully inherited under --dangerously-skip-permissions"
status: published
issue: 300
umbrella: 300
---

# Story 2.1: A driven session inherits no ambient authority

Canonical discussion record: [#300](https://github.com/tim-nish/tanuki/issues/300)
(with [#308](https://github.com/tim-nish/tanuki/issues/308) as the ledger-side
duplicate of the same incident). On conflict, the issue wins.

## Story

As **the operator of a dogfooding run**,
I want **a driven scenario to be unable to authenticate as me to any service**,
so that **a charter that brushes against real-service behaviour cannot create,
mutate, or orphan anything on my accounts — rather than relying on the driver's
own cleanup, which has already been observed to fail.**

## Context

`tanuki-drive` launches each scenario with `--dangerously-skip-permissions`
(`tools/tanuki-drive:1003`) and no environment restriction: the launch is
`subprocess.Popen(cmd, cwd=host_clone, stdout=out, stderr=err, text=True,
stdin=subprocess.DEVNULL)` (`tools/tanuki-drive:1013`), which passes no `env=`
and therefore hands the child the parent's entire environment — `GH_TOKEN`,
`GH_CONFIG_DIR`, `SSH_AUTH_SOCK` and every other credential in scope. The
disposable-clone isolation is filesystem-level only.

On 2026-07-26 a scenario needing GitHub's real "Base ref must be a branch"
rejection ran `gh repo create` on the owner's account; its own cleanup 41
seconds later failed with HTTP 403 because the token lacks `delete_repo`, and
the private repo was orphaned with no report anywhere.

The governing invariant was amended in the same sitting as this story
(`docs/tanuki-spec.md`, isolation invariant and the prototype-deviations entry):
isolation now means clones **and** a scrubbed environment, and a scenario
needing a credentialed service's behaviour gets a fixture or an explicitly
provisioned scoped credential.

## Acceptance criteria

**AC1 — the launch passes an explicit, scrubbed environment.**
Given `tanuki-drive` starts a scenario,
When the driven `claude` process is launched at `tools/tanuki-drive:1013`,
Then the call passes an explicit `env=` built from an allowlist rather than
inheriting `os.environ`, with at minimum `GH_TOKEN`, `GITHUB_TOKEN`, and
`SSH_AUTH_SOCK` absent and `GH_CONFIG_DIR` pointed at an empty directory inside
the run's workspace.

**AC2 — a driven session cannot authenticate to the forge.**
Given a scenario is running under AC1's environment,
When it invokes `gh` against any remote,
Then the call fails as unauthenticated, and the failure is visible in the
scenario's captured stream rather than silently succeeding against the
operator's account.

**AC3 — a fixture path exists for scenarios that need forge behaviour.**
Given a charter must exercise a real forge response (e.g. a PR-base rejection),
When the scenario runs,
Then it can obtain that behaviour from a `gh` shim placed on `PATH`, following
the pattern already shipped in this repo's own suite — `tools/tests/test-loop-gate-pr:36`
writes a scripted fake `gh` into `$root/bin/gh`, and `:98`/`:213` drive the
no-`gh` case by overriding `PATH` — and the shim is documented where charter
authors will find it.

**AC4 — the deliverables land together.**
Given AC1 removes the credentials the `gate-pr-base-origin-rejection` scenario
currently relies on,
When this story's PR merges,
Then AC3's fixture is available in the same change, so no interval exists in
which that scenario is broken.

**AC5 — an explicit opt-in remains possible.**
Given an operator deliberately wants a scenario to reach a real service,
When they provision a scoped credential for it,
Then a documented, explicit mechanism admits it into the driven environment —
the default is scrubbed, and reaching a real service is never implicit.

## Story questions (unverified — not acceptance criteria)

- Which environment variables beyond the three named in AC1 must be scrubbed?
  The full credential surface of a `claude` child process was **not** enumerated
  this sitting (cannot-determine — not read), so AC1 states a floor rather than
  a complete list. Enumerating it is part of this story's implementation.
- Does anything else in the pipeline depend on inherited environment beyond
  `TANUKI_ROOT` (`tools/tanuki-drive:78`)? Unverified — the allowlist must be
  derived by reading the driven session's actual needs, not assumed.

## Out of scope

- Container isolation and network-egress filtering. The amended spec keeps both
  as explicit, unaddressed prototype deviations; this story closes the
  ambient-authority half only.
- Re-authoring the `gate-pr-base-origin-rejection` charter itself. The scenario
  matrix lives in `~/.tanuki/scenarios/`, outside this repo, and Tanuki never
  writes configuration into a target repository — AC3 delivers the fixture the
  re-authoring will use, and the charter edit is operator-side work.
