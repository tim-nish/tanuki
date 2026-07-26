# contribute-back intake templates

`contribute-back-intake.example.md` is a working example of the
`contribute_back.schema` file that `tanuki-ledger distill --id` renders when an
accepted lesson candidate is emitted into your knowledge hub's staging
directory.

## Why the documentation lives here and not in the template

Rendering is a **mechanical whole-file substitution**: every occurrence of a
token is replaced wherever it appears, including inside HTML comments. So a
template that documents its own tokens rewrites that documentation at render
time — a copy carrying the line ``the {slug} token`` emits
``the tanuki-my-target-f12 token``, silently, and only a reader who opens the
staged file notices.

Keeping the token table in this file — which nothing ever renders — makes that
failure structurally impossible rather than a warning someone must obey.

## Using it

Copy the example anywhere you like, edit the prose to match your hub's intake
conventions, and point the target's `contribute_back` block at your copy:

```json
"defaults": {
  "contribute_back": {
    "path":   "/abs/path/to/your-hub/staging",
    "schema": "/abs/path/to/your-copy.md"
  }
}
```

That block goes under `"defaults"` in the target's scenarios file,
`~/.tanuki/scenarios/<target>.scenarios.json`. It is read per-target only — the
machine-wide `~/.tanuki/config.json` is not consulted. Validate it with
`tanuki-ledger --target <t> distill --check` before anything consumes it.

## Tokens

Every token below is substituted wherever it appears; everything else is copied
verbatim. No token is required — an unused one is simply never substituted.

| token | meaning |
|---|---|
| `{slug}` | staging slug, sanitized to `[a-z0-9-]`; defaults to `tanuki-<target>-<finding id>` |
| `{created}` | emission date |
| `{source_repo}` | the target the finding came from |
| `{title}` | the finding's title |
| `{kind}` | `friction`, `papercut`, or `gap` |
| `{tags}` | comma-separated tags derived from the finding |
| `{perishable}` | whether the lesson is time-bound |
| `{recurrence}` | the finding's recurrence count at emission |
| `{provenance}` | finding id, kind, recurrence, scenarios, evidence pointers |
| `{body}` | the rendered lesson body |

## What emission does and does not do

The write lands inside the configured staging directory or not at all, and an
existing file is never clobbered. Emission records a `contributed` marker on the
finding, so re-runs can bump recurrence but never emit a duplicate. Nothing here
is authoritative for the receiving hub: a staged file is a proposal awaiting
that hub's own gate.
