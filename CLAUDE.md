# Flight Navigation Tool

A single-file (`index.html`, ~10,000 lines) vanilla JS + Leaflet.js flight-planning
web app for Israeli VFR/CVFR aviation. No build step, no framework, no package.json —
every data table and all JS logic live in `index.html`; the page's CSS lives in the
sibling `styles.css` (plain `<link rel="stylesheet">`, no build step involved). Base
maps are OpenStreetMap and Google Satellite/Hybrid/Streets — there is no calibrated
chart-image overlay anymore (that feature, and the two JPGs it used, were removed).

## Standing instructions

- **Commit and push as we go** (user instruction, Nov 2026; this replaces the old
  "don't push" rule). Once a change is verified, commit it as its own focused commit
  and `git push` to `main`. Every commit has gone straight to `main`. The repo is
  **public** and GitHub Pages serves `main` live at
  https://arielf-idra.github.io/flight-navigation/, so every push deploys
  immediately. Only push verified work, and never force-push.
- **Update the docs with every change**: this file, plus the in-app What's New
  changelog for anything user-visible (see "Versioning & What's New" below).
- **Don't hardcode real-world aviation/navigation data from memory** — coordinates,
  frequencies, runway headings, route connectivity, etc. This data is safety-critical.
  Only add it from a source the user supplies, or a public source you've found and the
  user has explicitly reviewed/approved. Every data table in this file has a comment
  block above it documenting where its data came from and what confidence/caveats
  apply — read that comment before trusting or extending the data, and write one in
  the same style when you add a new table.
- Prefer editing existing functions/tables over adding new files. There is no build
  step, so "add a script" always means "add a `<script>`-scoped function in
  `index.html`."

## Architecture

- **Global `appState` object** (~line 512) holds all mutable state: waypoints, every
  map layer's marker/line arrays, visibility flags, calibration data, etc.
- **Toggle-pattern for map layers**: each layer has `initX()` (builds markers/lines
  from that layer's data table, pushes them into an `appState.x` array) and
  `toggleX()` (adds/removes everything in that array from the map, flips
  `appState.xVisible`, updates the button's text/class). Search for `initCVFRRoutes`/
  `toggleCVFRRoutes` as a representative pair.
- **A layer being "always shown" means no toggle at all** — its `initX()` just adds
  every marker to the map directly and there's no `xVisible` flag or button
  (`initAirports`/`initPamatLandingGrounds` are the current examples). Don't invent a
  visibility flag for a layer the user said should always be on.
- **Every toggle needs three things kept in sync**, or it'll default wrong on load:
  1. The `appState.xVisible` default value.
  2. The button's *static* HTML (class `btn-primary`/`btn-secondary`, initial text) —
     this is what's visible before any JS runs, and it drifts out of sync easily.
  3. A "show X by default" block in the init sequence that actually adds the markers
     to the map when `xVisible` is true on load (a flag alone does nothing — several
     real bugs this session were exactly this: flag was `true`, nothing rendered,
     because this block didn't exist yet).
- **Watch for duplicate function declarations.** The old top-level duplicates
  (`initAirspaceZones`, `toggleAirspace`, `showPreview`, `buildColorSwatches`) have
  since been removed, and as of Nov 2026 every top-level function name is unique.
  Names that repeat (`mm`/`mu`, `mmR`/`muR`, `showSelection`/`hideSelection`,
  `onMove`/`onUp`, `destPoint`, `cleanup`) are local helpers nested inside other
  functions, so they don't shadow each other. Non-strict JS lets a later top-level
  declaration silently win, so if an edit doesn't seem to take effect, grep for
  `function name` first, and don't introduce a new top-level duplicate.
- **Coordinate systems**: `dCoord(wp)` / `toDisplayLatLng()` / `calibrateDisplayToReal()`
  are kept as identity pass-throughs (display coords == real coords now that the
  calibrated-chart-overlay feature is gone) rather than inlined, because dozens of
  call sites throughout the app still route every marker/polyline position through
  these names. Any new polyline/marker geometry should still go through them (cheap
  today, and it's the one seam that would need touching again if a calibrated overlay
  ever came back), and should still be repositioned via `repositionMarkerList()` (or
  equivalent) in the base-map-change handler.
- **Rotation convention**: CSS `transform: rotate(0deg)` draws an element in its
  authored orientation, which for a horizontally-drawn line/glyph corresponds to a
  compass bearing of 90° (east), not 0°. The correct mapping is
  `cssRotation = bearing - 90`. This bug (using the raw bearing directly) has bitten
  multiple features in this codebase — check any new rotated icon/label against this.
- **`var`-in-closure bug**: event handlers built inside a loop must reference
  `e.target`, not the outer loop variable — capturing the loop variable directly
  binds every handler to the *last* iteration's value. Caused a real multi-leg bug
  here before.
- **Dashboard tabs**: the sidebar is a tab dashboard, not a long scroll — `.dashboard-tabs`
  holds one button per tab (`onclick="switchDashboardTab('name')"`), and every `.panel`
  in the sidebar (except the header) carries a `data-tab="name"` attribute matching one
  of those tabs; several panels can share the same tab. `switchDashboardTab()` toggles
  a `.tab-active` class (CSS: `.panel[data-tab] { display:none }` /
  `.panel[data-tab].tab-active { display:block }`) and persists the active tab to
  `localStorage`. Adding a new sidebar panel means giving it a `data-tab` matching an
  existing tab button (or adding a new tab button) — a panel with no `data-tab` never
  shows.

## Data tables (all near the top of the file, before the `<script>` logic)

Each has a comment block directly above it — read that for provenance/caveats before
touching it. Brief map:

- `cvfrPointsData` — CVFR reporting points, from the official AIP CSV.
- `vorPointsData` — VOR/VOR-DME stations, user-supplied.
- `cvfrRoutesData` — CVFR route lines between report points. **This one has a long,
  contentious history** — read its comment block fully before changing the extraction
  method again. Several automated approaches (shapefile polygon bisection, tracing
  the AIP PDF's own vector paths, tracing rendered PDF pixels) were each tried and
  rejected as inaccurate. The current approach resolves a third-party reference
  site's route *topology* against this app's own current `cvfrPointsData` positions,
  with per-route heading/distance/one-way-arrow labels precomputed once (not
  recalculated on every page load) and stored directly on each route object
  (`distanceNm`, `startHeading`, `endHeading`, `lineSegments`, etc.) — see
  `scratchpad/cvfr_ref_site/precompute_route_stats2.js`-style scripts if regenerating.
  Known incomplete/pending manual verification against the current chart.
- `airportsData` — the 5 civil airports (Ben Gurion, Ramon, Haifa, Herzliya, Rosh
  Pina), each with full runway data (supports multiple runways). Always shown, no
  toggle.
- `pamatLandingGroundsData` — the 12 official PAMAT "landing grounds". Always shown,
  no toggle.
- `airstripsData` — everything else (small/private strips). Toggle-controlled
  ("Show/Hide Airstrips"), default off.
- Airport/airstrip/landing-ground `heading` fields (runway direction) come from a
  user-supplied PAMAT/AIP-sourced JSON that explicitly self-describes as
  "reference dataset only... do not use as a substitute for current PAMAT/AIP/NOTAM."
  Most entries still have no heading at all and fall back to a fixed 90°.
- The LSA/AZM reporting-points layer (`lsaPointsData`, 205 points) was removed
  entirely in Nov 2026 at the user's request. Don't re-add it. CVFR point
  rotation, which used to share LSA's rotation function, now lives in
  `refreshCVFRPointRotation()`.
- `frequenciesData` — map labels for radio frequencies (Hebrew name plus frequency at
  a lat/lng). **It has no provenance comment block**, so its source is unknown.
  Don't extend or trust it until the user confirms where it came from, then add a
  comment block. Toggle-controlled (`frequenciesVisible`, default off).

## Route timing, labels & fuel

- **`legSeconds(i)` is the single source of truth for a leg's duration.** It uses
  cruise speed, or, if the leg has a climb (`appState.legClimbs[i]`, set from the
  leg popup), climb ground speed up to top-of-climb and then cruise. Every time
  readout (leg boxes, waypoint labels, totals, prints, fuel) goes through it, so
  don't reintroduce inline `distance / cruiseSpeed`. No wind model.
  - For a point *partway* along a leg, use `secondsAlongLeg(i, dNm)` (time to
    fly the first dNm, climb-aware) and its inverse `distanceAlongLegAt(i, secs)`.
    `legSeconds` is built on them. Ident-pin times and committed-leg minute ticks
    use them, so ticks bunch up during a climb.
  - Cruise speed must pass `validCruiseSpeed()` (finite and > 0). A speed of 0
    once made leg times Infinity, and `drawMinuteTicks`' loop hung the tab.
    Input, settings and plan restore all go through the check.
- **Per-waypoint/per-leg state is keyed by `wp.num` (or `"numA_numB"`) or by leg
  index.** That covers `waypointLabelOffsets`/`Styles`, `miniWaypoints`,
  `legBoxHidden`, `legLabelDraggedPositions`, `legAltitudes` and `legClimbs`.
  Anything that renumbers or reorders waypoints must remap all of these: delete,
  Split Leg, list reorder (`reorderWaypoint`) and plan load (`_finishApplyPlan`)
  do. `clearRoute()` resets them and removes every route-related layer (wind
  tables, TOC markers, ±45° guides included). Add any new keyed field to all of
  those places.
- **Waypoint labels** show the leg time, then the accumulated time in brackets,
  e.g. `06:14 (13:08)`. `computeAccumulatedTimeMap()` builds the accumulated
  values. The brackets are controlled by `appState.accumTimeVisible` ("Hide/Show
  Accum. Time" in Layers, default on). Like the other label toggles (distance,
  drift lines, ±45° guides), it is **not** persisted and resets on load.
- **Fuel** = `legSeconds` × fuel flow. Taxi, alternate and climb fuel-flow
  differences aren't modelled. The route printout adds a fixed 30-minute reserve.
  - `appState.fuelFlow` is **always stored in L/hr**, whatever unit is shown.
    `appState.fuelUnit` (`'gal'` = US gallons, the default, or `'L'`) only affects
    input and display. Convert with `fuelFromLitres()` / `fuelToLitres()`
    (1 US gal = 3.785411784 L). Format with `fuelUnitShort()`, `fuelUnitLong()`
    and `formatFuelFlow()`. `syncFuelFlowInput()` pushes state into the Data-tab
    input, selector and label. Never write `fuelFlow` into the input directly.
  - `fuelUnit` is saved in settings and **with each flight plan**. A plan with no
    `fuelUnit` predates the option ("the liters days") and loads in litres. Its
    `fuelFlow` was L/hr all along, so its value is unchanged.

## Versioning & What's New

`APP_NAME` / `APP_VERSION` (~line 612) drive the title, header, watermark, GPX
export and Contact form. The `<title>` tag and the header's static HTML carry a
hardcoded copy that must be bumped too. The What's New modal
(`showWhatsNewIfNeeded()`) is **only opened from the About tab's "🆕 What's New"
button** (`force=true`, which lists every `CHANGELOG` entry grouped by version).
Nothing calls it on page load, and that has been true for the repo's whole
history. Its "show only entries newer than `flightNavToolLastSeenVersion`" branch
is therefore dormant. To document a user-visible change: add a `CHANGELOG` entry
for the current version, or bump `APP_VERSION` (plus the two static copies) and
append the new version to `versionOrder`.

## Removed features

- **NOTAMs** (tab, map overlay, notam-isr fetch, filters, settings and plan fields)
  were removed entirely in Nov 2026 at the user's request. Don't re-add them.
  Old settings files and plans may still carry `notam*` keys, which are silently
  ignored. A remembered active tab of `notams` falls back to Route
  (`switchDashboardTab()` falls back for any unknown tab).
- **LSA/AZM reporting points**: see the Data tables section.
- **"Show Airspace" layer** (`initAirspaceZones`/`toggleAirspace`) removed in Nov 2026
  at the user's request. It was 7 hardcoded circles with made-up radii and no
  provenance, including Sde Dov CTR (closed 2019). If airspace is ever wanted again,
  it needs real data from a source the user supplies (see the data-integrity rule).
- **"Time Label Interval" slider** (`labelInterval`) removed in Nov 2026. It never
  did anything, and the user wants minute labels fixed on even minutes
  (`drawMinuteTicks`). Old settings and plans with `labelInterval` are ignored.
  "Time Tick Interval" (`tickInterval`) is a different setting and still works.

## Testing workflow

There's no test suite. The established pattern for verifying any change:

1. **Syntax check** both `<script>` blocks with Node before anything else:
   ```js
   const html = fs.readFileSync('index.html', 'utf8');
   [...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].forEach(m => new Function(m[1]));
   ```
2. **Serve locally** (a static file server rooted at the project dir; port 8123 has
   been the convention) and drive it with **Playwright** (chromium) to click
   buttons/toggles, inspect `appState`, and screenshot the result. (The "What's
   New" modal no longer opens on load; it only opens from the About tab, so no
   dismissal step is needed.) Note: some toggle states persist across page loads (localStorage), so a fresh
   Playwright browser context isn't always a true "first visit" — check
   `appState.xVisible` directly rather than assuming a load-time default.
3. For anything geometry/coordinate related, actually look at a screenshot (or
   compare against a real chart crop) rather than trusting the numbers alone —
   several bugs this session (90°-off rotation, wrong-panel transform, etc.) were
   only caught by eyeballing rendered output.

Practical gotchas:
- **`index.html` has mixed line endings**: mostly CRLF, but some HTML blocks are
  LF, and a few lines have trailing spaces. Scripted find/replace must accept
  either `\n` or `\r\n` and not assume trailing whitespace is absent.
- **Playwright on this machine**: there's no project `package.json`. Install
  `playwright` into the session scratchpad and launch with
  `executablePath` pointed at the already-downloaded
  `~/AppData/Local/ms-playwright/chromium-*/chrome-win64/chrome.exe`. The npm
  package's pinned browser revision usually isn't downloaded. `addWaypoint(lat, lng)`
  takes two numbers, not an `L.latLng`.
- One console 404 on load is pre-existing (it's not an app fetch, most likely the
  favicon), so ignore it when checking for regressions.

## Known gaps / follow-ups

- `cvfrRoutesData` needs a manual pass against the current official chart to add
  missing routes and correct/remove stale ones (the automated topology source is
  known incomplete — see its comment block).
- Most `airstripsData`/`pamatLandingGroundsData` entries have no runway heading yet.
- A saved flight plan or `waypoints.json` group from before the chart-overlay removal
  can still reference `"Air Nav Chart"`/`"AZM Chart"` as a map name. `_finishApplyPlan()`
  falls back to OpenStreetMap so loading such a plan doesn't crash; a waypoints-file
  group under one of those names just displays as an orphaned tab with nothing behind
  it — cosmetic only, not worth a fix unless it comes up. A user's own leftover
  `calibration_Air_Nav_Chart.json`/`calibration_AZM_Chart.json` files in their working
  folder are similarly harmless orphaned data.
