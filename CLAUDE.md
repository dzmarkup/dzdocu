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

## LAYOUT mode (4x6 only)

The four areas of a 4x6 label — `ret`, `main`, `postal`, `country` — are
independent absolutely-positioned boxes built by `buildLabel46Box`. Geometry
lives in `state.labelLayouts[size]` in **inches from the label's top-left**,
merged over `LABEL46_DEFAULT_LAYOUT` by `getLabelLayout()`, and persisted with
the document. Anything a saved file lacks falls back to the defaults, so
documents written before LAYOUT still open.

Things worth knowing before touching it:

- **A box's coordinates are label coordinates.** Absolute positioning resolves
  against the editable's padding box, whose origin is the label corner — do
  *not* subtract the page padding, or every box lands up and to the left.
- **`fitLabelBox` measures the content wrapper, not the box.** A flex box
  aligned middle/bottom over-reports its own `scrollHeight` (a box 92px tall
  holding 75px of content reports 96), which silently shrinks type that
  already fits.
- The panel and the box outlines are screen-only, and editing is switched off
  while the panel is open so a caret can't fight the controls.
- Only 4x6 uses the box model; 2x4 and 1x2.625 still use their own flow
  layout, which is why the tab is gated to `isLabel46Doc`.

`LABEL46_DEFAULT_LAYOUT` is measured from the layout the label shipped with,
so an untouched label renders identically to before LAYOUT existed. If you
change it, re-check against those numbers rather than by eye.

## The 4x6 label address block

Type scale is a constant above the class — retune there, do not unpick the
markup. `LABEL46_BODY_LEFT_IN` / `_RIGHT_IN` / `_BOTTOM_IN` are now only the
seed values behind the default box geometry:

- `LABEL46_BODY_LEFT_IN` / `LABEL46_BODY_RIGHT_IN` — where the Address box
  starts and ends. The province (`ON`) rides the box's right edge, so
  **widening that box is what moves the province right without disturbing the
  left alignment** — which is what the LAYOUT panel's Right nudge now does.
- `LABEL46_BODY_BOTTOM_IN` — the page's bottom inset, tighter than the generic
  label padding so the Country box can sit low.
- `LABEL46_BODY_BOOST` — type scale against the sizes the label shipped with.
  This one is still live for every box: it sets the base size each is fitted
  from.

Sizes inside a box are `em` against the box, so `fitLabelBox` rescales it by
setting one font-size. It steps down 3% at a time until the content stops
overflowing, with a floor at 35% of base.

**Fitting only runs from `setPageAddresses`** — when an address is entered,
re-rendered, or a box is nudged. Typing directly into a label does not re-fit
it; `handleInput` does not call it.

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
- **The ~97 lines of dead code previously noted here (marquee-zoom, the dead
  resume-prompt render props, `applyExecCommand` / `determineSizeOrientation`)
  were removed.** One loose end from that pass: `zoomActive`, `revertZoom()`,
  and the Escape/Ctrl+Z checks that reference them were left in place since
  they aren't exclusively part of the removed marquee-select button — but
  `applyZoom()` (deleted, it was the marquee-select drop handler) was the only
  code that ever set `zoomActive` to `true`, so that whole "Full Screen" zoom
  toggle is now unreachable too. Worth a follow-up pass if it's not wanted.
- **There is no way to clear a saved document from the UI.** The resume prompt
  that offered "Start New (deletes saved doc)" was removed from the markup but
  its handler remains. Restoring it is a real missing feature, not cleanup.
- **A reload does not restore the open document.** The save still happens, but
  nothing reads it back at boot — that was the resume prompt's job. Reopening
  means dropping the `.docu` in.

## Opening a saved .docu

Two routes, and they behave differently, which is worth knowing before
believing a bug report about either:

- **From inside an open document**, the "Open .docu" side tab works.
- **On the setup screen** the same tab is unreachable — that screen's overlay
  is `z-index:50` over the tab's `18`. The route the setup screen offers is
  the DROP-ZONE, so that is the one to test on a fresh visit.

`applySavedData` sets `docCreated` itself. It must: restoring *is* creating a
document, and without it the setup screen stays up, the editables never mount,
and the restored `pageHtml` has nowhere to go — silently, with no error.
