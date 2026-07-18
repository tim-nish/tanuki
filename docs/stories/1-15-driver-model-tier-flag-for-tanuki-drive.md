---
story: 1.15
epic: 1
title: "Story 1.15: tanuki-drive gains a driver model/effort tier setting (target default + CLI override), recorded in the manifest"
status: published
issue: 139
---

# Story 1.15: tanuki-drive gains a driver model/effort tier setting (target default + CLI override), recorded in the manifest

Canonical discussion: https://github.com/tim-nish/tanuki/issues/139 — on conflict, the issue wins.

As a tanuki operator running long loop iterations,
I want the simulated-user driver's model tier settable per target (scenarios `"loop"` block, e.g. `"drive_model"`) with a `tanuki-drive` CLI override, applying only to the driver side,
So that driver cost/latency stops scaling with the strongest session tier while the plugin under test is untouched.

**Acceptance Criteria:**

- Given a scenarios file whose `loop` block sets `"drive_model"`, when `tanuki-drive` runs without a model flag, then the simulated-user driver runs on that tier and the target plugin's session is unaffected.
- Given a `tanuki-drive` CLI model override, when both the flag and the `loop` block are present, then the flag wins (config < CLI, the existing precedence rule).
- Given any drive, when the manifest is written, then it records the effective driver model per scenario, so findings can later be joined against driver tier when judging chronic-vs-flaky.
- Given no `drive_model` and no flag, when a drive runs, then behavior is unchanged from today (per-scenario `"model"` keys still work and still win for their scenario).
- Given the default derivation, when `drive_model` is unset, then the default stays a cheap tier per the routing table — never the frontier session model; the issue's caveat (default one tier below the session model, not the floor) is honored or explicitly revisited in the PR.

Scope note from the issue: a driver too weak to follow multi-step skill instructions produces spurious friction findings; the promotion bar filters noise by design, but the chosen default must not be the floor tier.
