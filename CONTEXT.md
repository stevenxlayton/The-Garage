# Garage — project context

Single-file vehicle maintenance tracking PWA. Personal use only, built as a
free replacement for a paid app that paywalled basic features (notes, photos,
custom fields).

## Hard constraints

- **One HTML file.** No build step, no bundler, no npm. `index.html` is the app.
- **No external runtime deps.** Google Fonts (Sora + JetBrains Mono) is the only
  network request. Everything else is inline or base64.
- **localStorage only.** No backend, no accounts, no sync. JSON export/import in
  Settings is the backup mechanism.
- Target device is iPhone, installed via Add to Home Screen. Safe-area insets and
  standalone mode are already wired.

## Architecture

Vanilla JS, no framework. File is organized in commented sections:

```
STATE → STORAGE → UTILS → ICONS → SCREENS → ROUTER → ACTIONS → INIT
```

- `STATE` is a single mutable object. `render()` rebuilds `#app` innerHTML from it.
- Every `onclick` routes through the `action` object. Nothing calls internals directly.
- `persist()` writes to localStorage on every mutation.
- Vehicle shape: `{id, year, make, model, trim, plate, mileage, unit, vin, note,
  heroPhoto, heroFloating, photos[], services[], reminders[]}`

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

Cutouts are produced externally (ChatGPT, any photo editor) and imported. The app
does not remove backgrounds itself — see below.

The no-photo placeholder is a base64 PNG car silhouette embedded in
`CAR_SILHOUETTE`, rendered with `filter: invert(1)` and 35% opacity.

## Tried and rejected — don't redo these

- **`@imgly/background-removal`** — throws `DataCloneError: The object can not be
  cloned` on iOS Safari. Web Worker postMessage incompatibility. Removed. Any
  in-browser background removal needs to be verified on real iOS Safari before
  it goes in.
- **`capture="environment"` on file inputs** — forces the camera and blocks the
  photo library. All file inputs must omit it.
- **Hand-drawn SVG car silhouettes** — attempted twice, both looked terrible.
  Current base64 PNG stays.

## Known issues / open items

- Detail page shows `—` for every empty field (VIN, total spent, last service).
  Should hide empty spec cards instead of rendering dashes.
- Photos live in localStorage as base64. Fine for normal use, will hit the ~5MB
  quota if photos pile up. IndexedDB is the eventual fix.
- No service worker, so no true offline caching. Works offline once loaded.
- iOS PWAs can't do push notifications, so reminders are visual only.

## Deployment

GitHub Pages. The repo must be **public** (Pages won't serve private repos on
free accounts — it redirects to github.com instead, which looks like the site is
broken). The file must be named `index.html` at repo root.

Same URL across updates matters: localStorage is scoped to the origin, so
re-uploading to the same repo preserves all data. A new URL means an empty garage.
