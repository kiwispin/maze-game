# Maze Flow Cursor, Input, and Reliability Remediation Plan

## 1. Purpose and scope

This document is an implementation plan only. It does not change the game code.

The review covers the repository at commit [`2b17e6199b77f0c033a541b6f2d896c89080cd5e`](https://github.com/kiwispin/maze-game/commit/2b17e6199b77f0c033a541b6f2d896c89080cd5e) on `main`. The primary goal is to make the cursor and drawing interaction reliable for mouse, touch, Wacom Bamboo pen/tablet configurations, and other styluses. The secondary goal is to correct the reliability, performance, accessibility, and maintainability problems found during the same audit.

No GitHub issues or pull requests were present at the time of review, and the repository contains only `index.html` plus a minimal `README.md`. There is currently no automated test, build, lint, or deployment configuration.

## 2. Executive diagnosis

The reported cursor problem is unlikely to have a single cause. The code contains one confirmed visual regression and several input-lifecycle defects that together explain why the failure can vary by browser, tablet driver, and user action.

### Confirmed findings

1. **The explicit canvas crosshair was removed.** The original canvas CSS included `cursor: crosshair`; commit [`5bf45a4`](https://github.com/kiwispin/maze-game/commit/5bf45a4e06ca1d4d28b06580ad2a28a446439001) removed it during the layout rewrite. The [current canvas rule](https://github.com/kiwispin/maze-game/blob/2b17e6199b77f0c033a541b6f2d896c89080cd5e/index.html#L125-L131) sets no cursor at all. This is a definite regression in cursor affordance, although it does not by itself explain every report of the operating-system cursor vanishing.

2. **The app does not use the hardware-neutral Pointer Events model.** It registers separate legacy mouse and touch listeners only ([current listeners](https://github.com/kiwispin/maze-game/blob/2b17e6199b77f0c033a541b6f2d896c89080cd5e/index.html#L947-L961)). A Wacom pen may be exposed as `pointerType: "pen"`; relying on compatibility mouse events is driver- and browser-sensitive. The W3C Pointer Events standard is specifically designed to unify mouse, pen, and touch input and defines pointer capture and cancellation behavior: [W3C Pointer Events](https://www.w3.org/TR/pointerevents/).

3. **The gesture is not captured.** `mouseup` and `mouseout` are attached only to the canvas. There is no `setPointerCapture`, `pointercancel`, `lostpointercapture`, window-blur, or page-visibility cleanup. A pen leaving the canvas, driver jitter at the edge, a browser gesture, or release outside the element can therefore drop or strand the drawing session.

4. **There is no application-rendered hover cursor.** The canvas renders the path only after a valid press. If a Wacom/OS/browser combination does not display a system cursor, the app provides no independent indication of pen position or whether the pen is in range.

5. **Event coordinates are wrong when the canvas is CSS-scaled.** `getPos` only subtracts the canvas rectangle origin ([coordinate code](https://github.com/kiwispin/maze-game/blob/2b17e6199b77f0c033a541b6f2d896c89080cd5e/index.html#L690-L696)); it does not convert from CSS pixels to canvas coordinates. The canvas has CSS maximum dimensions, so scaling can occur. This can offset the visible path and hit testing enough that users describe the cursor/path as missing or detached from the pen.

6. **High-frequency stylus input can overwhelm rendering.** Every accepted move appends a point, scans every wall, and schedules another animation-frame callback. The entire maze and entire accumulated path are then redrawn. Wacom hardware can report dense movement samples, making dropped frames or an apparently frozen/disappearing trail more likely.

7. **An old Wacom/browser interaction is a relevant test variable, not an acceptable product workaround.** Wacom has documented Chrome click/lag behavior affected by Windows Ink settings ([Wacom support note](https://support.wacom.com/hc/en-us/articles/1500006273281-Why-is-Google-Chrome-not-accepting-clicks-or-is-lagging)). The application should be tested with Windows Ink both enabled and disabled, but users should not be required to change a global driver setting to use the game.

### Root-cause ranking

| Confidence | Cause | Why it matters |
| --- | --- | --- |
| Confirmed | Crosshair CSS removed | Directly regressed the intended cursor affordance. |
| High | Legacy mouse/touch listeners instead of Pointer Events | Pen streams and compatibility mouse events vary across Wacom driver/browser combinations. |
| High | No pointer capture or cancellation cleanup | Explains a cursor/path that works initially and then stops after leaving the canvas or losing focus. |
| High | Incorrect CSS-to-canvas coordinate conversion | Explains offset, invisible, or seemingly disconnected drawing on scaled canvases and zoomed displays. |
| Medium-high | Rendering and collision work on every dense pen sample | Explains lag and visual dropouts more strongly on high-rate tablet input. |
| Medium | Wacom driver/browser configuration | May amplify the problem, but should be treated as a compatibility condition rather than the primary fix. |

## 3. Target interaction contract

Before implementation, define and test one explicit behavior contract:

- Mouse, pen, and single-touch input use the same state machine.
- A mouse or hover-capable pen shows an obvious cursor while it is over the maze. Touch does not leave a phantom hover cursor.
- Only the primary pointer and the intended primary contact/button can start a route. Pen barrel buttons, erasers, right-click, and secondary touches do not accidentally start a route unless deliberately supported later.
- A route starts only in the start zone.
- The active pointer remains authoritative until it ends or is cancelled; other fingers or devices are ignored during that gesture.
- Moving outside the canvas cannot leave the game stuck in drawing mode.
- Normal and Strict modes have written, non-exploitable resume/restart rules.
- The drawn path, hover marker, collision geometry, and goal geometry agree visually at every CSS scale, browser zoom level, and device-pixel ratio.
- The game never requires a user to disable Windows Ink or change a system setting. Such configurations are test cases only.

## 4. Implementation phases

### Phase 0 — Reproduce and instrument before changing behavior

1. Record a reproducibility matrix containing OS version, browser/version, Wacom Bamboo model, Wacom driver version, Windows Ink state, browser zoom, display scaling, connection type, and exact failure sequence.
2. Add a development-only on-screen input diagnostics panel, disabled in production by default. It should keep a short in-memory ring buffer of event type, `pointerType`, `pointerId`, `isPrimary`, `button`, `buttons`, pressure, canvas-relative coordinates, capture state, and state-machine transition.
3. Do not collect persistent device identifiers, raw traces, or other fingerprinting data. Allow a user/tester to copy a short sanitized trace only when they choose to do so.
4. Reproduce these distinct symptoms separately:
   - operating-system cursor not visible on hover;
   - hover cursor visible but path does not begin;
   - path begins and then stops mid-gesture;
   - path is offset from the stylus;
   - path freezes or visibly lags;
   - drawing remains active after release or focus loss.
5. Capture a baseline performance profile for a 30-second continuous Wacom trace on Hard difficulty: event rate, maximum points, collision time, render time, long tasks, and frame rate.

**Exit criterion:** each reported symptom is mapped to an event trace or clearly marked as not yet reproduced. Avoid treating all “cursor disappeared” reports as the same defect.

### Phase 1 — Replace input handling and restore cursor visibility (P0)

1. Replace the parallel mouse/touch handlers with `pointerdown`, `pointermove`, `pointerup`, `pointercancel`, and `lostpointercapture`. Feature-detect Pointer Events and keep a small legacy adapter only if the supported-browser policy genuinely requires it; never run both paths for the same interaction.
2. Track one `activePointerId`. Accept the primary mouse button, pen tip, or primary touch only. Ignore unrelated pointers until the active interaction ends.
3. On an accepted `pointerdown`, call `setPointerCapture(pointerId)`. Release capture on normal completion and funnel `pointerup`, `pointercancel`, `lostpointercapture`, window blur, and hidden-page transitions through one idempotent cleanup routine.
4. Keep `touch-action: none` on the interactive canvas, not the entire `body`, so the browser delivers the game’s pointer stream without unnecessarily disabling normal behavior everywhere else.
5. Restore `cursor: crosshair` as the immediate system-cursor fallback.
6. Add a lightweight DOM-based hover/contact indicator positioned above the canvas with `pointer-events: none`. Show it for mouse and pen `pointerenter`/`pointermove`, hide it on `pointerleave`, cancellation, modal display, and for touch. Differentiate valid start, drawing, paused/invalid, and collision states without relying on color alone.
7. Do **not** set `cursor: none` by default. A custom cursor should supplement the system cursor until real-device testing proves hiding the system cursor is desirable and reliable.
8. Centralize event normalization so game logic receives a stable sample object rather than raw browser events.
9. Where available, feature-detect `getCoalescedEvents()` and process its chronological samples once rather than processing both the parent and coalesced samples. The API can improve path fidelity but is not universally available and may require a secure context, so correctness must not depend on it: [MDN compatibility note](https://developer.mozilla.org/en-US/docs/Web/API/PointerEvent/getCoalescedEvents), [W3C coalesced-event behavior](https://www.w3.org/TR/pointerevents/#coalesced-events).

**Exit criteria:**

- Mouse, touch, and pen can all start, continue, and finish a route through the same state machine.
- A Wacom pen has a visible in-app position indicator during hover and contact even if the system cursor is unavailable.
- Leaving the canvas and returning while captured does not drop the active route.
- Releasing outside, cancelling, switching tabs, or losing focus never leaves `isDrawing`/its replacement active.
- No duplicate path segments are created by compatibility mouse events.

### Phase 2 — Make coordinates, collision, goal detection, and mode rules correct (P1)

1. Replace `clientX || touches[0].clientX` with normalized Pointer Event coordinates. The current truthy fallback also mishandles a legitimate coordinate of zero.
2. Convert CSS coordinates to logical game coordinates with independent X and Y scale factors derived from `canvas.width / rect.width` and `canvas.height / rect.height`.
3. Separate logical maze size from backing-store size. Scale the backing store for `devicePixelRatio`, cap the ratio where necessary for memory/performance, and keep game geometry in CSS/logical units.
4. Clamp or reject samples outside the intended playfield. Define edge behavior explicitly rather than allowing an off-canvas sample to win or collide unpredictably.
5. Replace `lineIntersectsRect`, which currently reports any segment bounding-box overlap as a collision ([current implementation](https://github.com/kiwispin/maze-game/blob/2b17e6199b77f0c033a541b6f2d896c89080cd5e/index.html#L512-L525)), with a true swept-segment collision test against wall geometry expanded by a defined player/path radius. This prevents diagonal false positives and tunnelling at fast movement speeds.
6. Check all chronological movement samples, including coalesced samples when supported, and stop processing immediately after the first collision or win transition.
7. Define the win region from the visible target circle (with an intentional tolerance), not simply `x > endCell.x && y > endCell.y` ([current condition](https://github.com/kiwispin/maze-game/blob/2b17e6199b77f0c033a541b6f2d896c89080cd5e/index.html#L793-L797)). Use segment-to-target intersection so a fast sample cannot jump over the goal.
8. Replace the loose booleans with named states such as `idle`, `drawing`, `paused`, `collided`, and `won`, with a transition table.
9. Resolve resume semantics:
   - Normal mode should resume only from the actual last valid point and should validate the reconnecting segment; it must not teleport across a wall using the current `null` path break.
   - Starting again in the start zone should be an explicit restart action, with a clearly defined error-count policy.
   - Strict mode should make any lift/cancel require a restart, but should not silently reset the score unless that is the documented rule.

**Exit criteria:** a test point under the visible cursor maps to the same logical point at 100%, 125%, 150%, and 200% display scaling, multiple browser zoom values, CSS-scaled canvas sizes, and DPR 1/2/3. Collision and win tests agree with the rendered geometry and cannot be bypassed by sparse or resumed input.

### Phase 3 — Stabilize rendering and stylus performance (P1)

1. Use one animation-frame scheduler with a `renderPending`/dirty flag. Multiple input events in one frame must schedule only one render.
2. Separate static maze rendering from dynamic path/cursor rendering. Cache the background, goals, and walls in an off-screen canvas or static layer and redraw them only when the maze/layout changes.
3. Avoid rescanning every wall for every sample. Prefer collision against the current/adjacent cell walls or a simple spatial index.
4. Reduce stored path data with a minimum-distance threshold or geometry-preserving simplification while retaining enough intermediate points for collision correctness. Collision samples and visual path storage do not need to be identical.
5. Put an upper bound on trace memory and add a performance test for long sessions.
6. Decide whether the start/end pulse is truly animated. The current `performance.now()` value is only observed when another event happens, so the pulse does not animate continuously. Either drive it from the single scheduler or remove the pseudo-animation.
7. Respect `prefers-reduced-motion` by disabling/reducing pulsing, screen shake, and modal scaling.

**Performance budget to validate on supported low-end hardware:** no repeated long tasks during a 30-second pen trace, one render at most per display frame, stable cursor feedback, and no unbounded growth caused solely by raw event rate. Set numeric frame-time and memory thresholds after Phase 0 establishes a realistic baseline.

### Phase 4 — Fix layout, resize, and runtime resilience (P1/P2)

1. Measure the actual `#game-container` content box instead of subtracting magic values from `window.innerHeight`.
2. Replace the uncancelled resize timers ([current resize handler](https://github.com/kiwispin/maze-game/blob/2b17e6199b77f0c033a541b6f2d896c89080cd5e/index.html#L947-L950)) with a `ResizeObserver` or a properly cancelled debounce.
3. Clamp available dimensions and cell size before division. Very small or landscape viewports can currently produce zero/negative `cellSize`, invalid row/column counts, or invalid canvas dimensions.
4. Decide and communicate resize behavior: either preserve the same maze/path by reprojecting it, or deliberately restart once after resize with an accessible notice. Do not silently generate multiple new mazes during a resize gesture.
5. Bound total rows/columns so ultrawide displays cannot create excessive generation/render work.
6. Remove `maximum-scale=1.0, user-scalable=no` from the viewport policy so users can zoom. Make the layout robust under zoom instead of disabling an accessibility feature.
7. Replace runtime Tailwind CDN compilation and the CSS `@import` font dependency with checked-in/static production CSS and robust system-font fallbacks. This avoids blank/incomplete UI when a CDN is blocked, offline, slow, or disallowed by Content Security Policy.
8. Create `AudioContext` lazily after a valid user gesture, handle unsupported/failed/suspended states, and keep sound optional. The current eager constructor can fail before game initialization ([current audio setup](https://github.com/kiwispin/maze-game/blob/2b17e6199b77f0c033a541b6f2d896c89080cd5e/index.html#L376-L379)).

### Phase 5 — Accessibility and UI correctness (P2)

1. Replace clickable `div` controls with semantic buttons/menu controls. Add visible focus styles and full keyboard operation for difficulty and Strict mode.
2. Add accessible names, `aria-pressed`/`aria-expanded` state, and status announcements for sound, Strict mode, error count, collision, restart requirement, and win.
3. Give the canvas an accessible name, concise instructions, and a non-pointer alternative. The long-term goal should be a keyboard-operable route method or equivalent control, not just a focusable canvas.
4. Give the success modal `role="dialog"`, `aria-modal`, labelled/described relationships, initial focus, focus containment, Escape behavior, and focus restoration.
5. Restore outside-click and Escape closing for the difficulty menu. `closeDropdown` exists but is never registered, so the current menu can remain open indefinitely.
6. Stop selecting difficulty by `innerText.includes(label)` and mutating `innerHTML`. Use stable data values and update dedicated label/check elements.
7. Ensure status is not conveyed by color alone and verify contrast at normal, hover, focus, collision, and disabled states.

### Phase 6 — Structure, security, and maintainability (P2)

1. Split the monolithic inline document into small modules with clear ownership: maze model/generation, geometry, input adapter/state machine, renderer, audio, UI, and bootstrap.
2. Move inline `onclick` handlers to registered event listeners. This improves testability and permits a restrictive Content Security Policy without `unsafe-inline`.
3. Make geometry and state transitions pure functions wherever possible so they can be unit tested without a browser.
4. Add explicit configuration constants for wall width, path radius, start tolerance, target radius, difficulty limits, DPR cap, and performance sampling thresholds.
5. Add failure handling for a missing 2D context and a small user-facing fallback for unsupported browsers.
6. Expand `README.md` with local run instructions, supported input/browser policy, test commands, architecture notes, and the Wacom manual test procedure.

## 5. Prioritized defect backlog found during review

| Priority | Issue | Impact | Planned phase |
| --- | --- | --- | --- |
| P0 | Removed crosshair and no app-rendered pen hover indicator | Cursor affordance missing; no fallback when OS/driver cursor is hidden | 1 |
| P0 | No Pointer Events, pointer capture, cancellation, or active-pointer identity | Wacom and hybrid-device input can fail, duplicate, switch pointers, or become stuck | 1 |
| P1 | CSS-to-canvas coordinate scaling omitted; zero coordinate uses a faulty truthy fallback | Path and collisions can be offset or throw at an edge | 2 |
| P1 | Collision function uses only bounding-box overlap | False collisions on diagonal moves and inconsistent tolerance | 2 |
| P1 | Resume inserts a disconnected gap and does not validate the reconnecting segment | Users can jump walls; game rules and rendering disagree | 2 |
| P1 | One wall scan and one queued render per raw move; full scene/path redraw | Stylus-rate input can lag or appear frozen | 3 |
| P1 | Resize creates uncancelled timers and can calculate invalid dimensions | Progress resets repeatedly; tiny viewports can break initialization | 4 |
| P2 | Win condition does not match the visible circular target | Visual/gameplay inconsistency and sparse-event misses | 2 |
| P2 | Pulse is calculated without a continuous animation loop | Claimed animation is static except during unrelated redraws | 3 |
| P2 | Eager audio initialization is unguarded | A media-policy/API failure can prevent the game script from starting | 4 |
| P2 | Zoom disabled and custom controls/canvas lack semantic accessibility | Keyboard, screen-reader, low-vision, and motor-access barriers | 4–5 |
| P2 | Difficulty menu cannot close on outside click; fragile text/`innerHTML` state | Persistent overlay and brittle UI behavior | 5 |
| P2 | Runtime CDN dependencies and inline handlers | Offline/CSP/reliability risk and harder testing | 4–6 |
| P3 | No tests, linting, formatting, build validation, or useful README | Regressions are easy to introduce and hard to diagnose | 6–7 |

## 6. Test strategy

### Unit tests

Add tests for:

- CSS/client-to-logical coordinate conversion across offsets, non-uniform scale, zoom, and DPR;
- a legitimate `clientX` or `clientY` of zero;
- exact segment/swept-radius collision against horizontal, vertical, corner, tangent, and non-intersecting walls;
- target intersection, including a segment that crosses the target between sparse samples;
- legal and illegal state transitions for down/move/up/cancel/lost-capture/blur;
- primary pointer/button filtering and second-pointer rejection;
- Normal resume, restart, Strict lift/cancel, collision, clear, and win semantics;
- layout clamping for tiny, portrait, landscape, and ultrawide containers;
- deterministic maze validity with an injectable seeded random source: all cells reachable, wall symmetry intact, start/end defined.

### Browser integration tests

Use a lightweight local HTTP server and automated browser tests to cover:

1. Mouse completion and collision flows.
2. Synthetic Pointer Events with `pointerType` values of `mouse`, `pen`, and `touch`.
3. Pointer capture and release outside the canvas.
4. `pointercancel`, `lostpointercapture`, tab hiding, and window blur.
5. CSS-scaled canvas and browser viewport changes.
6. No duplicated path when compatibility mouse events are present.
7. Difficulty/Strict controls, outside click, Escape, keyboard navigation, modal focus, and sound state.
8. Reduced-motion behavior.
9. Long high-frequency traces with a render-count and memory assertion.
10. A production/offline smoke test that does not depend on Tailwind or Google Fonts being reachable.

Synthetic pen tests validate the application logic but cannot validate Wacom drivers, OS cursor behavior, or browser/driver integration. Real hardware testing is a release gate.

### Manual hardware/browser matrix

At minimum, test:

| Platform | Hardware/input | Browsers/configurations |
| --- | --- | --- |
| Windows 10 and 11 | Reported Wacom Bamboo model(s), pen hover and contact | Current supported Chrome, Edge, Firefox; Windows Ink on and off |
| Windows 10 and 11 | Standard USB/Bluetooth mouse | Same browsers; 100% and high-DPI display scaling |
| Windows touch device | Touch plus pen if available | Edge and Chrome; single and accidental second touch |
| macOS | Wacom tablet plus trackpad/mouse | Current supported Safari, Chrome, Firefox |
| iPadOS/iOS if supported | Touch and Apple Pencil | Safari |
| Android if supported | Touch and stylus where available | Chrome |

For every cell in the matrix, verify hover visibility, start gating, continuous route tracking, collision accuracy, leaving/re-entering, outside release, cancel/focus loss, browser zoom, resizing/orientation, Strict mode, completion, and a 30-second continuous-input soak.

## 7. Acceptance criteria for the cursor/Wacom fix

The P0/P1 remediation is complete only when all of the following are true:

1. The exact reported Bamboo configuration has been tested, or the unavailable hardware is explicitly documented as a residual validation gap.
2. A pen position indicator remains visible while the pen is in range over the maze and correctly follows the tip without material lag.
3. Pen contact begins a route at the start zone and continues without interruption until up/cancel/collision/win, including brief movement outside the element while captured.
4. Mouse and touch behavior remains correct and no device produces duplicate event handling.
5. Pointer cancellation, lost capture, release outside, blur, modal opening, and tab hiding all leave the app in a safe, non-drawing state.
6. Cursor/path position is accurate within one CSS pixel (or a documented device-appropriate tolerance) at supported zoom and display scales.
7. Collision and goal detection match the visible wall/path/target geometry under slow and fast movement.
8. A 30-second high-rate pen trace stays responsive, queues no redundant frame callbacks, and remains inside the agreed memory/frame-time budget.
9. Users are not instructed to change Wacom or Windows Ink settings as the product fix.
10. Automated unit and browser tests pass, the offline smoke test passes, and no new accessibility-critical violations are introduced.

## 8. Delivery sequence and pull-request boundaries

Keep changes reviewable and reversible:

1. **PR 1 — Reproduction harness and tests:** sanitized diagnostics, pure coordinate/geometry/state tests, and baseline browser flows. No user-visible behavior change except debug mode.
2. **PR 2 — Pointer/cursor hotfix:** unified Pointer Events adapter, capture/cancellation, active pointer, restored system crosshair, in-app pen hover indicator, and focused Wacom regression tests.
3. **PR 3 — Geometry and rules:** scaled/DPR coordinates, exact collision/goal tests, state machine, and explicit Normal/Strict resume behavior.
4. **PR 4 — Rendering/performance:** single scheduler, cached static layer, spatial collision lookup, bounded/simplified path data, reduced motion.
5. **PR 5 — Responsive/runtime hardening:** container-based sizing, safe resize, small-viewport handling, dependency removal, lazy audio.
6. **PR 6 — Accessibility/UI/structure:** semantic controls, modal/menu behavior, keyboard alternative, module split, CSP-compatible handlers, and documentation.

Each PR should include before/after evidence, automated tests, manual matrix rows completed, performance impact where relevant, and a rollback note. The pointer/cursor hotfix should be releasable independently; deeper refactors should not delay it.

## 9. Rollout and follow-up

1. Release the pointer/cursor hotfix to a small tester group that includes affected Wacom users.
2. Ask testers to classify any remaining problem using the Phase 0 symptom categories and attach the sanitized trace plus environment matrix.
3. Monitor client-side exceptions and performance only if a privacy-reviewed telemetry mechanism is added; the current project has none, so do not silently introduce tracking.
4. Keep the legacy input adapter for one defined compatibility window only if usage proves it is required. Remove it once the supported-browser policy allows.
5. After two stable releases, close the temporary diagnostics path or keep it behind an explicit query flag documented for support.
6. Update the manual hardware matrix whenever supported browser, OS, or Wacom driver baselines change.

## 10. Definition of done for the wider remediation

The wider plan is complete when the P0–P2 backlog is resolved or explicitly deferred with ownership and rationale; real Wacom hardware passes the release gate; automated input, geometry, layout, performance, UI, and accessibility tests run in continuous integration; production no longer relies on runtime styling/font CDNs; the README describes support and validation; and the repository has a repeatable build/test/release process.
