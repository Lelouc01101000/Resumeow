# Resumeow

Resumeow is a client side CV builder. It runs entirely in the browser with no backend. the CV renders live in a fixed size page next to it, and the result can be exported as PNG or PDF. Multiple CVs are kept in `localStorage` and every change is saved automatically.

## Stack

| Concern | Choice |
| --- | --- |
| Language | Vanilla JavaScript, one inline `<script>` in `resumeow.html` |
| Styling | Plain CSS in `resumeow.css`, theme values as custom properties |
| Rendering | Template literals written to `innerHTML` |
| Persistence | `localStorage` |
| Export | html2canvas 1.4.1 and jsPDF 2.5.1 from cdnjs |
| Fonts | Google Fonts (Fraunces, Lora, Playfair Display, Cormorant Garamond, Sora, Manrope, Inter, Work Sans, Nunito Sans, Karla) |

## Files

```
resumeow.html   markup and all application logic
resumeow.css    app chrome, editor panels, and the 8 resume templates
assets/images/resumeow_logo.png   favicon
```

## Running locally

There is nothing to install. Open `resumeow.html` in a browser, or serve the folder:

```
python3 -m http.server 8000
```

Network access is still needed for the CDN scripts and fonts. Without it the editor works but export does not, and text falls back to the system font stacks.

## State model

The active CV is described by one plain object, `state`. Its shape is defined by `defaultState()`.

| Field | Type | Notes |
| --- | --- | --- |
| `schemaVersion` | number | Currently `1` |
| `theme` | `'light'` or `'dark'` | Stored per CV, applied as `data-theme` on `<html>` |
| `template` | `1` to `8` | Selects the layout branch in `renderResume()` |
| `accent` | hex string | One of the six values in `ACCENTS` |
| `font` | string | Key of `FONT_PAIRS`: `serif`, `mixed`, `sans`, `refined`, `geometric`, `classic` |
| `photoShape` | `'circle'`, `'rounded'`, `'square'` | Mapped to a border radius by `shapeRadius()` |
| `photo` | data URL or `null` | Read with `FileReader`, stored inline |
| `name`, `title` | string | Header fields |
| `contacts` | object | `email`, `phone`, `location`, `website`, each `{ value, show }`. `website` also has `url` |
| `motivation` | array | `{ id, title, content }` |
| `sections` | array | `{ id, type: 'text', title, content }` or `{ id, type: 'list', title, items[] }` |

Ids come from `newId(prefix)`. Every user supplied string passes through `esc()` before it is interpolated into markup.

When the shape of `state` changes, bump `STATE_SCHEMA_VERSION` and add a case to `migrateState()`. Older stored CVs go through that function every time they are activated, so old data keeps loading.

## Rendering

`renderResume()` rebuilds the contents of `#resume-page` from `state`. It sets `--r-accent` and `--r-accent-soft` on the page, picks the font pair, then writes the markup for the selected template and applies the class `tpl1` to `tpl8`. It ends by calling `scheduleSave()`, which makes it the single hook that turns any edit into a save.

The editor side has its own render functions (`renderContactFields`, `renderMotivationEditor`, `renderSectionsEditor`, `renderSwatches`, `renderPhotoPreview`). Static inputs are filled by `bindStaticFields()`. `refreshEditorFromState()` runs all of them in order and is what a CV switch uses to redraw everything from a different `state`.

## CV collection and autosave

There is no save or update action. All CVs live in one array and edits are written back automatically.

**Storage keys**

| Key | Content |
| --- | --- |
| `resumeow_saved_cvs` | JSON array of CV entries |
| `resumeow_active_cv_id` | Id of the CV that was open last |

**Entry shape**

```js
{ id, name, updatedAt, state }
```

**How it fits together**

1. `state` is always the same object as `activeCv().state`. Editing the CV edits its stored entry directly, so nothing is copied on save.
2. `renderResume()` calls `scheduleSave()`, which debounces `saveNow()` by `SAVE_DELAY_MS` (450 ms).
3. `saveNow()` serializes `state` and compares it with `persistedSnapshot`. If nothing changed it returns without writing. This keeps `updatedAt` from changing when a CV is only opened, since activating a CV also triggers a render.
4. When the snapshot differs, `updatedAt` is set, the whole array is written with `writeCvs()`, and the snapshot is updated only if the write succeeded.
5. `saveNow()` is called directly before a CV switch, before creating a new CV, on `visibilitychange` to hidden, and on `pagehide`. A pending debounce is therefore never lost.

**Boot sequence**

```
loadCvs() -> loadActiveCvId() -> adoptLegacyDraft()
  -> create "CV 1" if the collection is empty
  -> activateCv(active id, or the first CV if that id no longer exists)
```

`activateCv(id)` fills missing fields from `defaultState()`, runs `migrateState()`, points `state` at the entry, records the snapshot, stores the active id and redraws the editor and the CV list.

**Rules**

* The collection is capped at `MAX_CVS` (10). Creating an eleventh shows a notice and does nothing.
* The collection is never empty. The delete control is not rendered while only one CV exists.
* Deleting the active CV activates its neighbour. Delete offers a five second undo that reinserts the entry at its old index, provided there is still room under the cap.
* New CVs get the first free name of the form `CV n`. Names are editable inline.
* A failed `localStorage.setItem` (usually quota) adds the class `save-failed` to the CV menu button. The next successful write clears it.

**Legacy data**

Earlier versions stored a working draft under `resumeow_state` next to manually saved CVs. `adoptLegacyDraft()` runs once at boot. If a CV was linked as active, the draft replaces that CV's state. Otherwise the draft is added as a new CV if there is room. The old key is removed only after the new collection was written successfully.

## Export

`exportCanvas()` sets the wrapper scale variable `--rs` to `1`, captures `#resume-page` with html2canvas at `scale: 2` on a white background, then restores the previous scale. Capturing at the true 794 px layout width keeps the output identical regardless of viewport size.

* PNG: `canvas.toDataURL('image/png')` through a temporary anchor.
* PDF: jsPDF with `unit: 'px'`, `format: [canvas.width, canvas.height]` and the `px_scaling` hotfix, so the file is a single page sized to the content.

The file name is `state.name` with whitespace replaced by underscores, or `resume` when the name is empty.

Both formats are raster captures. Text in the PDF is not selectable and links are not clickable.

## Layout and responsiveness

The resume is always laid out at 794 px wide. On narrower viewports `fitResumeToViewport()` sets `--rs` on `#resumeScale` and CSS scales the page with a transform. The export path bypasses that scale as described above.

At 860 px and below, the sidebar and the canvas become full screen panels. The Edit and Preview buttons in `#mobileNav` set `data-mobile-view` on `.app`. Above 861 px that attribute is ignored.

## Deployment

The site is static and can be served from any static host. Hosts such as GitHub Pages let browsers keep an old copy of `resumeow.css`, so the stylesheet URL carries a version query string. Whenever the CSS changes, update the same value in both places in `resumeow.html` and deploy the two files together:

```html
<meta name="asset-version" content="2026.09.24-1">
<link rel="stylesheet" href="resumeow.css?v=2026.09.24-1">
```

## Extending

**Add a template**

1. Add a button with `data-tpl="9"` to `#templateTabs`.
2. Add an `else if (state.template === 9)` branch in `renderResume()` that sets `page.className = 'tpl9'` and writes the markup.
3. Add the `.tpl9` rules to `resumeow.css`. Scope them under the class so they cannot affect the other templates.

**Add a font pairing**

Add an entry to `FONT_PAIRS`, an `<option>` to `#fontSelect`, and the families to the Google Fonts link in `<head>`.

**Add a state field**

Add it to `defaultState()`. Because `activateCv()` merges stored data over the defaults, a missing field is filled in for old CVs. Only bump the schema version and write a migration when an existing field is renamed, removed or restructured.

## Known limits

* `localStorage` is per browser and per origin, and is typically capped around 5 MB. Photos are stored as data URLs inside the entries, so large photos across several CVs are the usual way to reach the cap.
* There is no sync between devices. Clearing site data removes every CV.
* Export depends on html2canvas fidelity, so very new CSS features may not render exactly as they do on screen.
