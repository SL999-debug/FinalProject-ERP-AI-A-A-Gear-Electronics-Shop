# AGENTS.md

Working notes for coding agents (and for me) working in this repository. Read this before
editing anything.

## What this repository is

Documentation, workflows and a schema **derived from** two self-contained HTML applications.
The HTML files are the source of truth. Where this repository and `app/` disagree, `app/` wins —
and this repository is the thing that needs fixing.

## Layout

```
app/            two standalone HTML files, no build step, no dependencies
workflows/      n8n exports, original filenames preserved
schema/         schema.json, extracted from app/aa-gear-admin.html
docs/           written guides
docs/screenshots/  59 screenshots, one folder per workflow
```

## Rules for editing `app/`

- **Do not introduce a build step, a bundler, a framework, or an npm dependency.** The two
  files must keep working when opened from `file://`. That constraint is the reason the UI
  looks the way it does.
- Both files are **Hebrew-first and RTL**. The root carries `dir="rtl"`. New UI text goes into
  the existing Hebrew strings; English is the exception, not the rule.
- Keep every string user-visible and in `lang`/`aria-label` consistent with the surrounding
  Hebrew.
- The dashboard is a **single file by design** — the `<style>` and `<script>` are inside
  `aa-gear-admin.html`. Do not split them out.
- No comments in the shipped code. If a behaviour is surprising, document it in `docs/`
  instead.

## Re-deriving `schema/schema.json`

`schema/schema.json` was generated from the dashboard's own `SCHEMAS` object, so it cannot
drift silently — but it is a snapshot, not a build step. After changing `SCHEMAS`, regenerate
and diff it. The extraction lives in the transcript history, not in this repo; keep it
out of `app/`.

Fields whose `airtableField` is `null` are **collected by the form and then thrown away**. This
is deliberate documentation of real bugs, not a formatting oversight — see
[docs/09-known-issues.md](docs/09-known-issues.md). Do not "fix" them by guessing Airtable
column names; confirm against the live base first.

## Rules for editing `workflows/`

- **Never re-import a workflow JSON from this repository into an n8n instance that holds
  credentials.** The exports carry credential id references, and re-importing re-links to
  whatever those ids resolve to. Import into a scratch instance, or strip the `credentials`
  blocks first. See [SECURITY.md](SECURITY.md).
- Renaming the files is acceptable and improves the docs; renaming the *nodes* is not part of
  this project's scope.
- If a workflow is fixed, update `docs/03-workflows.md` in the same change. The active flag
  alone is not a description.

## Screenshots

`docs/screenshots/` was generated from the original screenshot dump by
`ffmpeg -i in.jpg -vf scale='min(1600,iw)':-2 -q:v 4 out.jpg`. The originals are not in this
repository, only the resized copies, so:

- do not upscale, crop, or re-encode beyond the resize above;
- keep the `NN-short-description` naming — the docs link them positionally;
- if you add a doc that references a screenshot, add the file in the same commit.

## Testing the dashboard

There is no test suite, because the app talks to one webhook and there is no local n8n. When
you change the dashboard, verify by hand with a real n8n instance and cover at minimum:

- a **viewport below 420px**. The product list is a card layout below this breakpoint, and the
  stock badge has a fixed `51x29` geometry that must not move when a description is expanded.
- **all six tables** (`products`, `customers`, `leads`, `orders`, `invoices`, `tasks`) for
  search → sort → expand → edit → delete, since the search path re-renders a table's DOM and
  therefore detaches every listener. This was a real bug; see the history in
  [docs/09-known-issues.md](docs/09-known-issues.md).
- the console. A silent `ReferenceError` in a debounce or a re-render path is the failure mode
  that has bitten this file twice.

A full smoke-test pass also means checking that `renderCurrent()` is still called after a
successful `Store` mutation, otherwise a save looks like it worked and the row never moves.

## Style

Match the file you are in. The app is dense, uses short identifiers, and does not wrap at 80
characters. Documentation is the opposite: prose, wrapped, and explicit about uncertainty.
