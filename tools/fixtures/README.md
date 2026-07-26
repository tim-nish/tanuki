# Fixtures for driven scenarios

A driven session gets a disposable filesystem **and a scrubbed environment**
(`docs/tanuki-spec.md`, the isolation invariant): it cannot authenticate as the
operator to any service. `tanuki-drive` builds the child's environment from an
allowlist (`DRIVEN_ENV_ALLOW`), points `GH_CONFIG_DIR` at an empty directory so
the `HOME`-based `gh` lookup finds nothing, and tells git never to prompt or
reach for an agent key.

That closes the hole behind issue #300 — a scenario created a real repository on
the operator's account and could not delete it — and it means a charter that
needs a credentialed service's *behaviour* must get it from a fixture here.

## `gh-shim/`

A scripted stand-in for `gh`. Put it on `PATH` and script one file per
subcommand:

```bash
export PATH="<plugin-clone>/tools/fixtures/gh-shim:$PATH"
export GH_SHIM_DIR="$PWD/.gh-shim"
mkdir -p "$GH_SHIM_DIR"

# make `gh pr create` fail the way GitHub does for a remote-tracking base
printf 'Base ref must be a branch\n' > "$GH_SHIM_DIR/pr-create.err"
printf '3\n'                         > "$GH_SHIM_DIR/pr-create.exit"
```

The key is the first two arguments joined by `-` (`gh pr create …` →
`pr-create`), or just the first when the second is a flag. Per key:

| file | effect | default |
|---|---|---|
| `<key>.out` | printed to stdout | empty |
| `<key>.err` | printed to stderr | an unscripted-call message |
| `<key>.exit` | exit code | `1` |

Every invocation is appended to `$GH_SHIM_DIR/calls.log`, so a scenario can
assert on what was attempted — including that something was attempted at all.

**Unscripted calls fail loudly.** A shim that invents a success is worse than no
shim: the scenario would report a forge behaviour that never happened, and the
finding derived from it would be fiction.

## The deliberate opt-in

When a scenario must reach a *real* service, that is a decision, not a default.
Name the variables in the target's scenarios file:

```json
"defaults": { "driven_env_passthrough": ["GH_TOKEN"] }
```

`tanuki-drive` applies these last, after the forced settings, and prints one
line per scenario naming what was admitted. Prefer a scoped, disposable
credential over the operator's own — and prefer the shim over either.
