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
  - The panel follows the **Cessna 172S analog panel**. The markings come from
    the 172S POH: ASI arcs, the stepped tach green arc and red line, the AI bank
    marks, and the layout with the tach below the VSI. The comment block above
    `buildASI` lists every figure, its POH section, and the figures that are
    generic (dial ranges, ALT/TC faces). Don't change those figures without a
    source the user approves.
- **Look**: the panel is styled after the C172 panel art in those FAA figures.
  The panel is grey metal with a glareshield. `caseAndBezel()` draws the corner
  screws, the chrome bezel and a shadow. The AI has yellow symbols and a black
  lower case with its knob. The altimeter has a barber-pole window. The shared
  gradients sit in one hidden `<svg>` at the top of `<body>`.
- **Turn coordinator** values are -2..2 in half-rate steps. Standard rate tilts
  the symbol 20°, onto the marks below the level marks.
- **Session loop** (`loop()`, requestAnimationFrame) runs a state machine:
  `wait` → `flash` (instrument uncovered for `cfg.flashMs`) → `mask` (150 ms
  pattern mask, so no after-image lingers). The default flash is 1000 ms
  (user request). Choosing a level in Settings still presets its own flash
  (Easy 900, Medium 500, Hard 250 ms), and the slider can override it. → `question` (multiple choice, keys
  1–5, timeout) → `feedback` → `wait`.
- **Questions**: `INSTR[key].q()` builds options with `numOpts()`: four
  consecutive values, spaced by the level's `LEVELS[level].sp`, that include the
  true value in a random position. Heading options wrap at 360.
- **Flying task**: the attitude drifts under a random disturbance (`TURB`
  off/light/moderate) plus a slight spiral tendency. The pilot corrects it with
  the arrow keys or WASD (Up = push, nose down), by dragging on the AI, or by tilting the phone. The
  results show "attitude held", the share of time within ±5° bank and ±3° pitch.
- **Screens**: only one overlay at a time, through `show(id)` (`startOverlay`,
  `settingsOverlay`, `pauseOverlay`, `resultsOverlay`, or `null` while flying).
  The page loads on the Start screen, with History and Settings at the bottom.
  Settings are saved only on Done (`readForm()`). During a run the header has
  Pause and Finish. Esc or P pauses, and Space or Esc resumes. Leaving the tab
  pauses too.
  - **Pause** shifts every timer in `TIMERS` by the paused time and covers a
    flashing instrument until Resume.
  - **Finish** with answers shows the results; with none it returns to Start.
- **Mistake review** (Easy and Medium only): after a wrong answer or a timeout,
  `showReview()` draws the instrument large at its true value. It shows the
  right answer and the pilot's answer, plus a "how to read it" line from
  `READ_HINTS`. The session stays paused (`pause('reviewOverlay')`) until any
  tap or key. Taps in the first 350 ms are ignored, so the tap that answered
  doesn't close it. Hard shows no review.
- **Scan techniques** (`SCANS`, `nextKey()`) come from the FAA Instrument Flying
  Handbook (FAA-H-8083-15B, ch. 6, p. 6-11, Figs. 6-17 to 6-19). The labels are
  the FAA names.
  - **Selected radial** (the default): random flight instruments, chosen in
    Settings.
  - **Inverted-V**: TC and VSI alternate.
  - **Rectangular**: counter-clockwise by default, matching the FAA's Fig. 6-19
    (ASI → TC → HI → VSI → ALT), or clockwise.
  - `scanFigure()` shows the FAA's own figure for each technique:
    `img/fig6-17.jpg`, `fig6-18.jpg` and `fig6-19.jpg`. They were cropped
    from the handbook PDF and are public domain, as a US government work. The
    figure appears on the Start screen and in Settings, in place of a text
    description.
  - The AI stays visible in every mode, because the flying task needs it.
- **Tilt control** (phones only, `TILT_OK`): `deviceorientation` beta/gamma
  becomes the "up" vector in screen axes, using `screen.orientation.angle` so
  landscape works.
  - It is **attitude control**, not rate control, as the user asked ("use the
    gyros to keep the attitude bird centered"). The phone's angle from neutral
    is added 1:1 to the attitude (`tiltOffset()`). Turning the phone like a
    wheel banks; tilting the top toward you raises the nose.
  - Turbulence and the keys/drag move `S.db`/`S.dp`. The shown attitude is
    `S.db + phone offset`. With tilt on, `S.db`/`S.dp` ease back slowly, so
    the phone never has to stay at an extreme angle.
  - Neutral is the phone position at session start, and Center resets it.
  - iOS (Safari, and Chrome, which uses the same engine) needs
    `DeviceOrientationEvent.requestPermission()`. It has to be called
    synchronously inside the Start tap or keypress, so keep `startTilt()`
    before any `await` or timer.
  - A screen wake lock is held while flying.
  - Tilt is **on by default on phones**. The setting is `undefined` until the
    user saves Settings.
  - The Start screen has a "Tilt to fly: On/Off" toggle. When turned on, the
    toggle asks for motion access straight away (`requestMotion()`).
  - There are no sensor readouts: the live readout and the header "Tilt ✓"
    chip were removed once the user confirmed the gyros work. Only an
    actionable "motion access was denied" message remains, under the toggle.
  - iOS remembers a denial for the rest of the tab session. The user has to
    reopen the page to be asked again.
- **Storage**: `scanTrainer.settings` and `scanTrainer.history` (last 50
  sessions) in localStorage, read and written through `load()`/`save()` with
  try/catch.

## Testing

Use the same pattern as the root app. Serve the repo root (`python -m http.server
8123`), open `/scan-trainer/index.html` in Playwright and drive it. `S` (session
state), `cfg` and `gauges` are globals. Setting `cfg.count` small and
`S.nextAt = performance.now()` after Start makes a quick session. Check gauge
geometry by looking at a screenshot.
