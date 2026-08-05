# Canvas Studio — Architectural Audit & Permanent Design Document

**Source file:** `stabilizationtest.html` (2,302 lines, single-file prototype)
**Audience:** Engineers integrating this prototype into the main project
**Status:** This is a read-only audit. No code was modified to produce this document.

---

## 0. Executive Overview

Canvas Studio is a single-file HTML prototype implementing a pressure-sensitive
digital painting surface. It combines:

- A **WebGPU** rendering backend with a **WASM** module for curve/vertex math,
  falling back to **Canvas2D** if WebGPU is unavailable.
- A **4x supersampled backing store** (1920×1080 logical → 7680×4320 backing
  pixels) resolved down via a box-filter blit shader for antialiasing.
- A hand-built **stroke stabilization ("smoothing") pipeline** that is the
  primary subject of this prototype — it is explicitly a research vehicle for
  reverse-engineering "Moving Average"-style stabilization (as documented in
  competing painting tools), including a visible "leash" overlay, zoom-aware
  compensation, pressure stabilization, and two distinct stroke-release
  ("finish") behaviors.
- Standard UI chrome: tool selection, color/size/opacity, zoom, pan, undo/redo,
  PNG export.

The file is organized as one big IIFE (`(async function(){ ... })()`) with a
top-level `try/catch` that surfaces fatal errors to a `#fatal` overlay. Inside,
there is a WebGPU/WASM branch and a much smaller Canvas2D fallback branch that
share the same outer input-handling code but implement `beginStroke`,
`moveStroke`, `endStroke`, `pushUndo`, `restore`, and `clearCanvas`
independently per backend.

**Important integration note:** the stabilization/pressure engine — the most
architecturally significant and heavily-instrumented part of the file — exists
**only in the WebGPU branch**. The Canvas2D fallback (lines 1767–1828) is a
bare-bones raw-line-segment renderer with no stabilization, no pressure
tapering, and no undo texture pooling; it exists purely so the app doesn't
hard-fail on non-WebGPU browsers.

---

## 1. Rendering

### 1.1 Backend selection
- `useWebGPU = !!navigator.gpu` (line 213) gates the entire rendering strategy.
- WebGPU bring-up (lines 217–245) requests an adapter with
  `powerPreference: 'high-performance'`, then a device, then configures the
  canvas context with the browser's preferred format and `alphaMode: 'opaque'`.
- All adapter/device/context calls are wrapped in `withTimeout()` (5000ms) —
  a defensive measure against a hung `requestAdapter()`/`requestDevice()`
  call, which has been observed to occur on some GPU/driver combinations.
- On any failure, `gpuFailReason` is recorded and `useWebGPU` is forced false;
  the app falls back to Canvas2D transparently. The `#gpuBadge` UI element
  reflects the active mode ("WebGPU active" vs "Software 2D mode").
- If `navigator.gpu` is entirely absent, a `#fatal` overlay is **available**
  in markup but is actually never shown in the current code path — the app
  always falls back to Canvas2D rather than displaying the fatal screen. (See
  Cleanup Opportunities §13 — this is likely dead/aspirational UI.)

### 1.2 Canvas2D fallback
- Used only when WebGPU is unavailable.
- Paints directly into a supersampled backing store (`BW × BH` = `W*SS ×
  H*SS`) using `ctx2d.arc()` for dabs and straight `lineTo` segments for
  strokes (`beginStroke`/`moveStroke`, lines 1793–1822).
- No stabilization, no pressure-based radius tapering beyond a static
  `scaledBrushSize()`, no midpoint quadratic smoothing.
- Undo/redo uses `getImageData`/`putImageData` snapshots (`snapshot()`,
  line 1773) — expensive CPU-side copies, capped at `MAX_UNDO = 12`.
- **Classification: Production fallback, functionally minimal.** It is a
  intentional degraded-mode path, not experimental, but it does not implement
  any of the stabilization features that are the prototype's real subject.

### 1.3 WebGPU pipeline (primary path)
Three render pipelines are constructed:

1. **`strokePipeline`** (lines 488–601) — draws stroke geometry (dabs and
   capsule segments) into a single-channel `r8unorm` **stroke mask texture**
   (`strokeMaskTex`). Fragment shader computes an *exact* analytic
   signed-distance-to-capsule per fragment (`sdSegment`) rather than relying
   on pre-extruded, linearly-interpolated coverage geometry. Blending mode is
   `max` (both color and alpha channels) so overlapping dabs within one
   stroke do not double up coverage/darken at self-intersections.
2. **`blitPipeline`** (lines 646–676) — resolves (box-filters) the 4×4
   supersampled `accTex` down to the swap-chain's presented resolution. This
   is the antialiasing step: `sourceOrigin = pixel * 4`, sums a 4×4 block of
   texels, divides by 16.
3. **`compositePipeline`** (lines 686–713) — merges the current stroke mask
   (coverage) onto the "base" texture (the canvas state from before the
   current stroke started) using `mix(base, paint, coverage)`, writing the
   result into `accTex`. This is what makes an in-progress stroke visible
   in real time without permanently baking it in until the stroke ends
   (though note: it actually *is* baked into `accTex` every time
   `compositeStroke()` runs — see State Management for nuance).

### 1.4 Vertex generation
- Two vertex-builder functions produce **bounding-box quads** (2 triangles,
  6 vertices) carrying true shape parameters as per-vertex attributes, so the
  fragment shader can compute exact analytic distance rather than relying on
  interpolated coverage:
  - `circleVerts(cx, cy, r, alpha)` (line 831) — a round dab; capsule with
    `p0 == p1`.
  - `segmentVerts(x0,y0,x1,y1,r0,r1,alpha)` (line 844) — a tapered capsule
    between two points, with independent start/end radii to support pressure
    tapering along a curve segment.
- Both add an `AA_MARGIN` (2.0 backing px) around the true geometry so the
  ~1px antialiased edge computed in the fragment shader fits fully inside the
  quad. This margin is purely a bounding-box padding, not a blur radius — the
  comments are explicit that the actual edge softness is computed exactly per
  fragment via `fwidth(d)`.

### 1.5 Accumulation / compositing / antialiasing model
- `accTex` (RGBA8, `BW×BH`) is the **full-resolution accumulated canvas
  image**. It's cleared to white once at startup (`clearTexToWhite`) and then
  is continuously overwritten by `compositeStroke()` while drawing.
- `strokeMaskTex` (`r8unorm`, `BW×BH`) is a **per-stroke coverage
  scratch buffer**, cleared at `beginStrokeCoverage()` (i.e., at the start of
  every stroke) and accumulated into (via `max` blending) as dabs/segments
  stream in during `drawVertsIntoAcc`.
- Antialiasing is achieved two ways simultaneously: (a) analytic per-fragment
  SDF edge softening in `strokePipeline`'s fragment shader, and (b) 4×4
  supersample box-filter downsampling in `blitPipeline`. This is a
  belt-and-suspenders AA approach — SS=4 gives 16 samples/pixel on top of
  already-analytic edges.
- `present()` runs the blit pipeline against the swap-chain texture every
  time a stroke updates, i.e., every batch/tick, not throttled beyond RAF.

### 1.6 WASM
- A single `<script type="text/plain" id="brush-wasm-b64">` element holds a
  base64-encoded WASM binary (line 177), decoded and instantiated at startup
  (`WebAssembly.instantiate`, line 457).
- **This WASM module is loaded but functionally unused by the active code
  path.** The only WASM exports referenced are `getOutPtr` (in the unused
  helper `readVerts`, line 463) — and `readVerts` itself is never called
  anywhere else in the file. All actual curve math (`drawQuadCurve`,
  `feedPoint`, vertex generation) is implemented in plain JavaScript.
- **Classification: Dead/vestigial integration.** The WASM brush appears to
  be a leftover from an earlier architecture (possibly the "brush = WASM,
  canvas = WebGPU" claim in the `#hint` UI text, line 166) that was since
  reimplemented in JS but never removed. This is a significant integration
  risk: shipping an unused multi-KB WASM blob with no effect, and a stale UI
  hint claiming WASM does the brush math when it does not.

---

## 2. Brush Engine

### 2.1 Stroke lifecycle
Three lifecycle functions are assigned per-backend: `beginStroke`,
`moveStroke`, `endStroke` (plus `pushUndo`, `restore`, `clearCanvas`).
In the WebGPU branch:

1. **`beginStroke(e)`** (line 1104) — cancels any pending finish/hold RAFs
   from a prior stroke, mints a new `activeStrokeSessionId`, converts the
   pointer event to canvas space, pushes an undo snapshot, clears the stroke
   mask (`beginStrokeCoverage`), resets/prefills `smoothBuf`/`pressureBuf` to
   the full moving-average window length (so stabilization strength is
   constant from the first sample, not ramping up), stamps an initial dab via
   `circleVerts`, and starts the persistent `strokeHoldTick` RAF loop.
2. **`moveStroke(e)`** (line 1319) — does *not* render directly. It only
   queues coalesced pointer samples into `pendingStrokeSamples`; actual
   processing happens via the always-running `strokeHoldTick` loop (see
   Performance §5) or via `flushPendingStrokeSamples()`.
3. **`endStroke(e)`** (line 1488) — the most complex function in the file. It
   determines a "finish mode" (Stationary Hold vs Moving Release, see
   Stabilization §3.6), flushes/discards queued samples accordingly, then
   runs an accelerating "catch-up" animation (`finishTick`, line 1679) that
   continues feeding the stabilization buffers with the frozen final
   position/pressure until the smoothed output converges to the true
   endpoint, extensively logging diagnostics along the way.

### 2.2 `feedPoint()` — curve generation entry point
`feedPoint(all, raw, pressure)` (line 1201) is the single choke point that
turns one stabilized `(x,y)` + `pressure` sample into rendered geometry:
- Computes radius (`pressureRadius`) and alpha (`pressureAlpha`) from
  pressure.
- Builds a quadratic Bézier segment using the **midpoint-quadratic
  technique**: `midStart` = previous stroke midpoint, `control` = `lastRaw`
  (the actual last stabilized raw sample, used as the Bézier control point),
  `midEnd` = midpoint of `lastRaw` and the new `raw`. This is the classic
  "smooth freehand curve from raw polyline" technique — each segment's
  control point is the actual sample, and the drawn curve passes through
  midpoints, guaranteeing C1-ish continuity without visible polyline kinks.
- Delegates to `drawQuadCurve()` to tessellate the Bézier into `steps` (10)
  straight capsule sub-segments with per-vertex radius interpolation
  (pressure taper along the curve).
- Advances `lastMid`, `lastR`, `lastAlpha`, `prevRaw`, `lastRaw`.
- Contains a **temporary diagnostic block** (`__feedPointCallIndex < 3`,
  lines 1208–1222) that `console.log`s the first three calls per stroke for
  investigating "startup kink" behavior. Explicitly marked TEMPORARY in
  comments — see Cleanup Opportunities.

### 2.3 `drawQuadCurve()`
`drawQuadCurve(all, p0, control, p1, r0, r1, alpha, steps)` (line 1171) —
tessellates one quadratic Bézier into `steps` line segments, each emitted as
a tapered capsule via `segmentVerts`, and returns the final point (used as
the next `lastMid`). Pure geometry; not stabilization-specific.

### 2.4 Segment generation / spacing
There is no explicit fixed-distance "dab spacing" model (no min-distance gate
between dabs) — instead, curve tessellation is continuous (`steps=10` per
feedPoint call) and long jumps between samples are subdivided
(`processStrokeBatch`, `maxStep = max(2, scaledBrushSize()*0.6)`, line 1238)
purely to keep curve tessellation dense; explicitly documented as **not
affecting the Moving Average** (comment, line 1237).

### 2.5 Pressure pipeline
- `getPressure(e)` (line 405) reads raw hardware pressure, clamped [0,1];
  mouse always reports 1; a zero-pressure pen sample (hover/contact
  threshold) is honored as zero rather than treated as "missing."
- `pressureInfluence(p)` (line 415) applies `p^1.2` easing.
- `pressureRadius(size, pressure)` (line 428) computes the dab radius: a
  special-cased 0.5× scale for ≤1px brushes (keeps the smallest presets in a
  tighter TVPaint-like subpixel range), blended between `MIN_ABS_RADIUS_BACKING`
  (0.05 × SS backing px absolute floor) and the full size via
  `pressureInfluence`.
- `pressureAlpha(_pressure)` (line 444) — **ignores its argument entirely**
  and returns the global `opacity` slider value. Pressure controls diameter
  only, not opacity — this is a deliberate design choice, documented in
  comments ("Hard Round pressure changes diameter, not paint opacity").
- A **separate pressure moving-average buffer** (`pressureBuf`,
  `pushPressureBuf()`, line 910) mirrors the position buffer
  (`pushSmoothBuf`) with the same window length (`movingAverageAmount()`),
  producing `delayedPressure` — a stabilized pressure value time-aligned with
  the stabilized position.
- A **cadence-invariant diagnostic** (`positionBufferAdvanceCount` /
  `pressureBufferAdvanceCount`, lines 894–895) is maintained to guarantee the
  position and pressure buffers advance in lockstep; a mismatch is logged as
  a warning at every `endStroke` call (line 1556–1562). Explicitly marked as
  a "temporary cadence-invariant diagnostic" in comments.
- **Release-tail filtering**: `isReleaseTailSample()` (line 902) and the
  contact-pressure tracker (`lastContactPressure`, `CONTACT_PRESSURE_FLOOR =
  0.02`, lines 928–950) exist to detect and exclude spurious near-zero
  pressure samples that some pen/tablet stacks emit while the tip is lifting
  off, *before* `pointerup` actually fires — these would otherwise corrupt
  `lastContactPressure` / stabilization pressure buffers with a false "the
  user eased off" signal.

### 2.6 Release ("finish") behavior — see §3.6 for full detail
Two modes, `Stationary Hold` vs `Moving Release`, both implemented inside
`endStroke`/`finishTick`. This is deeply intertwined with stabilization and
documented fully under §3.

### 2.7 Debug pressure override
`window.setStabDebugConstantPressure(on)` (line 944) — a global debug hook
that forces every sample's effective pressure to a fixed `0.7`, "isolating
position catch-up from pressure handling for testing." Off by default
(`DEBUG_CONSTANT_PRESSURE = false`). **Classification: debug-only, exposed on
`window`, inactive unless manually invoked from devtools.**

---

## 3. Stabilization — Complete Pipeline

This is the architectural core of the prototype. The UI exposes a single
"Stabilization Amount" slider (0–100) plus a "Leash" visibility toggle.
Internally, the raw slider value is transformed multiple times before it
actually affects drawing.

### 3.1 Conceptual model
The implementation is a **clean-room approximation of a documented "Moving
Average" stabilization mode** (explicitly called out in a code comment as
"not a claim about [a third-party tool]'s proprietary implementation," line
873–877). The core mechanism: every incoming raw pointer sample is pushed
into a fixed-length FIFO (`smoothBuf`), and the *arithmetic mean* of the
buffer's contents is what actually gets drawn. A larger window = more lag =
more smoothing.

### 3.2 `movingAverageAmount()` — window size derivation
```
movingAverageAmount():
  internalAmount = computeInternalStabilization()
  if internalAmount <= 0: return 1        // effectively "off" — window of 1
  return max(2, round(internalAmount * 100))
```
So the moving-average window length is a direct function of a 0–1 internal
stabilization amount, scaled ×100 (i.e., UI 30% ≈ 30-sample window, UI 100% ≈
100-sample window), floored at 2 whenever any stabilization/compensation is
active at all.

### 3.3 `computeInternalStabilization()` — where zoom compensation enters
This is **the single place the UI slider value becomes the value actually
used for the moving-average window** (explicit in comments, line 1014). It
blends in a "hidden" zoom-dependent minimum:
```
internalAmount = clamp(
  userAmount + zoomStabilizationMinimum(zoom) * zoomCompensationWeight(userAmount),
  0, 1
)
```
- `userAmount` = raw UI slider value (0–1), never itself mutated — the
  slider's displayed percentage always reflects only what the user set.
- The **zoom minimum** and **compensation weight** are described fully in
  §6 (Zoom System) since they are zoom-system outputs consumed here.
- A one-shot-per-stroke debug log exists behind `DEBUG_ZOOM_COMPENSATION`
  (`false` by default, line 304) via `zoomCompDebugLoggedForStroke`.

### 3.4 `pushSmoothBuf(p)` — position moving average
```
pushSmoothBuf(p):
  maxLen = movingAverageAmount()
  smoothBuf.push({x,y}); trim to maxLen (FIFO)
  if maxLen == 1: return p unchanged
  return arithmetic mean of smoothBuf
```
Called from `processStrokeBatch` (live drawing), `strokeHoldTick` (held-still
catch-up), and `finishTick` (release catch-up) — i.e., it is the **single
implementation** used across all three "modes" of stroke life, per the
architecture's explicit design goal of a single accumulator without a
separate "release taper" algorithm.

### 3.5 Pressure stabilization
`pushPressureBuf(p)` (line 910) — structurally identical moving-average FIFO
for pressure, sharing `movingAverageAmount()` as its window length so
position and pressure smoothing are always time-aligned. See §2.5 above.

### 3.6 Hold behavior — `strokeHoldTick`
A **persistent per-frame RAF loop** (line 1344), started once at
`beginStroke` and running for the entire duration the pointer is down
(session-guarded via `strokeHoldSession === activeStrokeSessionId`).
Each tick:
- If new samples arrived this frame (`pendingStrokeSamples.length`), flush
  them through the normal path.
- Otherwise (pen genuinely stationary, no new events): repeatedly re-feed the
  last known input position/pressure into `pushSmoothBuf`/`pushPressureBuf`,
  driven by **elapsed wall-clock time** rather than a fixed per-frame tick
  count (`ticksPerMs = movingAverageAmount() / catchUpDurationMs`, converge
  target `catchUpDurationMs = 350ms`), so the on-screen "catch-up to a held
  pen" animation speed is independent of frame rate/display refresh.
- This is what makes stabilization "catch up" visibly while the pen is held
  still, instead of only catching up once lifted.
- Also eases `recentSpeedPxMs` toward zero when idle so velocity-based finish
  pacing (§3.7) isn't left stale from the last real motion.

### 3.7 Finish (release) behavior — two modes
`endStroke()` first computes `wasStoppedBeforeLift` via
`computeWasStoppedBeforeLift()` (line 1432), which inspects
`lastMoveEventTime` **and** any still-queued (unflushed)
`pendingStrokeSamples` to decide whether the pen was already idle (beyond
`HOLD_BEFORE_LIFT_MS = 120ms`, filtered through the same jitter floor and
release-tail classifier used live) before it actually lifted.

- **Stationary Hold** (pen was idle before lift): `finishPressure` is a
  single frozen snapshot of `delayedPressure`, taken *before* any pending
  samples are flushed (to avoid mutating it), and any queued samples are
  discarded rather than flushed (they're known to be jitter-only). This
  frozen value is reused unchanged for every remaining catch-up tick — no
  further calls to `pushPressureBuf()` at all during this finish.
- **Moving Release** (pen was still moving at lift): pending samples are
  flushed normally first, then `finishPressure = lastInputPressure` (the
  last real sample) is used as a constant "boundary" value fed into
  `pushPressureBuf()` on every finish tick — exactly mirroring how position's
  frozen `endpoint` is fed into `pushSmoothBuf()` every tick. This is *not* a
  separate/different smoothing algorithm — same FIFO, just constant input.
- The switch between modes is centralized behind `nextFinishPressure()`
  (line 1544), so the two modes can never diverge across the shortcut-return
  branch, the main finish loop, and the final endpoint sample.
- **Position catch-up pacing** (`finishTick`, line 1679) is independent of
  pressure mode: an accelerating tick rate (`startRatePerMs` → `endRatePerMs`,
  smoothstep-interpolated over `targetDurationMs`, itself derived from gap
  distance and recent velocity, clamped to `[MIN_FINISH_MS=80,
  MAX_FINISH_MS=260]`) repeatedly calls `pushSmoothBuf(endpoint)` (feeding
  the *frozen* true pointer-up position every tick) until the smoothed output
  converges to within `0.15px` of the endpoint or the buffer empties, then
  emits one final `feedPoint` call landing exactly on the endpoint.
- Extensive `diag` object accumulates finish metrics (durations, tick counts,
  frame-by-frame pressure trace up to 60 frames) and is logged via
  `logFinishDiagnostics()` at completion — see Debug Systems §7.
- **Ownership/session guard**: `finishTick` captures `mySession =
  activeStrokeSessionId` at schedule time and bails out immediately if a
  newer stroke has begun (`beginStroke` bumps the token and cancels the RAF),
  preventing a stray finish callback from corrupting a new stroke's shared
  mutable state (`lastRaw`, `lastMid`, `smoothBuf`, etc.). This pattern is
  documented as a "REQUIRED OWNERSHIP RULE" in comments (line 330) and is
  applied consistently to `strokeHoldTick` too.

### 3.8 Velocity tracking
`trackVelocity(raw, timeStamp)` (line 969) maintains `recentSpeedPxMs` via
light exponential smoothing (`0.7/0.3` blend) of instantaneous raw pointer
speed. Used **only** to pace the finish catch-up duration (fast flick vs slow
deliberate stroke); explicitly does not affect curve reconstruction, spacing,
or what gets drawn during normal drawing (comment, lines 960–965). A prior
"release-velocity rolling window" / "flick-taper pressure envelope" system
that used to consume this has been **removed** (explicit note, lines 984–989)
— pressure is no longer synthesized from velocity at all; only position
catch-up timing uses velocity.

### 3.9 Leash visualization
`drawStabilizerLeash()` (line 1059) draws, into a dedicated overlay canvas
(`#stabViz`, transparent, `pointer-events:none`, resized to DPR in
`resizeStabViz()`), a dashed line from the raw/anchor pointer position to the
stabilized brush tip position, plus two dots (anchor = raw input, tip =
stabilized output). Gated on `leashEnabled && drawing && stabilization > 0`
and a minimum 1px separation. Purely visual — never affects drawing math.
Explicitly modeled after a well-known reference implementation's leash
indicator (comment, line 1052).

### 3.10 Zoom-dependent stabilization
See §6 in full; summarized: at low zoom, a fixed *screen-space* stabilization
window covers a much larger *canvas-space* distance, so a hidden minimum
smoothing floor is blended in that fades out both as zoom increases and as
the user's own slider value increases (fully faded by UI 20%).

### 3.11 Internal stabilization amount vs UI value
The codebase is careful to maintain a clear separation:
- `stabilization` (module-level variable, 0–1) — the **raw UI slider value**,
  only ever set by the slider's `oninput` handler. Never mutated by the
  stabilization math itself.
- `computeInternalStabilization()`'s return value — the **actual amount used
  internally** (window sizing, zoom-compensated). This is what
  `movingAverageAmount()` consumes. The UI display (`stabVal.textContent`)
  always reflects only the raw slider value, by design.

### 3.12 Stabilization-related state variables (see also §9)
`smoothBuf`, `pressureBuf`, `delayedPressure`, `lastInputRaw`,
`lastInputPressure`, `lastContactPressure`, `positionBufferAdvanceCount`,
`pressureBufferAdvanceCount`, `releaseTailClassifierActive`,
`releaseTailLastPressure`, `recentSpeedPxMs`, `lastVelRaw`, `lastVelTime`,
`lastMoveEventTime`, `zoomCompDebugLoggedForStroke`, `DEBUG_CONSTANT_PRESSURE`.

### 3.13 Data flow: pointer input → `feedPoint()`
See §10 for the full diagram. In prose: raw pointer/coalesced event →
`screenToCanvas` → jitter/idle/release-tail classification updates →
`pushPressureBuf` (pressure moving average) → `pushSmoothBuf` (position
moving average) → long-jump subdivision if needed → `feedPoint` (curve +
render). The **hold** and **finish** paths re-enter at the
`pushSmoothBuf`/`pushPressureBuf` → `feedPoint` stage using synthetic
(held/frozen) inputs instead of fresh pointer samples, but through the exact
same functions.

---

## 4. Input System

### 4.1 Pointer events
Registered on `stageWrap` (`pointerdown`, `pointermove`, `pointerenter`,
`pointerleave`) and `window` (`pointerup`, `pointercancel`) — `pointerup`
is on `window` rather than the canvas so a stroke correctly ends even if the
pointer leaves the canvas bounds before release.
- `pointerdown` (line 1831) — dispatches to pan/zoom-drag/size-drag start
  handlers based on modifier-key state (`cDown`, `spaceDown`, `ctrlDown`,
  middle-button), else begins a stroke. Explicitly **ignores a second
  pointer** while a stroke from a different pointer is in progress (palm
  rejection / stray-pointer guard, comment lines 1837–1843) to prevent a
  "ghost diagonal line" artifact from ownership confusion.
- `pointermove` — updates hover state unconditionally (for brush-size
  preview), then dispatches to whichever drag mode is active, else
  `moveStroke` if the event's `pointerId` matches the active stroke's owner.
- `pointerup`/`pointercancel` — ends whichever interaction was active; guards
  against ending a stroke owned by a different pointer.

### 4.2 `pointerrawupdate`
**Not used anywhere in this file.** The app relies entirely on standard
`pointermove` + `getCoalescedEvents()`. No `pointerrawupdate` listener exists.

### 4.3 Coalesced events
Used in two places: `processStrokeBatch` (line 1234) and `moveStroke` (line
1320) both check for `e.getCoalescedEvents` and prefer the coalesced array
over the single event when available, ensuring no sub-frame pointer samples
are dropped by browsers that batch multiple hardware samples per animation
frame.

### 4.4 Pressure & tilt
Pressure is read via `getPressure(e)` (§2.5). **Tilt is not read or used
anywhere in the file** — no `e.tiltX`/`e.tiltY` references exist.

### 4.5 Event batching
`moveStroke` doesn't render synchronously; it pushes samples into
`pendingStrokeSamples`, which is drained once per animation frame by the
always-running `strokeHoldTick` loop (or, when idle, by the loop's own
held-position catch-up branch). This decouples pointer event frequency from
render frequency (see Performance §5).

### 4.6 `screenToCanvas()`
```
screenToCanvas(clientX, clientY):
  x = ((clientX - wrapRect.left - panX) / zoom) * SS
  y = ((clientY - wrapRect.top  - panY) / zoom) * SS
  return {x, y}
```
Converts a browser client-space coordinate into backing-store (supersampled)
canvas space, accounting for pan, zoom, and the SS=4 supersample factor.
Companion functions `wrapToCanvas`/`canvasToWrap` (line 2003) do the same
conversion without the `wrapRect` subtraction, for callers that already have
wrap-relative coordinates (size-drag, zoom-drag anchors).

---

## 5. Performance

### 5.1 `requestAnimationFrame` usage
Three independent RAF loops can be active simultaneously, all
session-guarded against a newer stroke starting:
1. `strokeHoldRAF` / `strokeHoldTick` — runs for the entire duration a
   pointer is down (started in `beginStroke`, cancelled at `endStroke`
   entry).
2. `finishRAF` / `finishTick` — runs after pointer-up until the stabilized
   position catches up to the frozen endpoint.
3. (Implicit) browser's own coalesced pointer-event delivery cadence is
   independent of both.
Neither loop runs when `drawing` is false and no finish is in progress —
there is no continuous "idle" render loop; `present()` is only called when
new geometry is drawn.

### 5.2 Batching
- Pointer samples are batched per-frame via `pendingStrokeSamples` +
  `flushPendingStrokeSamples()` rather than issuing a GPU draw per raw event.
- Within a single batch, all generated vertices across all coalesced events
  are accumulated into one JS array (`all`) and submitted as **one**
  `Float32Array` / one `drawVertsIntoAcc()` call / one GPU submit — not one
  submit per dab.

### 5.3 GPU uploads
- `device.queue.writeBuffer(vertexBuf, ...)` uploads the entire batch's
  vertex data in a single call per frame (`drawVertsIntoAcc`, line 769-773).
- `ensureVertexCapacity()` (line 614) grows the vertex buffer geometrically
  (doubling) only when a batch exceeds current capacity, avoiding
  reallocation on every frame.
- Uniform buffers (`strokeUniformBuf`, `compositeUniformBuf`) are small,
  fixed-size, and rewritten (not reallocated) every draw.

### 5.4 Memory reuse / object allocation
- `vertexBuf` is reused across the whole session, only reallocated
  (destroy + recreate) on capacity growth.
- Undo/redo in the WebGPU path uses **GPU-to-GPU texture copies**
  (`copyTextureToTexture`, `snapshotCurrent()`, line 793) rather than any
  CPU readback — explicitly called out as "cheap, no readback" in a comment
  (line 792). This is materially cheaper than the Canvas2D fallback's
  `getImageData`/`putImageData` CPU round-trip.
- Each `circleVerts`/`segmentVerts` call still allocates fresh JS arrays per
  dab/segment (`out.push.apply(...)`) — not pooled. Given batches are
  typically small (tens of dabs per frame), this is likely an acceptable
  GC pressure tradeoff, but is a candidate for pooling if profiling shows
  GC pauses under heavy fast strokes.
- `Array.from({length: amount}, () => ({x,y}))` in `beginStroke` allocates a
  fresh object per buffer slot deliberately (comment explains: avoids
  aliasing between slots from a shared object literal).

### 5.5 Perf instrumentation
`reportPerf(ms)` (line 339) maintains a rolling 30-sample average of
per-batch processing time (`processStrokeBatch`'s wall-clock duration),
displayed live in the `#perf` UI element as "X.XX ms/batch." This is a
lightweight, always-on, production-facing perf readout (not gated by a debug
flag).

### 5.6 Optimizations of note
- Long pointer jumps are subdivided only enough to keep curve tessellation
  visually smooth (`maxStep`), not over-subdivided.
- The `blitPipeline`'s box filter, `strokePipeline`, and `compositePipeline`
  are all `layout: 'auto'` pipelines created once at startup, not rebuilt
  per frame.
- Undo texture pooling: `undoStack` holds actual GPU textures (each a full
  `BW×BH` RGBA8 copy — a real memory cost of `MAX_UNDO=12` full-resolution
  textures, ~12 × 7680×4320×4 bytes ≈ 1.6GB worst case — see Cleanup
  Opportunities, this is a meaningful VRAM concern worth flagging for
  integration).

---

## 6. Zoom System

### 6.1 Zoom variables
`zoom` (float, default 1), `panX`/`panY` (pixel offsets). Clamped to
`[0.05, 30]` (5%–3000%) uniformly across every zoom entry point (wheel,
buttons, zoom-drag).

### 6.2 Coordinate conversions
`screenToCanvas`, `wrapToCanvas`, `canvasToWrap` (see §4.6) — all zoom-aware,
all also account for the `SS` supersample factor.

### 6.3 Supersampling interaction
`SS = 4` is a **fixed, zoom-independent** constant — the backing store is
always rendered at 4× the logical 1920×1080 resolution regardless of current
zoom level. Zoom only affects the CSS transform (`applyTransform`,
`transform: translate() scale()`) used to present that fixed backing store,
not the backing store's actual resolution.

### 6.4 Zoom-dependent pixelation (nearest vs bilinear)
`updatePixelatedState(currentZoom, wasPixelated)` (line 287) implements
**hysteresis** switching between `image-rendering: pixelated` and `auto`
(browser bilinear): switches to pixelated once zoom ≥ `PIXELATE_IN_ZOOM`
(1.2/120%), switches back to blurred once zoom ≤ `BLUR_OUT_ZOOM` (1.0/100%),
holds the prior mode in between to avoid flicker at a single threshold. This
state (`canvasPixelated`, `sizePreviewPixelated`) is tracked independently
for the main canvas and the brush-size preview canvas.

### 6.5 Zoom-dependent stabilization interaction (full detail)
This is the most subtle system in the file.

**Problem statement** (from comments, lines 293–299): `pointermove` fires
based on *screen-space* movement. At low zoom, the same real hand-jitter
distance covers much more *canvas-space* distance (because
`screenToCanvas` divides by zoom), so a user-selected stabilization of 0
still looks jittery once zoomed far out, even though the same hand motion at
100% zoom would look fine.

**Two independent zoom-compensation mechanisms exist**, serving different
parts of the pipeline:

1. **`zoomSmoothingFactor()`** (line 401): `min(4, max(1, 1/zoom))` — used
   purely to scale the **jitter floor** (`JITTER_FLOOR_PX`, used in
   `processStrokeBatch` and `computeWasStoppedBeforeLift` for held-before-lift
   / release-tail classification), *not* the moving-average window itself.
   Capped at 4× so it doesn't run away at extreme zoom-out.

2. **Hidden stabilization-amount floor** — a second, separate mechanism that
   *does* affect the moving-average window size, via
   `computeInternalStabilization()` (§3.3):
   - `zoomStabilizationMinimum(z)` (line 996): log-interpolates a "hidden
     minimum" internal stabilization amount between `ZOOM_COMP_MAX_AMOUNT`
     (0.15, at `ZOOM_COMP_MIN_ZOOM` = 5% zoom) and `0` (at
     `ZOOM_COMP_ZERO_ZOOM` = 500% zoom). Log-space interpolation is used
     because zoom spans a multiplicative range (comment, lines 993–995).
   - `zoomCompensationWeight(userAmount)` (line 1007): a smoothstep fade that
     makes this hidden floor apply at full strength when the user's own
     slider is at 0, and fades it to zero by the time the user's slider
     reaches `ZOOM_COMP_FADE_UI_LIMIT` (0.20/20%) — so a user who has
     deliberately chosen meaningful stabilization isn't further altered by
     the hidden floor, but a user at "0% stabilization" at extreme zoom-out
     still gets some smoothing.
   - Debug logging behind `DEBUG_ZOOM_COMPENSATION` (off by default).

**Critical design invariant**: neither mechanism ever mutates the
user-visible `stabilization` variable or its slider display — both operate
purely on derived/internal values.

---

## 7. Debug Systems

### 7.1 Overlays / visualizers
- **Stabilizer leash** (`#stabViz` canvas, `drawStabilizerLeash()`) —
  production-facing, user-toggleable via the "Leash" checkbox (checked by
  default). Not purely a debug tool; it's a user-facing feature that happens
  to visualize internal state. Classification: **production feature**, not
  debug-only.
- **Brush size preview** (`#sizePreview` canvas, `showSizePreview()` /
  `updateBrushPreview()`) — production feature (the app deliberately hides
  the native OS cursor, `cursor:none`, and substitutes this circle).

### 7.2 Debug flags
- `DEBUG_ZOOM_COMPENSATION` (const, `false`, line 304) — one-shot-per-stroke
  console log of zoom-compensation math.
- `DEBUG_CONSTANT_PRESSURE` (let, `false`, line 943) — toggled via
  `window.setStabDebugConstantPressure(bool)`, forces pressure to 0.7.

### 7.3 Logging (always-on, not flag-gated)
Several `console.log`/`console.warn` calls fire unconditionally, i.e., are
**not** behind debug flags and will fire in a production build as currently
written:
- WebGPU bring-up steps (`[CanvasStudio] requesting adapter...`, `device:`,
  `context:`, `format:`, `configured OK`) — lines 219–237.
- WASM instantiation steps (lines 456, 460).
- `[feedPoint diag]` — first 3 feedPoint calls per stroke (line 1211),
  explicitly labeled TEMPORARY DIAGNOSTIC in comments.
- `[stabilizer cadence]` — every single `endStroke` call (line 1556), plus a
  `console.warn` if position/pressure buffer advance counts mismatch (line
  1562).
- `[stabilizer finish]` — every `endStroke` completion, via
  `logFinishDiagnostics()` (line 1469), a large structured diagnostic object.
- `[zoom stabilization compensation]` — gated behind `DEBUG_ZOOM_COMPENSATION`
  (off), so effectively silent in current config.
- `[stabilizer debug] constant pressure override:` — only fires when the
  debug hook is manually invoked.
- Top-level `catch` block logs `'Canvas Studio fatal error:'` on any
  uncaught exception during setup (line 2294).

**Classification: significant cleanup surface.** The cadence and finish
diagnostics fire on *every single stroke* unconditionally — appropriate for
active development/investigation of the stabilization algorithm, but not
appropriate to ship as-is; they should be gated behind a debug flag (or
removed) before/during integration.

### 7.4 Experimental code markers
Several comments explicitly self-identify as temporary/experimental:
- `feedPoint`'s diagnostic block: "TEMPORARY DIAGNOSTIC (pairing verification
  only — remove after confirming)" and "TEMPORARY DIAGNOSTIC (stroke-start
  stabilization investigation only — remove once the startup-kink question is
  settled)" (lines 1190–1199).
- `positionBufferAdvanceCount`/`pressureBufferAdvanceCount`: "Temporary
  cadence-invariant diagnostics" (line 888).
- `window.setStabDebugConstantPressure`: "Debug/validation hook (goal: 'test
  with position-only debug mode')" (line 939).

---

## 8. Experimental Features

| Feature | Active/Inactive | Notes |
|---|---|---|
| WASM brush module | **Loaded but inactive** | Instantiated at startup; its only referenced export (`getOutPtr`, via unused `readVerts()`) is never called. All brush math runs in JS. Effectively dead weight. |
| `DEBUG_CONSTANT_PRESSURE` override | Inactive by default | Toggle via `window.setStabDebugConstantPressure(true)` from devtools. |
| `DEBUG_ZOOM_COMPENSATION` logging | Inactive (const `false`) | Requires editing source to enable — not exposed as a runtime toggle. |
| `#fatal` "WebGPU isn't available" overlay | **Effectively unreachable** | Markup and CSS (`.show`) exist, but no code path calls `fatal.classList.add('show')` except the top-level generic error catch (which shows a *different* message: a script error, not the WebGPU-unavailable message). The WebGPU-unavailable case silently falls back to Canvas2D instead of showing this overlay. |
| Feed-point call diagnostics (first 3/stroke) | **Active, unconditional** | Console-only; no functional effect, but always runs. |
| Cadence-invariant check | **Active, unconditional** | Console-only; no functional effect but runs on every `endStroke`. |
| Finish diagnostics (`logFinishDiagnostics`) | **Active, unconditional** | Console-only; runs on every stroke completion. |
| Zoom-dependent hidden stabilization floor | **Active (production logic)** | Not experimental — this is load-bearing production behavior, just internally hidden from the UI. |
| Outline-only brush preview mode (`outlineOnly` param) | **Active, in use** | Used specifically for the live-drawing brush preview (`updateBrushPreview`, `drawing` branch) to show a ring rather than a filled dab while actively painting. |

---

## 9. State Management

### 9.1 `smoothBuf` (position moving-average FIFO)
Array of `{x,y}` objects, backing-store canvas space. Length dynamically
equals `movingAverageAmount()` (trimmed on push). Prefilled at
`beginStroke` to avoid a ramp-up period. Cleared in `resetStrokeState()`.

### 9.2 `pressureBuf` (pressure moving-average FIFO)
Array of numbers, same window-length semantics as `smoothBuf`, kept in
lockstep via the cadence-invariant counters.

### 9.3 Stroke state (scalars, mutated per-stroke)
`prevRaw`, `lastRaw`, `lastMid`, `lastR`, `lastAlpha`, `strokeMoved`,
`activePointerId`, `activeStrokeSessionId`/`nextStrokeSessionId` (monotonic
ownership tokens), `drawing` (bool).

### 9.4 GPU buffers/textures
- `accTex` — RGBA8 full-resolution accumulated canvas (the "true" image).
- `strokeMaskTex` — r8unorm per-stroke coverage scratch, cleared per stroke.
- `vertexBuf` — dynamically-grown vertex buffer for the current frame's batch.
- `strokeUniformBuf`, `compositeUniformBuf` — small per-draw uniform buffers.
- `undoStack` / `redoStack` — **arrays of full GPU textures** (WebGPU path)
  or full `ImageData` snapshots (Canvas2D path), capped at `MAX_UNDO = 12`.

### 9.5 Temporary caches / queues
- `pendingStrokeSamples` — queue of raw pointer samples awaiting the next
  per-frame flush.
- `smoothBuf`/`pressureBuf` — described above; functionally caches of recent
  history for the moving-average computation.

### 9.6 History
Undo/redo stacks only; there's no separate "stroke history" object model —
undo operates at the level of whole-canvas snapshots, not per-stroke command
objects, so undo always reverts an entire stroke's contribution in one step
regardless of how many dabs/segments it contained.

### 9.7 Global toggle/config state
`leashEnabled`, `tool`, `color`/`colorRgb`, `toolSizes` (per-tool
independent brush size), `opacity`, `stabilization`, `zoom`/`panX`/`panY`,
`canvasPixelated`/`sizePreviewPixelated`.

---

## 10. Architecture Diagram — Execution Flow

```
pointer event (pointerdown / pointermove / pointerup)
        │
        ▼
getCoalescedEvents() (batch expansion, if supported)
        │
        ▼
screenToCanvas() ── (pan, zoom, SS supersample factor applied)
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│ Jitter / idle / release-tail classification              │
│  - JITTER_FLOOR_PX (zoom-scaled)                          │
│  - isReleaseTailSample()                                  │
│  - lastContactPressure update                             │
└─────────────────────────────────────────────────────────┘
        │
        ▼
pushPressureBuf(pressure)  ──►  delayedPressure  (pressure moving avg)
        │
        ▼
pushSmoothBuf(rawPosition) ──►  stabilized position (position moving avg)
        │
        ▼
[long-jump subdivision if jumpDist > maxStep]
        │
        ▼
feedPoint(all, stabilizedPos, delayedPressure)
        │            │
        │            ├─► pressureRadius() / pressureAlpha()
        │            │
        ▼            ▼
drawQuadCurve() ── midpoint-quadratic Bézier tessellation (steps=10)
        │
        ▼
segmentVerts() / circleVerts() ── bounding-box quad + true shape params
        │
        ▼
drawVertsIntoAcc() ── writeBuffer(vertexBuf) + strokePipeline draw
        │                (renders into strokeMaskTex, coverage `max` blend)
        ▼
compositeStroke() ── compositePipeline: mix(baseTex, paintColor, coverage)
        │                writes into accTex
        ▼
present() ── blitPipeline: 4×4 box-filter downsample accTex → swap chain
        │
        ▼
screen (visible <canvas>)

Parallel/idle paths re-entering the same pipeline:
  strokeHoldTick (pen held still)  ──► pushSmoothBuf/pushPressureBuf(last input) ──► feedPoint ──► (same as above)
  finishTick (after pointerup)     ──► pushSmoothBuf/pushPressureBuf(frozen endpoint) ──► feedPoint ──► (same as above)
```

---

## 11. Dependency Map

### 11.1 Rendering / GPU subsystem
- **Inputs:** vertex float arrays (from Brush Engine), color/mode uniforms,
  `undoStack` textures (as composite base).
- **Outputs:** visible `<canvas>` pixels; `accTex` (persisted canvas state).
- **Calls:** `device.createShaderModule/RenderPipeline/Buffer/Texture`,
  `device.queue.writeBuffer/submit`.
- **Called by:** Brush Engine (`drawVertsIntoAcc`, `compositeStroke`,
  `present`), Undo/Redo (`snapshotCurrent`, `restore`, `clearCanvas`),
  startup (`clearTexToWhite`, initial `present()`).

### 11.2 Brush Engine
- **Inputs:** stabilized `(x,y)`/pressure samples from Stabilization,
  `getBrushSize()`, `color`/`opacity` from UI state.
- **Outputs:** vertex arrays fed to Rendering.
- **Calls:** `pressureRadius`, `pressureAlpha`, `drawQuadCurve`,
  `segmentVerts`/`circleVerts`, `drawVertsIntoAcc`.
- **Called by:** `processStrokeBatch`, `strokeHoldTick`, `finishTick`,
  `beginStroke` (initial dab).

### 11.3 Stabilization
- **Inputs:** raw pointer samples (via Input System), `stabilization` (UI
  slider), `zoom` (Zoom System).
- **Outputs:** stabilized position/pressure fed to `feedPoint` (Brush
  Engine); leash visualization pixels.
- **Calls:** `computeInternalStabilization`, `zoomStabilizationMinimum`,
  `zoomCompensationWeight`, `movingAverageAmount`.
- **Called by:** `processStrokeBatch`, `strokeHoldTick`, `finishTick`,
  `computeWasStoppedBeforeLift`, `updateBrushPreview` (radius display),
  `showSizePreview`.

### 11.4 Input System
- **Inputs:** browser `PointerEvent`s, `getCoalescedEvents()`.
- **Outputs:** canvas-space coordinates + pressure to Stabilization/Brush
  Engine; hover state to UI (brush preview).
- **Calls:** `screenToCanvas`, `getPressure`, `beginStroke`/`moveStroke`/
  `endStroke`, pan/zoom-drag/size-drag handlers.
- **Called by:** browser event dispatch only (top-level).

### 11.5 Zoom System
- **Inputs:** wheel events, zoom buttons, zoom-drag gesture.
- **Outputs:** `zoom`/`panX`/`panY`; feeds Stabilization's hidden floor and
  Rendering's pixelation-hysteresis state.
- **Calls:** `applyTransform`, `updatePixelatedState`, `resizeStabViz`.
- **Called by:** wheel listener, zoom buttons, `fitToScreen`, window resize.

### 11.6 Performance / Batching
- **Inputs:** `pendingStrokeSamples` (from `moveStroke`).
- **Outputs:** drained batches to `processStrokeBatch`.
- **Calls:** `processStrokeBatch`.
- **Called by:** `strokeHoldTick` RAF loop.

---

## 12. Public Constants (grouped)

### Stabilization
- `HOLD_BEFORE_LIFT_MS = 120` — idle duration before a stroke is classified
  "stopped before lift."
- `CONTACT_PRESSURE_FLOOR = 0.02` — minimum pressure to count as genuine
  contact (vs release artifact).
- `MIN_FINISH_MS = 80`, `MAX_FINISH_MS = 260` — finish catch-up animation
  duration bounds.
- Finish pacing: `startRatePerMs = avgTicksPerMs * 0.6`,
  `endRatePerMs = avgTicksPerMs * 1.6` (derived, not literal constants, but
  fixed multipliers).

### Zoom-compensation (stabilization ↔ zoom)
- `ZOOM_COMP_MIN_ZOOM = 0.05` (5%)
- `ZOOM_COMP_ZERO_ZOOM = 5.0` (500%)
- `ZOOM_COMP_MAX_AMOUNT = 0.15`
- `ZOOM_COMP_FADE_UI_LIMIT = 0.20` (20%)

### Rendering
- `SS = 4` — backing-store supersample factor.
- `AA_MARGIN = 2.0` — bounding-quad padding, backing px.
- `W, H = 1920, 1080` — logical canvas size.
- `BW, BH = W*SS, H*SS` — backing-store size (7680×4320).

### Spacing / geometry
- `maxStep = max(2, scaledBrushSize() * 0.6)` — long-jump subdivision
  threshold (derived, not a literal top-level constant).
- Bézier tessellation `steps = 10` (literal argument at each `drawQuadCurve`
  call site, not centralized as a named constant).

### Zoom (pan/zoom interaction)
- `PIXELATE_IN_ZOOM = 1.2` (120%)
- `BLUR_OUT_ZOOM = 1.0` (100%)
- Zoom clamp: `[0.05, 30]` (5%–3000%), applied at wheel/buttons/drag.
- Wheel zoom factor: `1.08` per notch.
- Button zoom factor: `1.15` per click.
- Zoom-drag factor base: `1.006^dx`.

### Pressure
- `MIN_SIZE_PERCENT = 0` — pressure can reach a true zero-relative diameter.
- `MIN_ABS_RADIUS_BACKING = 0.05 * SS` — absolute minimum dab radius floor.
- Pressure ease-in exponent: `1.2` (`pressureInfluence`).
- `PREVIEW_MIN_PRESSURE = 0.3` — floor applied only to the hover preview's
  displayed pressure, not to actual drawing.

### GPU
- `vertexBuf` initial size: `4 * 1024 * 1024` bytes (4MB), doubling growth.
- `strokeUniformBuf` size: 32 bytes; `compositeUniformBuf` size: 16 bytes.

### Debug
- `DEBUG_ZOOM_COMPENSATION = false`
- `DEBUG_CONSTANT_PRESSURE = false` (mutable via `window.setStabDebugConstantPressure`)
- `MAX_UNDO = 12`

---

## 13. Cleanup Opportunities (identified only — no code changed)

1. **Unused WASM module.** The entire base64 WASM blob and its instantiation
   (~40KB+ of markup, decode, and `WebAssembly.instantiate` call) appears to
   serve no active purpose — `readVerts()` (its only consumer) is never
   called. Either wire it back in or remove it; shipping it as-is adds
   startup cost and a misleading `#hint` UI string ("brush = WASM") for zero
   functional benefit.

2. **Unreachable `#fatal` "WebGPU isn't available" UI.** The dedicated
   WebGPU-unavailable messaging (markup + CSS) is never triggered by the
   actual WebGPU bring-up failure path, which silently falls back to
   Canvas2D instead. Either this overlay is dead code, or the fallback
   behavior is intentional and the overlay should be removed/repurposed.

3. **Unconditional per-stroke diagnostic logging.** `[stabilizer cadence]`
   and `[stabilizer finish]` console output fires on literally every stroke,
   with no debug-flag gate. Combined with the WebGPU/WASM bring-up
   `console.log` chatter (also unconditional), this will spam production
   consoles. Recommend gating all of it behind a single `DEBUG` flag
   (mirroring the existing `DEBUG_ZOOM_COMPENSATION`/
   `DEBUG_CONSTANT_PRESSURE` pattern) or removing entirely.

4. **Explicitly-labeled temporary diagnostics.** The `__feedPointCallIndex`
   first-3-calls logging and the position/pressure cadence counters are
   self-documented in comments as temporary, to be removed once specific
   investigations (startup-kink, pairing verification) are "confirmed" /
   "settled." These investigations appear complete based on surrounding
   comments describing settled conclusions — worth confirming with whoever
   owns this file, then deleting.

5. **Large undo/redo GPU memory footprint.** `MAX_UNDO = 12` full-resolution
   (`BW×BH` = 7680×4320) RGBA8 textures is a substantial VRAM commitment
   (~1.6GB worst-case at 12 full snapshots) with no downsampling or delta
   compression. Worth evaluating before shipping on memory-constrained
   devices (especially mobile/Android Chrome, explicitly called out as a
   supported target in the `#fatal` copy).

6. **Duplicate coalesced-event handling logic.** The pattern `(typeof
   e.getCoalescedEvents === 'function' && e.getCoalescedEvents().length) ?
   e.getCoalescedEvents() : [e]` is repeated verbatim in `processStrokeBatch`
   and `moveStroke`. A small shared helper would reduce duplication.

7. **Canvas2D fallback lacks feature parity by design**, which is fine as a
   fallback, but there is no code path currently exercising it during normal
   development (since WebGPU is required for the stabilization slider/leash
   toggle to even be enabled — both are explicitly `disabled` when
   `!useWebGPU`, lines 2246/2253). If the fallback is meant to remain
   permanently minimal, that should be an explicit, documented product
   decision before integration, not an implicit consequence of prototype
   scope.

8. **Debug hook exposed globally.** `window.setStabDebugConstantPressure` is
   attached to the global `window` object unconditionally. Fine for a
   prototype; should likely be namespaced or removed for a production
   integration to avoid polluting global scope / accidental invocation.

---

## 14. Overall Architecture Summary (for a new developer)

Think of this file as two mostly-independent halves glued together by a
shared set of pointer-input/UI-state variables:

**Half one** is a fairly conventional WebGPU 2D painting renderer: a
supersampled RGBA accumulation texture (`accTex`), a per-stroke coverage
scratch texture (`strokeMaskTex`), three small pipelines (stroke rasterize →
composite → blit-present), and GPU-native undo/redo via texture copies. If
you've built a WebGPU or OpenGL painting app before, this part will feel
familiar; the one genuinely clever trick is that dab/segment geometry is sent
down as *bounding quads carrying true shape parameters*, with the fragment
shader computing an exact analytic signed distance to the real capsule shape
— this avoids the classic "blurry overlapping stroke" problem that comes
from pre-extruding soft edges into geometry and letting them accumulate.

**Half two** — and the actual point of this prototype, judging by the file
name and the density of comments — is the stabilization/smoothing pipeline.
Its entire job is to take noisy, discrete pointer samples and produce a 
smooth and lagged stroke, with known geometric contraction on curved paths 
at large window sizes, using nothing more exotic than a fixed-length 
moving-average FIFO applied independently to position and
pressure. What makes it non-trivial is everything *around* that simple core:
keeping position and pressure buffers in lockstep; making the "catch-up"
after the pen stops or lifts feel natural (frame-rate-independent hold
catch-up, velocity-paced finish catch-up); correctly distinguishing "the pen
paused and then lifted" from "the pen was still moving when it lifted" so
release pressure behaves believably in both cases; filtering out
tablet-driver release artifacts (decaying pressure just before liftoff) so
they don't get mistaken for the artist genuinely easing off; and
compensating for the fact that a fixed screen-space smoothing window
represents wildly different amounts of *canvas*-space smoothing depending on
current zoom. None of these are decorative — each is guarding against a
specific, previously-observed visual artifact (bow-tie fragments, ghost
diagonal lines, jagged post-lift strokes, jittery zoomed-out drawing), as the
comments repeatedly document.

The rest of the file — pan/zoom/size-drag gestures, tool/color/size state,
undo/redo, PNG export — is standard app chrome and is not architecturally
interesting relative to the two halves above.

**For integration, prioritize in this order:**
1. Decide the fate of the unused WASM module (§13.1) — likely remove.
2. Gate or remove unconditional debug logging (§13.3, §13.4) before this
   ships anywhere near production console output.
3. Confirm the undo/redo VRAM budget (§13.5) is acceptable for target
   devices, especially mobile.
4. Decide whether Canvas2D fallback needs to gain stabilization feature
   parity, or whether "no WebGPU = no stabilization" is an accepted product
   constraint.
5. The stabilization/finish/hold state machine (§3) is the most valuable and
   most fragile part of this file — the session-token ownership guard
   pattern (§2.1, §3.7) is load-bearing correctness logic, not incidental,
   and should be preserved carefully during any refactor.
