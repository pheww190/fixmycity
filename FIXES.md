# What was fixed in this copy

Base: your `okokokokokok` build. Every page and section is still here — nothing
was removed. The changes are listed below.

## Run it

```powershell
npm start
```

`npm start` now installs the backend dependencies itself on the first run, so
there is no separate `npm install` step to forget.

---

## 1. The buttons did nothing (the big one)

`helmet` sets `script-src-attr 'none'` by default, which blocks **every** inline
`onclick="..."` in the app. Search, Use GPS, Fullscreen, the emergency chips —
none of them could fire. The CSP now allows inline handlers again.

## 2. Map

- **Search box** was squeezed to a few characters in the narrow card. It now
  takes the full width, with the Search and Use GPS buttons on the line below.
- **Fullscreen / Minimize** set the container to full size, but the map canvas
  kept its old size, so it looked broken. The MapLibre canvas is now resized
  together with Leaflet, and pressing Esc resets the buttons.
- **Search** now tries, in order: the built-in Bengaluru locality list →
  Nominatim (Bengaluru biased) → Nominatim → Photon. Photon answers browser
  requests when Nominatim rate-limits or refuses them.
- **Use GPS** explains the real reason when it fails (permission denied, no
  location provider, timeout) and tells you when the page is not on `https://`
  or `localhost`, where browsers block geolocation entirely.
- **Photo geotags**: the pin moves and the map recentres after the EXIF read.

## 3. Login / registration no longer wait 3–4 seconds

The OTP routes did `await sendOtpEmail(...)`, and the first send creates an
Ethereal account over HTTPS. The response now returns immediately; the mail goes
out in the background. SMTP timeouts were added as well.

## 4. Two JavaScript errors on every page

- `/shared/auth.js` was included **twice** on 13 pages, so `AUTH_KEY` was
  declared twice and the second copy threw. Duplicates removed.
- `portal.js` called `updateCoords()`, a function that no longer exists, on every
  load. Replaced with the equivalent inline code.

## 5. `/track` had no styling

The page is served at `/track`, so its relative `track.css` / `track.js` were
requested from the site root and came back as HTML. Both paths are absolute now.

## 6. Backend

- `backend/config.js` — one shared database path for both data modules, and a
  throw-away database when `NODE_ENV=test`. A second path is what produced
  "database is locked".
- Three `if (err) resolve(...)` calls without `return` reported a failed write as
  a success. Fixed.
- The schema is queued as soon as the module loads. A query issued before the
  connection finished opening used to run ahead of `CREATE TABLE` and fail with
  "no such table".
- Duplicate `dotenv` load removed.

## 7. Data

Test data deleted. What is left:

| | Before | After |
|---|---|---|
| Complaints | 92 | **2** |
| Citizens | 22 | **2** |
| Photos | 16 | **1** |

The two remaining complaints are yours (`jordan1019g@gmail.com`): the fallen
electric pole (In Progress, with its photo) and the water leak (Resolved). All
90 rows created by the test suite — "Upload Tester", "Status Tester", "Road
Citizen", "Water Citizen", "Citizen One" — are gone, along with their citizens
and 15 orphaned photos.

---

## Verified

- `npm test` — 26/26 pass
- All 27 pages loaded in a real browser: no JavaScript errors
  (Leaflet and Chart.js are CDN scripts, so they need an internet connection)
- Portal: search box 334 px wide, fullscreen grows the map from 300 px to the
  full viewport height

## 8. A filed complaint did not appear on the map

Two causes, both fixed:

- **The portal filled the hidden coordinates with the city centre**, so any
  complaint filed without moving the pin was saved at 12.9716, 77.5946 — the same
  spot for every complaint. On the public map they stacked under a single marker
  and looked missing. The coordinates now start empty, and submitting without a
  pin is refused with: *"Set the location first — tap the map, use GPS, or search
  an area."*
- **The public map drew one marker per complaint**, so several complaints at one
  location hid each other. Markers are now grouped by location: a single
  complaint gets a pin, several get a numbered badge whose popup lists them all.
  The map also refreshes itself every 60 seconds, so a complaint filed in another
  tab appears without a manual reload.

Existing complaints already saved at the city centre are not moved — they will
now show as one marker with a count instead of disappearing.

## 9. A complaint did not appear on the admin map

The data path was fine — `/api/complaints/all` returns the new complaint. What
was missing was any explanation of *why* a map can legitimately be empty:

- **Officers only see their own department.** That is deliberate: an officer must
  not read another department's citizens' data. A complaint routed to Electricity
  is invisible to a Water Supply officer, with no error message. The map sidebar
  now says so: *"Showing Water Supply Department only. A complaint routed to
  another department will not appear here."* An admin sees everything.
- **The page never refreshed.** If the map tab was open when the complaint was
  filed, it kept showing the old data. It now reloads every 60 seconds.
- **The sidebar now states what is loaded** — "3 complaints on the map" — and
  shows *"Nothing matches the current filters"* when a filter empties it, instead
  of a blank map.
- **Clustering hid stacked pins.** Complaints filed at the same coordinate merged
  into one bubble; clustering now switches off at zoom 15 so zooming in separates
  them.

## 10. Invented data removed

Four panels presented made-up numbers as analysis. They are gone:

- **Admin dashboard, "AI Predictive Insights (Next 7 Days)"** — hard-coded badges
  ("High Risk: Water Logging (Zone B)", "Med Risk: Potholes (Zone C)") with no
  model behind them.
- **Public map, "Red Zones"** — two circles at fixed offsets from the city
  centre, labelled "high frequency of repeated complaints".
- **Public map, "AI Prediction"** — a circle claiming an "85% probability of
  severe waterlogging in the next 15 days".
- **Public map, "Monthly View"** — dimmed markers at random (`Math.random()`) to
  imitate a comparison.

The map keeps the layers that show real data: Complaints, Heatmap, Ward Zones and
Photo Gallery.

## 11. Public map no longer dies when the CDN fails

`initMap` called `L.maplibreGL(...)` without a guard, so if that CDN script failed
to load the whole map stayed blank. It now falls back to CARTO raster tiles.
