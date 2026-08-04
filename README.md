# Maze Flow

Draw a continuous path from the green start circle to the red target without touching
the black walls. A single self-contained `index.html` — no build step, no CDN
dependencies, and no runtime styling or font loading.

## Running

Open `index.html` directly in a browser, or serve it locally:

```sh
python -m http.server 8000
# then visit http://localhost:8000
```

The game has no external assets: CSS and JavaScript are inline, fonts use the system
stack, and it works offline. Opening the file via `file://` is supported.

## Controls and rules

- **Start:** press inside the green start cell.
- **Draw:** keep the pointer (mouse, pen, or finger) down and move.
- **Collision:** touching a wall adds an error, plays a buzz, and stops the route.
- **Resume (Normal mode):** press within about one cell of the end of your path and
  the route continues; the reconnecting segment is validated so you cannot jump a wall.
- **Restart:** press in the green start cell at any time to restart from scratch
  (clears the path and resets the error count).
- **Strict mode:** any lift, cancel, focus loss, or interruption requires a restart
  from the green cell. The error count is only reset by an explicit restart or a new maze.
- **Win:** reach the red target circle. The win region matches the visible circle and
  uses segment-to-circle intersection, so a fast stroke cannot skip over the goal.
- **Timer:** starts when you begin a route and ticks in the top bar. It keeps running
  across lifts/resumes in Normal mode, and resets on any restart from the green cell,
  on Clear Path, and on New Maze. It stops when you reach the target, and the final
  time is shown in the win dialog.
- **New Maze / Clear Path / Sound:** bottom controls; the top menu sets difficulty and
  Strict mode.

## Supported input

The app uses the W3C Pointer Events model (`pointerdown`/`pointermove`/`pointerup`/
`pointercancel`/`lostpointercapture`) with pointer capture, an active-pointer identity
guard, and coalesced-event sampling where available. A legacy mouse/touch adapter is
used only when Pointer Events are unavailable; the two paths are never active together.

- Mouse (left button only; right-click and context menu are ignored).
- Stylus/pen (tip only; barrel buttons and eraser are ignored). Wacom and other
  tablets work through the unified Pointer Events stream.
- Single-touch (the primary touch only; extra fingers are ignored during a gesture).

No driver or Windows Ink configuration is required as a workaround. Windows Ink should
still be tested both on and off as part of the manual matrix below.

## Testing

### In-browser self-test

Open the page with the `selftest` query flag:

```
index.html?selftest
```

This runs the pure-logic unit assertions (coordinate conversion, swept collision,
target intersection, state transitions, start/resume/strict semantics, layout clamping,
deterministic maze validity) and renders a pass/fail overlay. Results are also written
to the console and `window.__SELFTEST_RESULT`.

### Node-based checks (no browser needed)

The pure functions (geometry, maze generation, state machine, layout) are extracted and
exercised in Node. See the `CURSOR_INPUT_AND_CODE_QUALITY_FIX_PLAN.md` remediation for
the full backlog these cover. A DOM smoke harness stubs the browser and drives
mouse/pen gestures through `init()`; it is part of the development tooling in the plan.

### Input diagnostics

Open the page with the `debug` query flag:

```
index.html?debug
```

A bottom-left panel shows a ring buffer of recent input events (type, `pointerType`,
`pointerId`, `isPrimary`, `button`, `buttons`, pressure, canvas coordinates, capture
state, and state-machine transitions). Use **Copy trace** to copy a sanitized trace for
support reports. No persistent device identifiers are collected; `pointerId` is
session-scoped only.

## Architecture

`index.html` is organized into sections: configuration constants, pure geometry, pure
maze generation, the state machine (`idle`/`drawing`/`paused`/`collided`/`won`), pure
layout, the input adapter, the two-layer renderer (cached static layer + dynamic layer),
lazy audio, UI/accessibility wiring, and bootstrap. Pure functions and read-only game
state are exposed as `window.MazeFlow` for testing.

Key implementation notes:

- **Coordinates:** input is converted from CSS pixels to logical game units using
  `canvas.width / rect.width` scale factors, so cursor, path, collision, and goal agree
  at any CSS scale, browser zoom, and device-pixel ratio.
- **Backing store:** the canvas backing store is scaled by `devicePixelRatio` (capped);
  all game geometry stays in logical units.
- **Collision:** swept segment tests against wall edges expanded by the path radius
  (matching the rendered 6px path), with a per-cell wall lookup instead of a full scan.
- **Rendering:** one `requestAnimationFrame` scheduler with a dirty flag; the maze is
  cached on an off-screen layer and only the path/collision layer is redrawn on change.
- **Resize:** a debounced `ResizeObserver` regenerates the maze once after a resize
  gesture and announces it; tiny/portrait/landscape/ultrawide viewports are clamped.
- **Audio:** `AudioContext` is created lazily after a user gesture; sound is optional
  and degrades gracefully when the API is unavailable.
- **Accessibility:** semantic controls, keyboard-operable menu, focus-managed dialog
  with Escape, `aria-live` announcements, `prefers-reduced-motion` support, and
  status that is never conveyed by color alone.

## Manual hardware test procedure (Wacom / tablets)

This is a release gate. For each supported browser (current Chrome, Edge, Firefox,
Safari) and Windows Ink setting (on and off), verify:

1. Hover: the in-app indicator is visible while the pen is over the maze and follows
   the tip without material lag.
2. Start gating: pen contact begins a route only in the green start cell.
3. Continuity: a route continues without interruption through brief movement outside
   the canvas while captured.
4. Collision accuracy at slow and fast speeds, matching the visible walls.
5. Leaving/re-entering the canvas, releasing outside, cancel, focus loss, tab hiding,
   and modal opening all leave the app safely not drawing.
6. Browser zoom and window resizing keep cursor, path, and targets aligned; resize
   regenerates once with a notice.
7. Strict mode requires a restart after every lift.
8. A 30-second continuous input soak stays responsive with stable cursor feedback.

## License / status

Personal project. See the remediation plan document for the audit and roadmap.
