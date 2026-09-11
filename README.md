# Garage

A personal vehicle maintenance log that runs as a single HTML file. Track
services, mileage, reminders, photos, and notes for each car — stored entirely
on your device, no accounts, no backend.

Built to replace a paid app that paywalled basic features. Designed for iPhone,
installed via Safari → Share → **Add to Home Screen**.

## Running it

Open `index.html` in a browser, or serve it from any static host. There is no
build step and no dependencies beyond the Google Fonts request.

## Deploying (GitHub Pages)

1. Keep the repo **public** — Pages won't serve a private repo on a free
   account; it redirects to github.com, which looks like the site is broken.
2. Settings → Pages → deploy from `main`, root folder.
3. Open the Pages URL on your phone and Add to Home Screen.

Keep the URL stable across updates: your data is scoped to the origin, so
re-deploying to the same repo preserves everything. A new URL is an empty garage.

## Backups

Settings → **Export backup** writes a JSON file (on iOS it opens the share sheet
so you can save it to Files or AirDrop it). **Import backup** restores it.

See [CONTEXT.md](CONTEXT.md) for architecture, design decisions, and things that
were tried and rejected.
