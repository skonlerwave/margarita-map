# Seattle Margarita Map — Build Spec

Oct 21, 2026 · @Ryan Faber

## Overview

Build a small, static, mobile-first website that shows Seattle's best margaritas and Margarita Monday deals on a map. It is a personal project: publicly viewable, curated by one person (Ryan), with all data in a hand-edited YAML file. The data layer is shaped so a backend can replace the file later without frontend changes.

Goals:

- Answer "where should I get a margarita right now?" from a phone in under 10 seconds.
- Make adding or updating a spot a one-file edit plus a push.
- Keep it cheap and low-maintenance: no database, no accounts, no paid APIs.
- Show Ryan's own ratings and tasting notes, so it has a point of view.

## Scope

v1 is a single-page static site with a map, a list, filters, and a "today" mode, fed by one YAML file.

In v1:

- Map view and list view of all spots, kept in sync.
- Venue detail panel with margaritas, ratings, notes, and any deals.
- Filters: Margarita Monday deals, deal active now, neighborhood, margarita style.
- "Today" mode that highlights deals running today, using Pacific time.
- "Last verified" date on every deal, with stale deals flagged.
- A data validation script that fails on malformed entries.
- Deploy to a ryanfaber.net subdomain on Cloudflare.

Out of v1 (do not build):

- User accounts, submissions, comments, or a submission form.
- A database or backend API (planned for later; see Backend-ready data layer).
- Per-venue URLs and SEO work.
- Analytics, cookies, or tracking.
- Paid map providers (Google Maps, Mapbox) or Google Places data.
- Photos (add later, own photos only).

## Stack and hosting

Plain HTML, CSS, and JavaScript with Leaflet, served as static assets from a Cloudflare Worker on a ryanfaber.net subdomain.

| Concern | Choice | Notes |
| --- | --- | --- |
| Frontend | Vanilla HTML/CSS/JS (ES modules) | No framework, no build step for the frontend itself |
| Map library | Leaflet 1.9.x | Load from a CDN or vendor into the repo |
| Map tiles | OpenStreetMap standard tiles | Free; must show OSM attribution |
| Data source | `data/spots.yaml` | Hand-edited, committed to git |
| Data served | `public/data/spots.json` | Generated from the YAML by a build script; not committed |
| Hosting | Cloudflare Workers static assets | Same account as the personal site |
| Domain | `margs.ryanfaber.net` (placeholder) | Custom domain on the Worker |
| Tooling | Node + wrangler + `yaml` npm package | `npm run dev`, `validate`, `build`, `test`, `deploy` |

Keep it in its own repo, separate from the personal-website Worker, so the two deploy independently.

## Data model

All content lives in `data/spots.yaml`: a list of venues, each holding its margaritas and deals. Ratings and deals are separate so a venue can have one, the other, or both. The build turns this into `public/data/spots.json` with the same shape, which is the contract the frontend reads.

**Venue**

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `id` | string | yes | Kebab-case slug, unique |
| `name` | string | yes |  |
| `address` | string | yes | Street address, Seattle |
| `lat`, `lng` | number | yes | Entered by hand |
| `neighborhood` | string | yes | e.g. Capitol Hill, Ballard, Fremont |
| `website` | string (URL) | no |  |
| `margaritas` | Margarita\[\] | no | May be empty |
| `deals` | Deal\[\] | no | May be empty |

**Margarita**

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `name` | string | yes | As listed on the menu |
| `style` | enum | yes | `classic`, `spicy`, `frozen`, `mezcal`, `fruit`, `other` |
| `price` | number (USD) | no | Regular price |
| `rating` | number 1–10 | yes | Overall score |
| `scores` | object | no | `balance`, `lime`, `tequila`, `value`, each 1–5 |
| `notes` | string | no | Short tasting notes |
| `tried_on` | date (YYYY-MM-DD) | no |  |

**Deal**

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `title` | string | yes | e.g. "Margarita Monday" |
| `days` | enum\[\] | yes | `mon` … `sun` |
| `start`, `end` | time (HH:MM, 24h) | no | Omit both for all day |
| `price` | number (USD) | no |  |
| `conditions` | string | no | e.g. "Bar seating only, house marg" |
| `last_verified` | date (YYYY-MM-DD) | yes | Drives the stale flag |

Example entry:

```yaml
- id: example-cantina
  name: Example Cantina
  address: 123 Example Ave, Seattle, WA
  lat: 47.6145
  lng: -122.3200
  neighborhood: Capitol Hill
  website: https://example.com
  margaritas:
    - name: House Margarita
      style: classic
      price: 13
      rating: 8
      scores: { balance: 4, lime: 5, tequila: 3, value: 4 }
      notes: Fresh lime, not too sweet.
      tried_on: 2026-10-12
  deals:
    - title: Margarita Monday
      days: [mon]
      start: "15:00"
      end: "18:00"
      price: 7
      conditions: Bar only
      last_verified: 2026-10-12
```

Quote times like `"15:00"` so YAML doesn't read them as numbers. The build script converts dates to `YYYY-MM-DD` strings in the JSON.

## Features and UX

Design for a phone first: someone standing on a sidewalk deciding where to go.

**Layout**

- Phone: full-screen map with a bottom sheet listing spots; a toggle switches between map and list.
- Desktop (≥ 900px): list in a left sidebar, map on the right.
- Header: site name, the filter bar, and a "Today" toggle.

**Map**

- One marker per venue, centered on Seattle (about 47.61, -122.33), zoom \~12.
- Venues with a deal active right now get a distinct marker style (color or badge), not color alone.
- Tapping a marker opens the venue detail; the matching list item highlights.
- "Locate me" button using browser geolocation, only on tap; never request location on load.

**List**

- Shows the venues matching current filters, in sync with the map.
- Default sort: best rating first; when "Today" is on, deals active now first, then deals later today.
- Each row: name, neighborhood, top margarita rating, and a deal badge if relevant.

**Filters** (combine with AND)

- Margarita Monday: venues with a deal whose `days` include `mon`.
- Deal active now: current Pacific time falls inside a deal's day and time window.
- Neighborhood: multi-select from values in the data.
- Style: multi-select from the style enum.
- Show a "clear filters" control and a count of matches.

**Today mode**

- All "now" and "today" logic uses `America/Los_Angeles` via `Intl`, regardless of the viewer's device time zone.
- On Mondays, today mode is on by default.
- Deal labels read "Now until 6pm", "Later today, 3–6pm", or "All day".

**Venue detail**

- Name, neighborhood, address, website link, and a "Directions" link (Google/Apple Maps URL from lat/lng).
- Margaritas: name, style, price, rating, rubric scores if present, notes.
- Deals: title, days, time window, price, conditions, and "Last verified" date.
- Deals with `last_verified` older than 90 days show a "may be out of date" note.

**Other**

- Small "How I rate" section or modal explaining the 1–10 score and the four rubric scores.
- Empty state when no venues match filters.
- Accessible: keyboard-reachable controls, labeled buttons, sufficient contrast.

## Design

Two themes from one system: **Citrus pop** (light) and **Neon bar** (dark), switched by `prefers-color-scheme`. Match the [mockups](https://claude.ai/artifact/GucFG2HC4BYSsJaK8u8hdA): phone light, phone dark with a venue open, desktop light, desktop dark.

**Typography** (Google Fonts)

| Role | Font | Use |
| --- | --- | --- |
| Display | Bricolage Grotesque 600/800 | Venue names, banners, ratings, stickers; light-theme wordmark |
| Body | DM Sans 400/500/700 | Everything else |
| Neon script | Yellowtail | Dark-theme wordmark and Monday banner only |

**Color tokens** (define as CSS custom properties, swapped per theme)

| Token | Citrus pop (light) | Neon bar (dark) | Use |
| --- | --- | --- | --- |
| `--ground` | #FFFDF5 | #14111C | Page and panels |
| `--surface` | #FFFDF5 | #1E1A2B | Sheets, cards, popups |
| `--ink` | #1E2A16 | #F3F0FF | Text, outlines |
| `--ink-muted` | #4D5A43 | #B9B2D0 | Secondary text |
| `--now` | #C0134F | #FF5FAE | Deal on now |
| `--later` | #9BE15D | #B6FF3B | Deal later today, primary dark button |
| `--sticker` | #FFE14D | n/a | Price stickers, Monday banner (light) |
| `--none` | #FFFDF5 marker | #8C84A8 dot | No deal today |
| `--stale` | #8A4B00 | #FFC46B | Stale-deal note |
| `--map-land` / `--map-water` | #F3EEDC / #BFE3E0 | #1A1724 / #0E2630 | Map styling |

**Theme character**

- Citrus pop: 2–2.5px ink outlines, hard offset shadows (2–4px, no blur), pill buttons, slightly rotated stickers (±3–8°), dashed dividers.
- Neon bar: 1.5px colored outlines with soft glows (`box-shadow` / `text-shadow` in `--now` or `--later`), no hard shadows, no rotation.

**Markers and legend**

- Deal on now: largest marker in `--now`, margarita-glass icon, price sticker ("$7").
- Deal later today: medium marker in `--later`, start-time sticker ("4pm").
- No deal today: small `--none` marker, no sticker.
- Stickers and list badges include the words NOW or LATER, so meaning never relies on color alone.
- A legend is always visible on the map: compact pill on phone, card on desktop.
- Selected venue: desktop shows a popup anchored to its marker; phone opens the venue in the bottom sheet.

**Components**

- Ratings: number tile ("9 /10") plus five lime slices, filled = round(rating ÷ 2).
- Rubric scores: four thin bars (balance, lime, tequila, value).
- Monday banner: tilted yellow sticker (light) or neon-script card (dark), shown only on Mondays.
- Phone: header with wordmark, Today toggle and filter chips; bottom sheet with Map/List toggle; locate-me button.
- Desktop: top bar with filters; list sidebar (340–420px) beside the map; zoom buttons on the map.

**Copy voice**: playful and short. Empty state "Dry spell. No margs match those filters."; stale deal "This deal's lime may have dried out"; loading "Shaking…".

**Map tiles**: the mockups use a hand-drawn stand-in. In the build, use tiles styled to match each theme (for example CARTO Positron/Dark Matter, confirming their terms and attribution), with OSM attribution visible.

**Motion and accessibility**: subtle pulse on now-deal markers only; disable all animation under `prefers-reduced-motion`. Text meets 4.5:1 contrast in both themes; touch targets are at least 44px.

## Project structure and implementation notes

Suggested layout:

```
seattle-margarita-map/
  data/
    spots.yaml       # source of truth, hand-edited
  public/
    index.html
    styles.css
    js/
      main.js        # boot, state, wiring
      data.js        # loads spots; the only module that knows where data comes from
      map.js         # Leaflet setup, markers
      list.js        # list rendering
      filters.js     # filter state + matching
      time.js        # Pacific-time helpers, deal status
      detail.js      # venue detail panel
    data/
      spots.json     # generated by build; gitignored
  scripts/
    build-data.mjs   # YAML -> JSON
    validate.mjs     # schema + sanity checks for spots.yaml
  test/
    time.test.mjs    # deal-status tests
  wrangler.jsonc
  package.json
  README.md
```

Notes for the implementer:

- Keep filter and deal-time logic as pure functions in `filters.js` and `time.js`, so they can be unit-tested with Node's built-in test runner.
- `time.js` must answer, for a deal and a moment: active now, later today, earlier today, or not today. Test Monday edge cases: before start, exactly at start, exactly at end, no time window, and a viewer in another time zone.
- `data.js` exposes one function, `loadSpots()`, that fetches `/data/spots.json`. Nothing else in the frontend fetches data, so a backend can later swap in by changing only this URL.
- `validate.mjs` parses `spots.yaml` and checks required fields, enum values, unique `id`s, lat/lng inside a Seattle bounding box (roughly 47.48–47.74, -122.46 to -122.22), date formats, and `start` before `end`. It exits non-zero on any error with the venue id and field in the message.
- `build-data.mjs` runs validation, then writes `public/data/spots.json`.
- Seed `spots.yaml` with 5 clearly fake placeholder venues spread across neighborhoods, including one Monday deal, one all-day deal, and one stale deal, so every UI state is visible.
- No external requests besides map tiles and the Leaflet CDN (if not vendored).
- README covers: how to add a spot, how to run locally, how to validate, how to deploy.

## Deployment

Deploy as an assets-only Worker with a custom domain on the existing ryanfaber.net zone; no Worker script is needed.

Starting `wrangler.jsonc` (confirm keys against current wrangler docs):

```json
{
  "name": "seattle-margarita-map",
  "compatibility_date": "2026-10-01",
  "assets": { "directory": "./public" },
  "routes": [
    { "pattern": "margs.ryanfaber.net", "custom_domain": true }
  ]
}
```

Scripts in `package.json`:

- `validate`: `node scripts/validate.mjs`.
- `build`: `node scripts/build-data.mjs` (validates, then writes the JSON).
- `dev`: run `build`, then `wrangler dev` for local preview.
- `test`: `node --test`.
- `deploy`: run `build` and `test`, then `wrangler deploy`.

Adding or updating a spot:

1. Edit `data/spots.yaml`.
2. Run `npm run validate`.
3. Run `npm run deploy` (or push, if a GitHub Action is added later).

## Backend-ready data layer

The JSON shape in Data model is the API contract; a future backend serves the same shape, so the frontend doesn't change.

- Today: YAML in git → build → static `/data/spots.json`.
- Later: Cloudflare D1 (venues, margaritas, deals tables) plus a Worker route `GET /api/spots` that returns the identical JSON, and a password-protected admin page for adding spots from a phone.
- Migration: a one-time script imports `spots.yaml` into D1; `loadSpots()` switches its URL to `/api/spots`.

For v1, do not build any of the backend. Just keep the shape stable, keep all fetching inside `data.js`, and use the same `id` slugs a database would key on.

## Acceptance criteria

v1 is done when every box below is true.

- [ ] `npm run dev` serves the site locally with the 5 seed venues on the map and in the list.
- [ ] `npm run build` turns `data/spots.yaml` into `public/data/spots.json` matching the Data model shape.
- [ ] Map and list stay in sync when filters change; tapping a marker or row opens the same venue detail.
- [ ] Margarita Monday, deal-active-now, neighborhood, and style filters work alone and combined.
- [ ] Today mode uses Pacific time even when the device is set to another time zone.
- [ ] Deal labels show now / later today / all day correctly; `npm test` covers the edge cases.
- [ ] Deals older than 90 days show the out-of-date note.
- [ ] `npm run validate` fails on invalid YAML, a missing field, bad enum, duplicate id, out-of-Seattle coordinates, or end before start.
- [ ] Usable on a 375px-wide phone screen with no horizontal scrolling.
- [ ] On desktop (1280px and wider), the list sidebar and map show side by side and both stay usable.
- [ ] OSM attribution is visible on the map. Both themes match the mockups and switch with the device's light/dark setting; the map legend is visible on phone and desktop.
- [ ] No requests beyond map tiles and Leaflet; no cookies or analytics.
- [ ] Deployed and reachable at the chosen ryanfaber.net subdomain over HTTPS.
- [ ] README explains adding a spot, validating, and deploying.

## Open decisions and later ideas

Decide before handoff (Claude Code can use the defaults if left open):

- [ ] Subdomain name. Default: `margs.ryanfaber.net`.
- [ ] Repo name and location. Default: new repo `seattle-margarita-map` under the same GitHub account as the personal site.
- [ ] Visual style: decided; see Design.
- [ ] Stale-deal threshold. Default: 90 days.

Later, not v1:

- Own photos per venue.
- Happy hours on other days of the week.
- Per-venue pages with their own URLs.
- Auto-deploy on push via GitHub Actions.
- The backend described above, then a "suggest a spot" form on top of it.
- A standalone domain pointed at the same Worker.
