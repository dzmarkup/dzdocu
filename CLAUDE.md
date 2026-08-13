# DZDocu — working notes for Claude

This repo serves **dzdocu.com**: `CNAME`, `index.html`, and the `assets/`
folder it loads from. `index.html` is the application. Work here and nowhere
else.

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

- `assets/*.js` loads through ordinary `<script src>` tags — React and
  react-dom among them. Nothing is fetched from `unpkg.com` any more. There's
  no `fonts/` folder or `@font-face` any more either — see "Fonts" below.
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

## Custom properties don't fall back the way ordinary properties do

The classic progressive-enhancement trick —

```css
--setup-panel-bg: #182738;
--setup-panel-bg: color-mix(in srgb, var(--color-accent) 18%, #0a1420 82%);
```

— redeclare a token, trusting an old browser to reject the second line and
silently keep the first — **does not work for custom (`--x`) properties.**
Ordinary properties are validated when declared, so an old browser really
does drop an unparseable second line and keep the first. Custom properties
are not: their grammar accepts almost any token stream, so the `color-mix()`
line always wins the cascade regardless of whether the browser understands
`color-mix()` — it only breaks once something actually consumes the token via
`var()`, at which point *that* property (not the custom property) fails and
computes to nothing. This silently broke the setup screen's dark panels
(transparent background/border/glow) on browsers old enough to lack
`color-mix()` — a real bug, filed and fixed as a Windows 7 report. The fix:
gate the `color-mix()` versions behind a real feature query,

```css
@supports (color: color-mix(in srgb, red 50%, blue)) {
  :root { --setup-panel-bg: color-mix(in srgb, var(--color-accent) 18%, #0a1420 82%); }
}
```

leaving the hex line as the sole, unconditional `:root` declaration. If more
`color-mix()` tokens get added later, they need the same `@supports` gate —
the redeclare trick will look correct on every browser this box can test on
and still be silently broken on anything old.

## Fonts

The document's font stack (`.editable`, box content, most UI chrome) is
`Arial, sans-serif` — a deliberate choice, not an oversight. There's no
`fonts/` folder any more; nothing is bundled. Two things are worth knowing
before "fixing" this again:

- **This went through a Barlow Condensed detour and back.** Arial Narrow was
  the original default; it isn't installed on Linux, so the browser fell
  through to a regular-width Arial substitute there, throwing off label
  auto-fit and box-overflow math relative to Windows/Mac. That got fixed by
  bundling Barlow Condensed as a guaranteed-consistent webfont — but the
  user preferred the plainer, wider look of ordinary Arial over either
  condensed option, so the stack was simplified to plain `Arial` instead of
  reintroducing Arial Narrow. If bundling a webfont ever comes back up,
  check git history for the Barlow Condensed commits rather than redoing
  that research from scratch.
- **Plain `Arial` is a safer default than `Arial Narrow` was, even without a
  bundled fallback.** Linux doesn't ship real Arial either, but it does ship
  "Liberation Sans," a font expressly built to be metric-compatible with
  Arial (same character widths) — unlike the regular-width substitute Arial
  Narrow got, which was never designed to match a condensed face. So the
  cross-platform width mismatch that motivated the Barlow Condensed detour
  in the first place doesn't apply here.

The font picker still lists "Arial Narrow" (and everything else in
`SYSTEM_FONTS`/`POPULAR_FONTS`) as ordinary, literal choices — picking one
just applies that name directly (`'<name>', sans-serif`), no substitution.
Google-hosted picks (`POPULAR_FONTS`) are fetched live from Google Fonts on
first use, independent of anything bundled locally.

## A dropped file on the entry screen is queued, not imported

`handleFile()` deliberately holds a file in `pendingDropFiles` when
`docCreated` is false — importing before a page size is picked used to flow
the content at whatever size happened to be current. The catch: for a long
time nothing on screen changed when this happened, because
`pendingDropFiles` is an instance field, not state. A drop that worked
perfectly looked identical to one that did nothing, and got reported as
"drag and drop is broken" more than once. `queuedDropNames` mirrors the
queued filenames into state purely so the entry screen can show a
"Ready to import" confirmation; `importPendingDrops()` clears it.

If you ever chase a "drop doesn't work" report again: verify with a **real**
drag (CDP `Input.dispatchDragEvent`, which does genuine hit-testing), not a
synthetic `new DragEvent('drop')` dispatched at an element. The synthetic
kind bypasses hit-testing entirely and will happily pass while real drags
fail — that exact blind spot hid this for a while.

## PDF import: pages first, text only if asked

A dropped PDF comes in as **rendered page images** (`renderPdfAsPages`), so
it looks like the original. It used to prefer text extraction whenever the
PDF had a text layer, which silently discarded the document's actual
appearance. Extraction is still there but is now an offer, not the default:
`pdfHasTextLayer()` probes the first few pages, and only if there's real
text does a banner appear offering "Extract Text"
(`onExtractPdfText` → `extractPdfTextInto`). Taking that offer collapses the
document back to one page and re-imports as flowing text, so it needs the
original `File` kept around (`_pdfExtractFile`) — re-read it rather than
reusing the first `ArrayBuffer`, since pdf.js takes ownership of the buffer
it's handed and can detach it.

**pdf.js's worker still comes from cdnjs.cloudflare.com** (see the
`workerSrc` line in `onDropPDF`). That means PDF import needs the network
even though everything else in the app is local — worth bundling the worker
alongside `assets/pdf.min.js` if offline use ever matters.

## PDF/DOCX import is lazy-loaded

`pdf.min.js` and `mammoth.browser.js` (~940 KB together) are not `<script
src>` tags — they're fetched on first use via `loadScriptOnce()`, called from
`ensurePdfJs()`/`ensureMammoth()` at the top of `onDropPDF`/`onDropDocx`. Every
drop entry point (a page, the DROP-ZONE button, a pasted PDF URL) already
funnels through those two methods, so nothing else needs to know about the
lazy-load. Don't add a static `<script src="assets/pdf.min.js">` back for
convenience — that's the exact ~940 KB this was written to avoid paying on
every visit.

## D-Z vs the entry screen's DROP-ZONE button — two different jobs

These look like the same button (D-Z is the in-document top-bar tab,
DROP-ZONE is its entry-screen counterpart) but do deliberately different
things on click, so don't merge their handlers again:

- **D-Z** (`onDropButtonClick`) only steps back to the entry screen
  (`docCreated: false, setupStep: 'size'`) — nothing else. No file picker,
  no modal. It used to also try to open a URL/Browse chooser modal; that
  modal is gone now (see below), and even while it existed this was the
  wrong button to trigger it from — a user clicking D-Z from inside a
  document wants to get back to the entry screen, full stop, not have a
  file dialog shoved in front of them immediately.
- **DROP-ZONE** (`onEntryDropZoneClick`) is the entry screen's own button,
  and goes straight to the OS file picker (`fileInputEl.click()`) — no
  modal, no intermediate choice. Drag-and-drop and paste are the other two
  ways to bring a file in; a plain click is the third and simplest.

There used to be a `dropStep`/`dropChooserOpen` modal offering "Enter a
URL" or "Browse" as a choice, reachable by neither button (a bug — see git
history around the `claude-design-ai-debug-szlzab` branch if curious). Once
the click-to-open bug was noticed and about to be fixed, the URL option
turned out to be unwanted entirely, so the whole modal/state
(`dropStep`, `dropResumeDoc`, `closeDropChooser`, `onDropChooseUrl`,
`onDropChooseFile`) was removed rather than kept around half-used.
`handleUrl()` itself is still very much alive — dragging a URL (not a file)
onto the entry screen or a page still works, that's a separate path.

Drop/paste coverage on the entry screen itself is intentionally wider than
just the button: `onEntryScreenDragOver`/`Drop` are wired on the screen's
outer overlay (not the gradient background div — that one closes before the
logo/button panel even opens in the DOM, so it doesn't cover them), and a
document-level `paste` listener (`_onEntryScreenPaste`, added in
`componentDidMount`) catches pasted files anywhere. Both are gated so they
only act while `setupStep === 'size'` / `docCreated` is false — this is what
stops them from hijacking the orientation/label-address steps' own text
fields, which share the same overlay and background.

## Known and unfinished

Nothing outstanding right now. The prior items here (unused font subsets,
Arial Narrow's Linux fallback, unconditional pdf.js/mammoth loading, a
resume/auto-restore gap, and some leftover dead zoom-toggle code) were all
resolved — see the sections above for the font/lazy-load/color-mix notes,
and "Opening a saved .docu" below for how restore now works.

## Opening a saved .docu

Three routes now, and they behave differently, which is worth knowing before
believing a bug report about any of them:

- **A reload while a document is open restores it automatically**, with no
  setup screen shown at all. `buildSaveData()` tags every save with
  `docWasOpen` (true only while `docCreated` is true — not while sitting on
  the setup screen) and `savedAt`. The constructor's `findAutoResumeData()`
  checks the three size buckets for the most recent `docWasOpen` save
  *before the first render*, so there's no flash of the setup screen before
  snapping over — only the page/box HTML pour-in (`pourSavedContent()`,
  shared with the two routes below) waits for real DOM refs in
  `componentDidMount`. A dismissible banner ("Resumed your last document." /
  Start New / ×) is the only way back to a clean setup screen once this
  fires — `onResumeStartNew()` clears the bucket and resets `docCreated`.
- **From inside an open document**, the "Open .docu" side tab works.
- **On the setup screen** the same tab is unreachable — that screen's overlay
  is `z-index:50` over the tab's `18`. The setup screen offers its own two
  routes instead: the DROP-ZONE, and (if a save exists but isn't flagged
  `docWasOpen`, e.g. cleared via Start New from a previous session) the red
  "In browser memory" list, which still opens a chosen doc via
  `applySavedData` directly.

`applySavedData` sets `docCreated` itself. It must: restoring *is* creating a
document, and without it the setup screen stays up, the editables never mount,
and the restored `pageHtml` has nowhere to go — silently, with no error.
