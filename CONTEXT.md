# Garage — project context

Single-file vehicle maintenance tracking PWA. Personal use only, built as a
free replacement for a paid app that paywalled basic features (notes, photos,
custom fields).

## Hard constraints

- **One HTML file.** No build step, no bundler, no npm. `index.html` is the app.
- **No external runtime deps for the app itself.** Google Fonts (Sora +
  JetBrains Mono) is the only network request on load. Everything else is inline
  or base64. The one exception is the on-demand "Cut out car" button, which
  lazy-loads a background-removal model from a CDN — the app works fully
  without it.
- **On-device storage only.** No backend, no accounts, no sync. IndexedDB is the
  primary store (localStorage is a best-effort mirror and the migration source
  for pre-IDB data). JSON export/import in Settings is the backup mechanism.
- Target device is iPhone, installed via Add to Home Screen. Safe-area insets and
  standalone mode are already wired.

## Architecture

Vanilla JS, no framework. File is organized in commented sections:

```
STATE → STORAGE → UTILS → ICONS → OWNER'S MANUAL DATA → SCREENS → SHEETS → ROUTER → ACTIONS → INIT
```

- `STATE` is a single mutable object. `render()` rebuilds `#app` innerHTML from it.
  Fields prefixed `_` are transient UI state and never persisted.
- Every `onclick` routes through the `action` object. Nothing calls internals directly.
- `persist()` writes on every mutation (fire-and-forget; a failure alerts once
  instead of being swallowed).
- `loadState()` is async — `INIT` is `loadState().then(render)`.
- Vehicle shape: `{id, year, make, model, trim, plate, mileage, unit, vin, note,
  heroPhoto, heroFloating, photos[], services[], reminders[]}`
- `vehicle.manual` is an optional key into `MANUALS` (see below).
- Service (history entry): `{id, items[], title, date, mileage, cost, note}` —
  `items` is what was done; `title` is a derived join kept for older entries
  and the global Services list.
- Service item (roster entry, `v.items[]`): `{id, title, status, priority,
  deferred, dueDate, dueMileage, intervalMonths, intervalMiles, code, note,
  custom}`. `status` is `needed | ok | ignored`. Dates are ms timestamps at
  **local** midnight — always go through `fromDateInput` / `toDateInput`.

### The Service tab

One tab holds everything about maintenance. Every vehicle carries a standing
**roster** of service items (`ROSTER`, seeded by `ensureRoster()` — idempotent,
called on render). The tab shows:

1. **Needed** — items whose `itemState()` is needed: status `needed`, or
   status `ok` with a due date/mileage that is overdue or within 14 days.
   Sorted priority first, then urgency. Colored bar: red overdue, amber soon,
   grey deferred.
2. **Everything else** — the rest of the roster with "Next · …" (has a due)
   or "Last · date" (from history) or "Never logged". Ignored items in a
   dimmed group at the bottom.
3. **History** — the logged services.

Tapping any item opens the **item sheet**: status (Needed / OK / Ignore),
high-priority toggle, due date/mileage, defer quick-picks (set the due and
flag `deferred`), repeat interval, notes, and **Log as done**. Custom items can
be removed; roster items can only be ignored.

**Log Service** is a checklist: every needed item is a row (Done / Later),
every other non-ignored roster item is a chip, plus free text for one-offs.
Saving records the entry and resets each included roster item to `ok`
(clearing due/deferred/priority/code); if the item has an interval, the next
due is set from the service date/mileage. "Later" rows are untouched. The
circle on a Needed row opens this sheet with that item pre-ticked — there is
no "mark done" that bypasses the log.

The Minder decoder's "Mark these as needed" flips the matching roster items
(via `ROSTER[].codes`) to needed with the code badge; it never creates items.
The manual's calendar rules live in `MANUAL_INTERVALS` and are applied as
repeat intervals on Engine oil & filter (12 mo) and Brake fluid (36 mo).

`migrateVehicle()` upgrades pre-roster data (`reminders[]`) on load.
Checklist and item-sheet state patch the DOM — same no-re-render rule as
every other sheet.

## Design language

- Dark. `--bg: #0a0d0c`, accent `--accent: #00d68f`.
- Sora for UI, JetBrains Mono for data/labels (uppercase, letterspaced).
- "Garage stall" motif: radial floor glow, faint bay-line texture, overhead
  spotlight gradient. Cars sit in a lit bay.
- Icons are 1.5–1.75 stroke weight, 18–19px. Thin and sharp. Earlier versions were
  too heavy and got rejected.

## The home screen is deliberate — do not add to it

The garage view is a **full-screen horizontal photo carousel and nothing else.**
No title, no car name, no stats, no dots, no page indicator, no drawer button.
Swipe between cars, tap to enter detail.

This went through several rounds. Stats and labels were explicitly removed. The
only thing allowed on that screen is a pulsing red dot above the car when that
vehicle has an overdue reminder (`vehicleHasOverdue()`).

Bottom nav (Home / Services / Settings) stays. That's the only chrome.

## Photo handling

`processImage()` samples the alpha channel after drawing to canvas:

- **Transparent PNG** → `heroFloating: true` → car floats centered with drop
  shadow over the stall background. Saved as PNG to preserve alpha.
- **Opaque image** → full-bleed `object-fit: cover` with a gradient overlay.
  Saved as JPEG at 0.82 for size.

Cutouts can be made in-app ("Cut out car" in the vehicle form) or externally
(iOS Photos → press-and-hold the car → Copy Subject; ChatGPT; any editor) and
imported. Transparent images are cropped to their visible pixels on import
(`alphaBounds`) so cutouts with big transparent margins still fill the frame.

**In-app cutout** uses `@imgly/background-removal` (ISNet, `isnet_quint8`),
dynamically `import()`ed from jsdelivr on first tap, ~40 MB of model + ONNX
runtime downloaded once and browser-cached. `proxyToWorker: false` is
mandatory — the worker path throws `DataCloneError` on iOS Safari (this is what
killed the first attempt). Runs on the main thread, ~10 s on a laptop; slower on
a phone. Threaded WASM is unavailable on GitHub Pages (no COOP/COEP headers) so
it's single-threaded. "Use original photo" reverts to the pre-cutout image
until the sheet is saved or closed.

Plain (opaque) photos on the home screen render as a framed print in the stall
(86% width, rounded) rather than full-bleed — full-bleed made every photo look
enormous.

The no-photo placeholder is a base64 PNG car silhouette embedded in
`CAR_SILHOUETTE`, rendered with `filter: invert(1)` and 35% opacity.

## Tried and rejected — don't redo these

- **`@imgly/background-removal` with its default worker** — throws
  `DataCloneError: The object can not be cloned` on iOS Safari. Now used with
  `proxyToWorker: false` instead (see Photo handling). If it misbehaves on the
  phone (memory, hang), that flag is the first thing to check, not the library.
- **`capture="environment"` on file inputs** — forces the camera and blocks the
  photo library. All file inputs must omit it.
- **Hand-drawn SVG car silhouettes** — attempted twice, both looked terrible.
  Current base64 PNG stays.

## Known issues / open items

- No service worker, so no true offline caching. Works offline once loaded.
- iOS PWAs can't do push notifications, so reminders are visual only.
- Photos are still base64 strings inside the one state blob, so every
  `persist()` rewrites all of them. Fine at tens of photos; if it ever feels
  slow, split photos into their own IDB records.
- Roster items match history by normalized title (`normTitle`), so renaming a
  custom item orphans its "Last · …" lookup.
- The manual's calendar intervals only start counting after the first logged
  service of that item (no way to know when it was last done before the app).
- In-app cutout is verified on desktop Chrome only. Needs a real run on iOS
  Safari (memory pressure on the ~40 MB model is the risk).

## iOS specifics

- `apple-touch-icon` is an inline base64 PNG (iOS ignores the manifest's icons
  and needs PNG, not SVG). Regenerate via a canvas if the look changes.
- Export uses `navigator.share({ files })` when available — a plain
  `<a download>` doesn't reliably work in standalone mode.

## Deployment

GitHub Pages. The repo must be **public** (Pages won't serve private repos on
free accounts — it redirects to github.com instead, which looks like the site is
broken). The file must be named `index.html` at repo root.

Same URL across updates matters: localStorage is scoped to the origin, so
re-uploading to the same repo preserves all data. A new URL means an empty garage.
