# DZDocu — working notes for Claude

This repo serves **dzdocu.com**: `CNAME`, `index.html`, and the `assets/` and
`fonts/` folders it loads from. `index.html` is the application. Work here and
nowhere else.

## There is an older copy of this app elsewhere — ignore it

The same app also sits at `bluejetty.ca/docu/index.html`, in the unrelated
`bluejetty/bluejetty` repo. That was the test bed before DOCU got its own
domain. **This repo is the live one.** If the two ever differ, this one wins;
do not try to reconcile them, and do not go looking for that repo.

## Standing preferences

- **Confirm the destination before pushing.** The user owns two GitHub accounts
  (`bluejetty` and `dzmarkup`) with several repos between them, so a change can
  look right and still land in the wrong place. Name the repo and branch and
  get a yes first.
- **Hand over a backup zip with the work**, not just loose files: the changed
  files at their real paths, and a `git bundle` of any unpushed commits.
- Push to a branch and let the user merge the PR. Do not push to `main`.

## The app is plain, hand-editable HTML

`index.html` is ~157 KB of ordinary indented markup. **Edit it directly.** It
was once a single ~870 KB bundle with the assets gzip+base64'd into a JSON
manifest and the document as one JSON-escaped string; that is gone, along with
the `json.loads` / re-serialise / `</` escaping ritual it needed. If you
find notes describing that, they are out of date.

Layout:

- `assets/*.js` and `fonts/*.woff2` load through ordinary `<script src>` and
  `@font-face` — React and react-dom among them, as real script tags. Nothing
  is fetched from `unpkg.com` any more.
- The application logic is one `<script type="text/x-dc">` block near the end,
  a single `class Component extends DCLogic`. dc-runtime (`assets/support.js`)
  evaluates it.
- The `{{ … }}` bindings in the markup are filled by `renderVals()`.

Tuning constants live at the top of that script block, above the class.

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

## The 4x6 label address block

Sizing and placement are four constants above the class — retune there, do not
unpick the markup:

- `LABEL46_BODY_LEFT_IN` / `LABEL46_BODY_RIGHT_IN` — the box the delivery
  address lives in, in inches from the label's left edge. The province (`ON`)
  is flush against the right edge, so **widening the box is what moves the
  province right without disturbing the left alignment.**
- `LABEL46_BODY_BOTTOM_IN` — bottom inset, deliberately tighter than the
  label's own padding. Postal + country are pinned to the floor of the box with
  `margin-top:auto`; everything above them gets the height that frees up.
- `LABEL46_BODY_BOOST` — type scale against the sizes the label shipped with.

Every size inside the block is in `em` against the wrapper, so `fitLabelBody`
rescales the whole thing by setting one font-size. It runs after the markup is
in the DOM (it needs real measurements) and steps down 3% at a time until the
content stops overflowing, with a floor at 45% of base.

**`fitLabelBody` only runs from `setPageAddresses`** — when an address is
entered or re-rendered. Typing directly into a label does not re-fit it;
`handleInput` does not call it.

## Known and unfinished

- **~85 KB of the bundled fonts never render.** The design-system stylesheet
  sets Barlow on body/buttons/inputs, then the document overrides it all with
  `font-family:'Arial Narrow', Arial, sans-serif !important`. Measured live: of
  the 6 woff2 files the browser fetches exactly one,
  `barlow-condensed-600-latin.woff2` (22 KB), for the "Aa" tab. The plain
  Barlow faces are already gone; the leftovers are the 400 weight and the
  latin-ext/vietnamese subsets. Dropping them needs a before/after screenshot
  diff across the setup screen, font picker and document first.
- **`Arial Narrow` is not installed on Linux**, so the labels fall back to
  Arial there — wider than on a machine that has it. Screenshots taken on this
  box match what a Linux user sees, not a Windows/Mac one.
- **pdf.js (313 KB) + mammoth (626 KB) load on every visit** but are only used
  when a PDF or DOCX is dropped. Together they dwarf everything else in
  `assets/`. Lazy-loading them is the largest single win available.
- **~97 lines of dead code**: the marquee-zoom subsystem (59) whose button was
  removed from the markup, so `selecting` can never become true; the resume
  prompt (21); and `applyExecCommand` / `determineSizeOrientation` (17), never
  called.
- **There is no way to clear a saved document from the UI.** The resume prompt
  that offered "Start New (deletes saved doc)" was removed from the markup but
  its handler remains. Restoring it is a real missing feature, not cleanup.
