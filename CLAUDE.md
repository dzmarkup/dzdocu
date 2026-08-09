# DZDocu — working notes for Claude

This repo serves **dzdocu.com**. It is two files: `CNAME` and `index.html`.
`index.html` is the whole application. Work here and nowhere else.

## There is an older copy of this app elsewhere — ignore it

The same bundled app also sits at `bluejetty.ca/docu/index.html`, in the
unrelated `bluejetty/bluejetty` repo. That was the test bed before DOCU got its
own domain. **This repo is the live one.** If the two ever differ, this one
wins; do not try to reconcile them, and do not go looking for that repo.

## Standing preferences

- **Confirm the destination before pushing.** The user owns two GitHub accounts
  (`bluejetty` and `dzmarkup`) with several repos between them, so a change can
  look right and still land in the wrong place. Name the repo and branch and
  get a yes first.
- **Hand over a backup zip with the work**, not just loose files: the changed
  files at their real paths, and a `git bundle` of any unpushed commits.
- Push to a branch and let the user merge the PR. Do not push to `main`.

## The app is one bundled file — not hand-editable as it stands

`index.html` is ~870 KB and self-contained. Nothing loads from disk beside it.
Structure:

- **Line 378** — JSON manifest of assets (gzip + base64): fonts, React, pdf.js,
  the docx reader, the `doc-page` web component.
- **Line 390** — the whole application document as one JSON-escaped string.

To change it: parse line 390 with `json.loads`, edit the extracted HTML, then
re-serialise. Escaping detail that matters — the original escapes `</` as
`</`, so apply `.replace('</', '<\\u002F')` after `json.dumps`, and assert
the round-trip before writing.

**Do not strip React or react-dom from the manifest.** They look unreferenced
because no `<script>` tag names them, but the dc-runtime fetches React from
`unpkg.com` at boot and the bundler serves the inlined copies in its place.
Removing them makes the app a blank page with no internet. This was tried.

## Verify print changes for real

Playwright is installed globally (`/opt/node22/lib/node_modules/playwright`),
Chromium lives at `/opt/pw-browsers`. Drive the app, then
`page.pdf({ preferCSSPageSize: true, printBackground: true })` and count
`/Type /Page` in the output. That is what catches an extra sheet.

**Delete the PDF before each run and let failures be loud.** A suite that
silences its scripts and reads last run's PDFs will report passes for an app
that no longer boots. Assert something rendered — e.g. that a visible button
exists — before trusting any page count.

## Label printing — the trap that caused the black page

Two screen-only affordances used to reach the printer and cost a whole extra
sheet:

- `.doc-stage-wrap` carries a 60px top gutter so the sheet clears the fixed
  toolbar. A label page is *exactly* one sheet tall, so 60px of offset pushed
  the whole page onto sheet 2.
- The `doc-page` host's desk colour (`#2b2b2b`) is set as an **inline** style —
  and re-applied in JS by `updateColsLayout()` — so it outranks the component's
  own `:host { background: none }` print rule. The component forces
  `print-color-adjust: exact`, so it prints even with "Background graphics"
  off. That is what filled the empty sheet 1 solid dark.

Both are reset in the document's print CSS. If a dark sheet ever comes back,
check those two rules survived a rebuild.

## Label addresses

Which address sits on which page is keyed by **page id** (`pageAddresses`), not
by position — "+ Page" can insert a repeated or blank label mid-run, and a
positional list makes the return-address toggle rewrite later pages with the
wrong address. The map is persisted with the document.

## Known and unfinished

- **~174 KB of Barlow fonts never render.** The design-system stylesheet sets
  Barlow on body/buttons/inputs, then the document overrides it all with
  `font-family:'Arial Narrow', Arial, sans-serif !important`. Measured live: of
  15 bundled woff2 files the browser loads exactly one, Barlow Condensed 600,
  for the "Aa" tab. Removing plain Barlow needs a before/after screenshot diff
  across the setup screen, font picker and document first.
- **pdf.js (116 KB) + mammoth (182 KB) load on every visit** but are only used
  when a PDF or DOCX is dropped. Lazy-loading them would roughly halve the
  initial download. Largest single win available.
- **~97 lines of dead code**: the marquee-zoom subsystem (59) whose button was
  removed from the markup, so `selecting` can never become true; the resume
  prompt (21); and `applyExecCommand` / `determineSizeOrientation` (17), never
  called.
- **There is no way to clear a saved document from the UI.** The resume prompt
  that offered "Start New (deletes saved doc)" was removed from the markup but
  its handler remains. Restoring it is a real missing feature, not cleanup.
