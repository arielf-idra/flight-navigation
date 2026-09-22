# Flight Navigation Tool

A single-file (`index.html`, ~10,000 lines) vanilla JS + Leaflet.js flight-planning
web app for Israeli VFR/CVFR aviation. No build step, no framework, no package.json —
every data table and all JS logic live in `index.html`; the page's CSS lives in the
sibling `styles.css` (plain `<link rel="stylesheet">`, no build step involved). Base
maps are OpenStreetMap and Google Satellite/Hybrid/Streets — there is no calibrated
chart-image overlay anymore (that feature, and the two JPGs it used, were removed).

## Standing instructions

- **Don't push.** Commit locally only when explicitly asked; never `git push` unless
  told to.
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

- **Global `appState` object** (~line 540) holds all mutable state: waypoints, every
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
- **Duplicate function declarations exist in this file** (e.g. `initAirspaceZones`,
  `toggleAirspace`, `showPreview`, `buildColorSwatches` are each declared twice).
  Non-strict JS lets the later declaration silently win. If you're editing one of
  these and changes don't seem to take effect, grep for the function name first —
  you may be editing the one that gets shadowed.
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

## NOTAM overlay

Live Israeli NOTAMs, plotted on the map and listed in a collapsible sidebar section
("NOTAMs" — the toggle/refresh/status/list are always visible; everything
configuration-related lives behind the "⚙️ NOTAM Filters" collapsible, same UI
pattern as Display Settings).

- **Data source**: a companion project,
  [notam-isr](https://github.com/arielf-idra/notam-isr), scrapes Israel Airports
  Authority's live NOTAM page server-side (a real headless browser clears its
  bot-management challenge, then clicks each row's "+" to capture the *full* ICAO
  text via the AJAX response rather than the truncated preview the page shows by
  default) and republishes the result as a static `notams.json` every ~10 minutes.
  `fetchNotamsFromUrl()` tries `arielf-idra.github.io/notam-isr/notams.json` first,
  falling back to the `raw.githubusercontent.com` copy — the latter is a known
  target for corporate/network firewalls to block outright, confirmed by testing,
  so Pages is primary. NOTAMs are fetched proactively on page load (`fetchNotams()`
  in the init sequence) since it's a small CDN-served file; the map layer itself
  still stays off by default (`notamsVisible: false`).
- **Every NOTAM carries a `position`** (lat/lon/radiusNm) tagged `source: "e_text"`
  (precise, parsed from a "CENTERED ON PSN ... RADIUS" phrase) or `"q_line"` (the
  ICAO Q-line's mandatory summary position/radius, precise only to the nearest
  arc-minute — every NOTAM has one per Annex 15, even multi-point boundary NOTAMs
  whose vertex list can't safely be reduced to one shape). Q-line-sourced circles
  render dashed specifically so they're never mistaken for an exact affected area;
  some (e.g. FIR-wide restrictions) can have an 80+ NM radius — this is the Q-line's
  generic locator, not a real depiction of the shape, especially for NOTAMs that are
  actually about a border/linear corridor rather than a circular area.
- **Multi-vertex boundary NOTAMs (`"BTN [THE] FLW PSN(S) ..."`) are never drawn as
  polygons** — not even as an approximate centroid. Confirmed directly during
  development: the source page silently truncates NOTAM text with no ellipsis, so a
  vertex list that ends cleanly can still be missing real vertices. Only the Q-line
  fallback circle is shown for these.
- **Relevance filtering** (`getFilteredNotams()`) hides NOTAMs by default rather than
  just sorting them (there's no sort — removed once filtering covered the same need):
  a NOTAM is shown if it's within the (configurable) route corridor width, OR at/near
  the selected takeoff/landing airport (exact ICAO match — PAMAT landing grounds are
  deliberately excluded from those two dropdowns since this app has no verified ICAO
  code for them, see the data-integrity rule above), AND its Q-line `traffic` field
  matches the selected flight rules (VFR hides IFR-only, IFR hides VFR-only, All
  disables this check) AND its Q-line `purpose` qualifier contains `B` (selected for
  PIB) unless the purpose filter is set to All. An unparseable/missing field (Q-line
  altitude, purpose) is never treated as grounds for exclusion — only a confidently
  *known* mismatch hides a NOTAM. "Show all ISR NOTAMs" bypasses all of the above.
- **Settings persistence**: `notamOpacity`, `notamFlightDate`, `notamShowAll`,
  `notamCorridorWidth`, `notamFlightRules`, `notamPurposeFilter` live in
  `DEFAULT_SETTINGS`/`collectSettings()`/`applySettings()` like any other display
  setting. `notamTakeoffSelect`/`notamLandingSelect` additionally save *with the
  flight plan* (`planToJson()`/`_finishApplyPlan()`), since they're route-specific.
- **`updateRoute()` debounces the NOTAM list refresh** (`scheduleNotamPanelRefresh()`,
  300ms) rather than calling `updateNotamPanel()` directly — `updateRoute()` fires
  continuously during a waypoint drag and in a tight loop while a saved plan's
  waypoints load; recomputing distance-to-route for ~90 NOTAMs on every one of those
  calls caused visible jank before this was added.

## Testing workflow

There's no test suite. The established pattern for verifying any change:

1. **Syntax check** both `<script>` blocks with Node before anything else:
   ```js
   const html = fs.readFileSync('index.html', 'utf8');
   [...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].forEach(m => new Function(m[1]));
   ```
2. **Serve locally** (a static file server rooted at the project dir; port 8123 has
   been the convention) and drive it with **Playwright** (chromium) to click
   buttons/toggles, inspect `appState`, and screenshot the result. A "What's New"
   modal (`#whatsNewContinue`) blocks interaction on first load — dismiss it first.
   Note: some toggle states persist across page loads (localStorage), so a fresh
   Playwright browser context isn't always a true "first visit" — check
   `appState.xVisible` directly rather than assuming a load-time default.
3. For anything geometry/coordinate related, actually look at a screenshot (or
   compare against a real chart crop) rather than trusting the numbers alone —
   several bugs this session (90°-off rotation, wrong-panel transform, etc.) were
   only caught by eyeballing rendered output.

## Known gaps / follow-ups

- `cvfrRoutesData` needs a manual pass against the current official chart to add
  missing routes and correct/remove stale ones (the automated topology source is
  known incomplete — see its comment block).
- Most `airstripsData`/`pamatLandingGroundsData` entries have no runway heading yet.
- `pamatLandingGroundsData` entries have no `icao` field, so they don't appear in the
  NOTAM takeoff/landing dropdowns (only real `LL`-prefix airports do). If verified
  ICAO/PAMAT codes for specific landing grounds are supplied, add an `icao` field to
  the relevant `pamatLandingGroundsData` entries and they'll appear automatically —
  don't guess codes from memory (see the data-integrity rule above).
- A saved flight plan or `waypoints.json` group from before the chart-overlay removal
  can still reference `"Air Nav Chart"`/`"AZM Chart"` as a map name. `_finishApplyPlan()`
  falls back to OpenStreetMap so loading such a plan doesn't crash; a waypoints-file
  group under one of those names just displays as an orphaned tab with nothing behind
  it — cosmetic only, not worth a fix unless it comes up. A user's own leftover
  `calibration_Air_Nav_Chart.json`/`calibration_AZM_Chart.json` files in their working
  folder are similarly harmless orphaned data.
