# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A source-of-truth trip plan and a single-file mobile web app for a 16-day NZ South Island family road trip (2026/12/19–2027/1/3, 4 adults, self-drive). There is no build step, no package manager, and no test suite — this is a static site plus planning documents, deployed via GitHub Pages.

## Files

- `index.html` — the entire deliverable: one self-contained HTML/CSS/JS file (no separate `.css`/`.js` files, no bundler). This is what gets deployed.
- `itinerary.md` — the canonical day-by-day itinerary (source of truth for dates, times, addresses, check-in/parking info, prices, booking status). `index.html` must be kept in sync with this file whenever it changes.
- `nz_map_points.csv` — geodata (`Day,Spot_Name,Latitude,Longitude,Category,Notes,Booking_URL`) that is manually baked into the `tripMapPoints` JS array inside `index.html` (see below) — it is not fetched at runtime.
- `spots.md` — early draft notes on must-see spots/food; largely superseded by `itinerary.md`.
- `images/` — local photo files referenced by the 攝影打卡 (photo checklist) page. Some filenames contain trailing spaces before the extension (e.g. `Baldwin Street .jpeg`) — preserve them exactly when referencing.

## Running / testing

No dev server or build is required — open `index.html` directly in a browser, or serve the directory statically. To validate a change after editing:

```bash
# JS syntax check (extracts the inline <script> and parses it)
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const code = html.match(/<script>([\s\S]*?)<\/script>/g).pop().replace(/^<script>/, '').replace(/<\/script>\$/, '');
new Function(code);
console.log('JS OK, length', code.length);
"

# HTML tag-balance check
python3 -c "
import re
html = open('index.html', encoding='utf-8').read()
for tag in ['section','article','details','table','div','script']:
    print(tag, len(re.findall(r'<'+tag+r'(?:\s[^>]*)?>', html)), len(re.findall(r'</'+tag+r'>', html)))
"

# CSV structural check (66 rows, 7 columns each)
python3 -c "
import csv
rows = list(csv.reader(open('nz_map_points.csv', newline='', encoding='utf-8')))
print('rows', len(rows), 'bad', len([r for r in rows if len(r)!=7]))
"
```

There is no linter/formatter config — match existing style (2-space indent, no semicolon-heavy JS conventions beyond what's already there) when editing.

## Architecture of `index.html`

Single file, three parts in order:

1. **`<head>`**: PWA meta tags + a data-URI `manifest` (no separate manifest file), a data-URI SVG favicon, and the Leaflet 1.9.4 CSS from unpkg (pinned with SRI `integrity` hash).
2. **Inline `<style>`**: CSS custom properties block at the top (`--bg`, `--primary`, `--accent-purple`, `--accent-orange`, `--accent-green`, etc.) followed by sections in this order: header, page container, top overview map, status tags, day cards, photo tip / accordion, photo checklist, bookings/finance, car-rental & notes, packing list, bottom nav, footer.
3. **`<body>`**: `<header>`, then `<main>` containing five `<section class="page" id="page-*">` blocks toggled by the bottom nav, then `<nav class="bottom-nav">`, then Leaflet JS (pinned via SRI) and one inline `<script>` IIFE at the very end.

The five pages (`data-page` values match `<section id="page-*">` ids):
- `page-itinerary` — day-by-day cards (`<article class="day-card" id="day-N">`), each with a `<ul class="timeline">`, optional `<div class="photo-tip">`, optional `<details class="accordion">` blocks for check-in/parking notes, and a `.day-footer` with meal/stay tags.
- `page-photos` — photo checklist grid, each card's `<img>` points to a local `images/...` file.
- `page-bookings` — booking cards + NZD↔TWD calculator + budget tables.
- `page-info` — car rental / entry requirements / food summary reference tables.
- `page-packing` — per-person tabs (爸爸/媽媽/哥哥/妹妹) + a shared tab.

### The `<script>` IIFE, in this order (grep for the numbered `====` comments)

1. Bottom-nav tab switching (`.nav-btn[data-page]` ↔ `.page` visibility).
2. Custom accordion behavior — overrides native `<details>` toggle with `preventDefault()` and animates via `content.style.maxHeight` (not just relying on native `<details>` open/close).
3. Photo checklist state — `LS_PHOTOS = "nzTrip_photos_v1"`, keyed by `"photo-"+index`.
4. Packing list — per-tab checked/removed/custom-added items, persisted as `LS_PACK_CHECKS` / `LS_PACK_REMOVED` / `LS_PACK_CUSTOM` (currently `_v2`; bump the version suffix whenever the underlying item list/order changes, since state is keyed by index and would otherwise silently mismatch).
5. Booking tracking — `LS_BOOKINGS = "nzTrip_bookings_v3"`, keyed by `"booking-"+index`; drives the dynamic unbooked-items summary (`#unbooked-list`, `#booking-summary-count`). On page load, a saved localStorage entry for a card **overrides** that card's HTML `checked` attribute (see the `saved` branch in the script) — so a returning visitor's browser will keep showing an old pending/booked state even after the HTML default changes underneath it. **Bump the version suffix whenever any `.booking-card`'s checked state changes** (not just on insert/reorder) so existing visitors' browsers discard their stale cached state and re-read the current HTML defaults.
6. NZD↔TWD calculator — `LS_RATE = "nzTrip_calc_rate_v1"`, default rate 19.5.
7. Top overview map (`#trip-map`) — `var tripMapPoints = [...]` is a **hand-maintained copy of `nz_map_points.csv`** as JS objects (`{d,n,lat,lng,cat,note,url}`), baked in rather than `fetch()`-ed specifically to avoid CORS failures when the file is opened via `file://`. Leaflet map init has `scrollWheelZoom:false` (enabled only after a click, so page-scroll on mobile isn't hijacked); day-filter chips filter markers and call `fitBounds`; `catColors` maps `cat` to marker color and must stay in sync with the CSS `--accent-*` variables and the `.map-legend` markup.

**Whenever `nz_map_points.csv` or `itinerary.md` changes, `tripMapPoints` and the corresponding day-card HTML in `index.html` must be updated to match by hand** — there is no build step that does this automatically.

## Content conventions (established across this project's history — follow these when editing any of the three source files)

- **4 adults only, no children** — never apply family/child pricing or family tickets.
- All prices are **NZD**; do not fabricate or "helpfully" adjust figures — flag uncertain/estimated data explicitly rather than guessing.
- Never invent URLs, coordinates, prices, or times. When geocoding is needed, use a real source (e.g. OpenStreetMap Nominatim) and note in the CSV/itinerary that the coordinate was derived from an address lookup and should be reconfirmed.
- **Before adding any new stop, restaurant, or activity to the itinerary — even one the user names casually or pastes from a blog/social post — verify it with a real source (web search, the business's own site) first.** Confirm at minimum: address, current opening hours for the relevant day of week, and reservation/booking policy if that's relevant to the plan. Don't take a secondhand source (blog post, social media caption) at face value — check it against the business's own site or a review aggregator, since blog posts and word-of-mouth recommendations are frequently outdated (e.g. a blog claiming a shop "closes at 3:30pm" when its current official hours are 4:30pm). When verified info conflicts with what the user or a source supplied, use the verified figure and note the discrepancy in the changelog rather than silently overwriting or silently keeping the old figure.
- Status-tag vocabulary is fixed — reuse exactly one of: `📌已訂` (paid/booked), `📌待訂` (not yet booked), `📌待選擇` (option not yet decided), or `以訂單確認信為準` (booked but details pending confirmation). These map to CSS classes `status-booked` / `status-pending` in `index.html`.
- `itinerary.md` is the source of truth; `index.html` and `nz_map_points.csv` are derived from it and must be synced after any edit to it. When making a correction pass, append a dated, numbered changelog section at the bottom of `itinerary.md` (see the existing `V8`/`V9`/`V10` sections) summarizing what changed and why, rather than silently rewriting history.
- Keep edits scoped to what was explicitly requested/confirmed — this project has repeatedly relied on getting user confirmation before applying researched changes (e.g. verifying business hours/check-in locations via web search, reporting findings, then only editing after explicit approval).

## Deployment

Static GitHub Pages site deployed from `main` branch, `/ (root)`, at `https://411730178xian.github.io/nz-south-island-trip/`. Command-line `git push` fails in this environment (`fatal: could not read Username for 'https://github.com'` — no CLI credential helper); the user pushes manually via GitHub Desktop after commits are made here.
