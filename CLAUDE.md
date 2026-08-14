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

- `.doc-stage-wrap` carries a 90px top gutter (60px on screens under 700px) so
  the sheet clears the fixed toolbar. A label page is *exactly* one sheet tall,
  so any of that offset pushed the whole page onto sheet 2. The gutter is
  deliberately larger than the bar needs — the bar ends at 47px, and the
  remaining ~33px is desk showing above the stage, so the page doesn't look
  welded to the bottom of the toolbar. Both values are **inline** styles, which
  is why the print reset and the narrow-screen override both need
  `!important`.
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

## A file dropped on the entry screen answers setup by itself

Dropping onto the entry screen no longer asks for a page size, orientation
or layout — `autoStartFromFile()` works out the answer from the file and
goes straight into the document. Only there: dropping into a document
that's already open keeps the old per-type behavior, since silently
re-sizing something being worked in would be destructive.

- **PDF** — the page size is read from the PDF itself
  (`detectSetupForFile` → `pdfSetupFromViewport`, shared with
  `renderPdfAsPages` so the page created and the page poured into can't
  disagree). Letter if it fits, otherwise tabloid; orientation follows the
  PDF. Bigger than 11×17 sets `pdfWasResized`, which adds a line to the
  notice rather than silently shrinking it.
- **Everything else** — letter portrait. Not a shortcut: TXT and RTF carry
  no page size at all, and the one inside DOCX's XML isn't exposed by
  mammoth, so there is genuinely nothing to read.

PDF pages are marked in `pdfPageIds` (persisted with the document), which
is what drops their margin box to zero and suppresses the header — they
print edge to edge at the PDF's own scale. The render is wrapped in a
`contentEditable="false"` holder with `pointer-events:none`: "locked"
means the artwork can't be dragged or resized out of scale, while the page
underneath still takes a caret and can be typed on. Extracting text clears
`pdfPageIds`, so margins and headers come back with the editable text.

## Verifying drag-and-drop

`pendingDropFiles`/`queuedDropNames` still exist and still show a
"Ready to import" confirmation, but the entry screen now imports straight
away (above), so that queue is only reached on paths that don't auto-start.

If you ever chase a "drop doesn't work" report again: verify with a **real**
drag (CDP `Input.dispatchDragEvent`, which does genuine hit-testing), not a
synthetic `new DragEvent('drop')` dispatched at an element. The synthetic
kind bypasses hit-testing entirely and will happily pass while real drags
fail — that exact blind spot hid this for a while.

## PDF import works but is deliberately not advertised

The DROP-ZONE button says `DOCU-DOCX-TXT-RTF` — no PDF, on purpose. PDF
import is fully built and working (everything in the next section); it just
isn't offered, because importing PDFs isn't what this app is for. Saving
*to* PDF via print is the PDF story users are meant to know about.

Dropping a PDF still imports it, and the file picker still accepts one, so
someone who tries it anyway gets something sensible instead of nothing.
**Don't "fix" the button label by adding PDF back**, and don't strip the
import code as dead — it's reachable, tested, and costs nothing when
unused (pdf.js is lazy-loaded).

## The top bar reflects the selection, it doesn't just command it

The Font/Size dropdowns show what the caret is actually sitting in, and
B/I/U glow blue when the selection already carries that formatting.
`readSelectionFormat()` reads it from the DOM (computed style at the
selection's *start* — the first character wins where a selection spans
several formats, which is the usual toolbar convention), and
`syncSelectionFormat()` writes it to state from a `selectionchange`
listener coalesced to one read per frame, only setting state when a value
really changed. `selectionchange` fires on every caret move and keystroke,
so neither of those guards is optional.

Two things that look like bugs but aren't:

- **A select can't display a value it has no `<option>` for.** The current
  font or an odd size (15px, say) is folded into the option list when it
  isn't already a preset, otherwise the box would just render blank.
- **Typing after formatted text keeps showing that formatting.** The caret
  really is inside the formatted run and the new text really does inherit
  it — the readout is correct, not stale.

## Walking the setup flow must build a genuinely new document

`onPickFullPageDocument` / `onPickDocLayoutTemplate` both go through
`freshDocumentState()`, which resets pages, content, box trees, PDF flags,
title and header. They used to only flip `docCreated: true`, which meant the
setup flow handed back whatever the *previous* document had left in state:
choosing a boxed layout, pressing D-Z, then choosing FULL PAGE the second
time still produced the first run's boxes, its extra pages, and its typed
text. Anything new that belongs to "a document" rather than "the app"
should be cleared there too.

Deliberately not applied to the reopen paths — `applySavedData` and the
constructor's auto-resume set their own state wholesale, and
`resumeIfSavedMatches` short-circuits at the orientation step, so a document
being reopened never reaches the layout choice.

## Inserting a page mid-document shifts everyone's content

Page content lives in the DOM, written imperatively (`el.innerHTML`), while
the page list renders positionally — so splicing a page into the middle of
`pageIds` hands each existing DOM node to the *next* page along, and the
last page's content falls off the end entirely. This is not theoretical: it
wiped page 2's render the first time `extractOnePdfPage` inserted a sheet
after page 1.

Anything that inserts a page anywhere but the end must snapshot every
page's `innerHTML` **by page id** before the `setState`, then pour it back
after the re-render — `extractOnePdfPage` shows the pattern, and it's the
same one `pourSavedContent()` uses. Appending to the end is safe and needs
none of this.

## Anything persisted needs adding to findAutoResumeData's list, too

`buildSaveData`/`applySavedData` and the constructor's auto-resume are
**two separate explicit field lists**. A new persisted field added to the
first two but not the third survives a save-and-reopen but silently
vanishes on a reload — which is exactly how resumed PDFs briefly lost their
full-bleed pages and their Extract Text buttons.

## PDF import: pages first, text only if asked

A dropped PDF comes in as **rendered page images** (`renderPdfAsPages`), so
it looks like the original — it used to prefer text extraction whenever the
PDF had a text layer, silently discarding the document's appearance.

Extraction is now **per page and non-destructive**. Each page's text is
pulled out at import and kept in `pdfPageText` (persisted), so the feature
still works after a reload, a save and reopen, or a browser restart —
deliberately not by holding the dropped `File`, which wouldn't survive any
of those. A page with real text shows its own screen-only **Extract Text**
button; pressing it inserts a normal letter sheet *directly after* that PDF
page (`extractOnePdfPage`), so it's obvious which text belongs to which
page, and clears that page's entry so the button disappears and the same
sheet can't be added twice. The banner's button does the same for every
remaining page. Nothing is ever destroyed — the PDF pages stay as they are.

Flat/scanned pages produce no text and so get no button at all, rather than
one that yields an empty sheet.

**PDFs over `MAX_PDF_PAGES` (40) are refused outright** with a plain "load a
smaller file" message, checked before the document is created so the entry
screen is simply still there. Every page becomes a full-sheet image inside
the saved document and the browser only gives localStorage a few MB; past
roughly that many pages the save silently starts failing, which is a much
worse experience than being told up front.

**The pdf.js worker is bundled** (`assets/pdf.worker.min.js`), not fetched
from a CDN, so PDF import works with no network at all like the rest of the
app. pdf.js refuses to run a worker whose version doesn't match the main
library exactly, so that file is the 3.11.174 worker from the same
`pdfjs-dist` build as `assets/pdf.min.js` (verified byte-identical against
the npm tarball). **If `pdf.min.js` is ever upgraded, replace the worker
from the matching build in the same commit** or PDF import dies with a
version-mismatch error. cdnjs is blocked by this box's proxy; npm
(`registry.npmjs.org/pdfjs-dist/-/pdfjs-dist-<version>.tgz`) is reachable
and is where that file came from.

**A PDF page gets its own document page, filling it edge to edge** —
`renderPdfAsPages` writes the image straight into each page's editable
rather than going through `insertImage()`. `insertImage` builds a resizable
inline figure (border, margin, drag-to-resize badge) meant for pictures
dropped into flowing text, and one of those is fractionally taller than the
page, so a one-page PDF used to flow onto three document pages. Don't
"simplify" this back into `insertImage`.

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

## E-SAVE sends two files, and why the second one isn't a PDF

The email carries the `.docu` **and** a standalone `.html` copy. The `.docu` is
the editable original that only DZDocu opens; the HTML is so a recipient
without the app gets something they can actually read — and, because it ships
with the app's own stylesheet, printing it produces the same pages the app
would. That is also the PDF story: **the app has no PDF writer.** Every PDF it
makes comes from `window.print()`, and the browser hands that file straight to
the save dialog — JavaScript never sees the bytes, so a PDF simply cannot be
attached without either a rasterising library (loses real text, and none of the
print CSS) or server-side rendering (Cloudflare Browser Rendering, a paid
add-on). Neither was worth it; the HTML gets a recipient to a correct PDF in
one keystroke.

`buildStandaloneHtml()` **clones the live `<section class="page">` elements**
rather than rebuilding from state — they already render boxes, labels, headers
and full-bleed PDF pages correctly, and there is no second code path to keep in
step. Three things it must keep doing:

- **Strip the inline layout `updateColsLayout()` writes onto each section.** In
  a two-page spread that's `width:calc(50% - 28.8px)` plus an `aspect-ratio`,
  and being inline it beats the exported sheet geometry — the export came out
  654px wide and spilled onto a third sheet until this was cleared. The sheet
  rules also carry `!important` as a second line of defence.
- **Carry the whole app stylesheet** (every `<style>` in the document), so
  `.editable`, fonts and the print rules behave identically. Its UI rules match
  nothing in the export.
- **Restate the page box in inches**, because doc-page's geometry lives in a
  shadow root that can't travel with the clone.

The `htmlFilename`/`htmlDataUrl` fields are **additive** — a Worker predating
them ignores both and sends the `.docu` alone, so app and Worker can be
deployed in either order. The client drops the HTML (keeping the `.docu`) if
the pair would exceed the Worker's 15MB cap; a document full of PDF pages can
be most of that by itself.

The footer link back to dzdocu.com is `.dz-made`, screen-only — deliberate, and
deliberately never printed.

## The magnifier places the caret; it is not a second editor

Clicking or dragging in the magnifier strip moves the **document's** selection
— it never becomes editable itself. That was the deliberate choice over making
the bar a real editable mirror, which would have meant a second editor kept in
sync with the first (two-way offset mapping, formatting preservation, write-back)
for no gain: once the selection is right, the toolbar and the keyboard already
do everything.

Three things make it work, and all three are load-bearing:

- **`collapseWithMap()` records where every shown character came from.** The bar
  collapses `\s+` to one space, which destroys the 1:1 offset relationship with
  the document — without the map there is no way back from "character 37 in the
  bar" to a caret position. `magnifierModel()` also builds the reverse
  (`shownAt`) for painting.
- **The painted string no longer depends on the caret.** It used to be split at
  the caret with each half collapsed separately, so the text changed shape
  whenever the selection moved. Fine for a read-only mirror, impossible to drag
  on — the coordinates would shift under the mouse. Collapse the whole block
  once; paint the splits at arbitrary indices.
- **`_magDragging` freezes the repaint for the duration of a drag.**
  `paintMagnifier` rebuilds its spans from the selection, and a drag changes the
  selection on every `mousemove`, so an unfrozen bar would destroy the very
  elements the pointer coordinates are read from. Same self-defeating loop that
  blocked selection in the bar in the first place.

Also: `mousedown` calls `preventDefault()` so the editable keeps focus and the
browser doesn't start a native selection inside the bar — a document only gets
one selection, and it has to be the document's. Each painted span carries
`_magStart` (the shown index of its first character), which is what makes a
click reversible; any new span added to the paint needs it too.

Only the strip's *text* takes pointer events; the strip around it stays
`pointer-events:none` so clicks still reach the page behind it.

Scope limit worth knowing before a bug report: the bar only ever mirrors the
**block the caret is in** (`host.closest('div,p,li,…')`), so it can't reach the
rest of the page. The whole-document big-text view is Reflow, which is
phone-only (`reflowVisible` requires `narrowScreen`) and rewrites pagination on
the way in and out.

## Click-away dismissal must check where the press STARTED

A modal backdrop that closes on `click` closes when a user drag-selects text
inside the panel and releases a few pixels outside it. The `click` event's
target is the nearest common ancestor of mousedown and mouseup, so an
overshooting selection lands its click on the backdrop — the inner panel's
`stopClickBubble` is no help, because the click really did happen on the
backdrop, it just didn't *start* there. Reported as the Save to Email box
"closing randomly" and refusing to let several letters be selected and
retyped; the address field is narrow, so overshooting it is the normal case.

All four modal backdrops (add-page, email, font picker, char picker) now pair
`sc-camel-on-mouse-down="{{ onBackdropMouseDown }}"` with a `backdropClose(...)`
wrapper that only fires when the press began on the backdrop itself. Any new
click-away panel needs both halves, not just the click handler.

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
