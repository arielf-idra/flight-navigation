# Instrument Scan Trainer

A training aid for instrument-flying scan discipline. The pilot keeps their eyes on
the attitude indicator (AI), which is always visible. One other instrument (ASI,
altimeter, turn coordinator, heading indicator, VSI or tachometer) is uncovered
for a brief flash, then covered again. The pilot reports what it showed. The aim
is the habit of glancing and returning to the AI, never fixating on another
instrument.

Live at https://arielf-idra.github.io/flight-navigation/scan-trainer/. This is a
single self-contained `index.html` (inline CSS and JS, no libraries, no build). It
is independent of the flight-navigation app at the repo root; see the root
`CLAUDE.md` for the shared-origin rule.

## How it works

- **Gauges** are inline SVG, drawn in a 200×200 viewBox. The angles in `pt()`
  are clockwise from 12 o'clock. Each `buildX(svg)` draws a dial and returns
  `update(value)`. `buildAI` returns `update(pitch, bank)`. The bank ring rolls
  with the horizon, and the orange index and miniature aircraft are fixed.
  - The markings (arcs, redlines) are **generic**, not any real aircraft's. Keep
    them that way unless the user supplies a specific aircraft's figures.
- **Turn coordinator** values are -2..2 in half-rate steps. Standard rate tilts
  the symbol 20°, onto the marks below the level marks.
- **Session loop** (`loop()`, requestAnimationFrame) runs a state machine:
  `wait` → `flash` (instrument uncovered for `cfg.flashMs`) → `mask` (150 ms
  pattern mask, so no after-image lingers) → `question` (multiple choice, keys
  1–5, timeout) → `feedback` → `wait`.
- **Questions**: `INSTR[key].q()` builds options with `numOpts()`: four
  consecutive values, spaced by the level's `LEVELS[level].sp`, that include the
  true value in a random position. Heading options wrap at 360.
- **Flying task**: the attitude drifts under a random disturbance (`TURB`
  off/light/moderate) plus a slight spiral tendency. The pilot corrects it with
  the arrow keys or WASD (Up = push, nose down) or by dragging on the AI. The
  results show "attitude held", the share of time within ±5° bank and ±3° pitch.
- **Storage**: `scanTrainer.settings` and `scanTrainer.history` (last 50
  sessions) in localStorage, read and written through `load()`/`save()` with
  try/catch.

## Testing

Use the same pattern as the root app. Serve the repo root (`python -m http.server
8123`), open `/scan-trainer/index.html` in Playwright and drive it. `S` (session
state), `cfg` and `gauges` are globals. Setting `cfg.count` small and
`S.nextAt = performance.now()` after Start makes a quick session. Check gauge
geometry by looking at a screenshot.
