# Brush Pipeline Audit — `behind-me-is-winner`
Read-only investigation. No files modified. All line numbers refer to the repo at the commit cloned for this audit.

Primary file: **`src/brush/brush-engine.js`** (4498 lines — the entire brush/eraser/line/curve stroke pipeline lives here as one file).
Supporting files: `src/ui/panels.js` (ensureKey/saveActiveToKey/recomposite), `src/ui/tools-color.js` (pushUndo), `src/brush/brush-presets.js` (settings ↔ `window._ts*` binding), `src/core/core-state.js` (`activeC`, `ctx`, `CW/CH`, `curLayer`, `curFrame`, `layers`).

---

## 1. True stroke entry point

- **Listener:** `activeC.addEventListener('pointerdown', _brushPointerDown)` — `brush-engine.js:4247`.
  - A second listener, `canvasArea.addEventListener('pointerdown', ...)` at `4258`, forwards off-canvas presses (panning margin, overlay canvases) into the *same* start logic for `tool==='brush'|'eraser'` only.
- **Function called:** `_brushPointerDown(e)` — `brush-engine.js:4093`.
- **Active tool check:** `4102` — `if(tool!=='brush'&&tool!=='eraser'&&tool!=='fill'&&tool!=='line'&&tool!=='curve') return;`. `tool` is a module-level global set elsewhere (setTool, not in this file).
- **Layer/frame/cel selection:** Not selected here — `curLayer`/`curFrame` are pre-existing globals (`core-state.js`) that the stroke simply reads/tags. Frozen into stroke ownership at `4165`: `_strokeOwnerLayer=curLayer;_strokeOwnerFrame=curFrame;`.
- **Undo capture:** `pushUndo()` at `4190`, once per stroke, *after* the early-return branches (fill/line/curve have their own handling — fill pushes its own undo at `4163`; line/curve defer undo to `_pointerEndStroke`/`_commitCurveTool`).
- **Stroke ownership begins:** `_activeStrokePointerId=e.pointerId;` (`4164`), `_activeStrokeSession=++_strokeSessionSerial;` (`4165`) — a monotonically increasing session id used later to reject stale async work (RAF callbacks, stabilizer finalizers) from a superseded stroke.
- **PointerId/stylus ownership tracking:** `activeC.setPointerCapture(e.pointerId)` (`4186`). All subsequent move/up handlers verify `e.pointerId===_activeStrokePointerId` before acting (`4346`, `4408`).

Guard conditions before any of this runs (`4097–4104`): blocks if a layer group is active, panning, zoom-dragging, space-held pan, transform tool active, wrong mouse button, eyedropper tool (routed separately), unsupported tool, hidden drawing frame, or locked layer.

Notably, `_brushPointerDown` also defensively **finishes or ends any still-live previous stroke** before starting a new one (`4110–4117`) — see §17 (compatibility risks) and §9.

---

## 2. Live pointer movement — exact execution order

```
pointermove / pointerrawupdate
  → activeC event listener (4394 / 4400)
    → _handleMoveEvent(e)                              [4341]
       → events = e.getCoalescedEvents() || [e]         [4349]
       → for each raw sample ev in events:
          → newPressure = _getPressure(ev)              [3582]
          → raw = getPos(ev)                             [5]      (screen→canvas coords)
          → conditionedSamples = _baselineConditionerPush(
                _baselineSampleFromEvent(ev,raw,newPressure))     [2356 / 2349]
             (canonical resampling: corner detection, duplicate rejection,
              distance-based emission, coarse time-gap fallback)
          → for each conditioned sample:
             → _stabilizerSetSampleContext(pressure,event)        [2409]
             → p = _stabilizePoint(x,y,t)                          [2367]
                  (first-order low-pass + spatial leash)
             → _updateVelocity(p.x,p.y,t)                          [3624]
             → _curveAddPoint(p.x,p.y,pressure,event)              [3095]
                  → (low-zoom only) Savitzky–Golay reconstruction  [3054–3107]
                  → _curveAddReconstructedPoint(x,y,pressure,e)    [3030]
                       → _stampQuadCurve(...)                       [2919]
                            → walks the curve, emits dabs           → _queueDab [2200]
                                 → _drawDabNow / rasterization      [2166 / 1606 / 1258…]
                                      → paints onto _strokeCanvas (offscreen scratch)
             → currentPressure, lx, ly, _lastPointerEvent updated
    → _scheduleRecomposite()                                       [3400]
         → recomposite(curLayer, curFrame, dirtyRect)               [panels.js:337]
              → reads _getLiveStrokePreview() for the active layer  [506]
              → draws into compCtx/artworkCompositeCtx
              → presented to visible <canvas> (compC)
```

Per-stage detail:

| Stage | File:Fn | Input | Output | State changed | Next |
|---|---|---|---|---|---|
| Raw event fan-out | `4341 _handleMoveEvent` | PointerEvent | list of coalesced PointerEvents | none | `getPos`/`_getPressure` per event |
| Coordinate conversion | `5 getPos` | client X/Y + `activeC`/`canvasArea` rect, pan/zoom/rotation/flip state | `{x,y}` canvas-space float | none (pure) | conditioner |
| Sample conditioning | `2356 _baselineConditionerPush` | `{x,y,pressure,tiltX,tiltY,twist,time,event}` | 0..N canonical samples | `_baselineConditionerState` (rolling last-raw/last-forwarded/carry) | stabilizer |
| Stabilization | `2367 _stabilizePoint` | conditioned x,y,t | filtered `{x,y}` | `_stabilizerX/Y`, `_stabilizerTargetX/Y`, `_stabilizerSpeed`, schedules `_stabilizerRAF` | velocity/curve |
| Velocity | `3624 _updateVelocity` | filtered x,y,t | none (side effect only) | `_strokeVelocity` (EMA) | curve add |
| Curve reconstruction | `3095 _curveAddPoint` → `3030 _curveAddReconstructedPoint` | filtered point+pressure | none directly | `_curveP0/_curveP1/_curvePr0/_curvePr1` rolling buffer | `_stampQuadCurve` |
| Geometry/dab walk | `2919 _stampQuadCurve` → `2745 _strokeSegment`-family | midpoint-quadratic control points + pressure | per-dab `{x,y,r,alpha,rotation,...}` | `_strokeSegCarryOver`, `_strokeDistSoFar`, `_strokeDabCount` | `_queueDab` |
| Dab rasterization | `2200 _queueDab` → `2166 _drawDabNow` → `1606 _dabAA`/`1685 _dabAliased` | dab params | pixels written | `_strokeCanvas` (offscreen), `_frameDirty`/`_strokeDirty` rects | recomposite scheduling |
| Recomposite scheduling | `3400 _scheduleRecomposite` | dirty rect / firstDab flag | schedules or runs synchronously | `_recompRAF`, `_recompGeneration` | `recomposite()` |
| Compositing | `panels.js:337 recomposite` | `curLayer`,`curFrame`,dirtyRect | pixels into `compC`/on-screen canvas | none persistent (reads `_getLiveStrokePreview()` for active layer mid-stroke) | screen paint |

---

## 3. Input handling

- **pointermove vs pointerrawupdate:** Feature-detected once — `const _hasRawUpdate = 'onpointerrawupdate' in window;` (`4393`). If available, **pen** input is handled *exclusively* via `pointerrawupdate` (`4400`); the plain `pointermove` listener explicitly ignores pen events in that case (`4396`) to avoid double-stamping (documented at `4385-4392`). Mouse/touch always use `pointermove`.
- **`getCoalescedEvents()`:** used at `4349` inside `_handleMoveEvent`, shared by both listeners, falling back to `[e]` if unsupported/empty.
- **Pointer capture:** `activeC.setPointerCapture(e.pointerId)` on down (`4186`); released on `pointerup`/loss (`4494`, `lostpointercapture` handler `4497`).
- **PointerId ownership:** `_activeStrokePointerId` gate in `_handleMoveEvent` (`4346`) and `_pointerEndStroke` (`4408`) — a second concurrent pointer is ignored for move/up purposes while a stroke owns the canvas.
- **Palm/second-pointer handling:** No explicit palm-rejection heuristic found. The only protection is single-pointer ownership via `_activeStrokePointerId`; a second finger/pointer touching the canvas mid-stroke would hit `_brushPointerDown` again, which (per §1) force-ends the prior stroke and starts a new one from the new pointer. No `touch-action`/pen-only gating was found in this file (CSS may restrict this — outside scope of this audit).
- **Screen→canvas conversion:** `getPos(e)` (`5`) — inverts flip mirror around nav pivot, then rotation/pan/zoom.
- **Pan/zoom/rotation transform:** handled inside `getPos` via `getNavPivot()`, `flipX/flipY`, `rotation`, `panX/panY`, `zoom` (all external globals from the camera/view system).
- **Supersampling/backing-scale:** Not visible in this file; `activeC.width/height` (used to size `_strokeCanvas`) are assumed to already be backing-store pixel dimensions (`CW`/`CH`), i.e. conversion happens upstream in canvas setup, not in the brush pipeline itself.
- **Pressure/tilt/twist/buttons:** `_getPressure(e)` (`3582`) reads `e.pressure`; tilt/twist are captured (not used for geometry) in `_baselineSampleFromEvent` (`2349`: `tiltX`, `tiltY`, `twist`/`rotationAngle`) — currently only pressure feeds size/opacity (§5); tilt/twist are plumbed through but not consumed downstream in this file.
- **Mouse fallback:** `_getPressure` returns `1.0` for mouse always (`3615`); touch force falls back to `1.0` if unsupported (`3612`).

**Answers:**
- First canonical canvas-space position: the return of `getPos(e)` inside `_brushPointerDown` at `4153`, fed straight into `_resetStabilization(p.x,p.y,...)` (`4158`).
- Coordinate units after conversion: canvas-space **float** pixels (document/artwork pixel space, post `/zoom`), not screen pixels.
- Float vs rounded: consistently float throughout the geometry/stabilization/pressure pipeline; no rounding until actual pixel rasterization inside dab-drawing helpers (outside this audit's line-by-line trace, but implied by canvas `drawImage`/pixel APIs).
- Driving events: `pointerdown`, then either `pointermove` (mouse/touch, and pen when `pointerrawupdate` unsupported) or `pointerrawupdate` (pen when supported), terminated by `pointerup`/`pointercancel`/`lostpointercapture`.

---

## 4. Current position stabilization

Two independent, chained smoothing stages exist. Do not conflate them:

**Stage A — Sample conditioner (`_baselineConditionerPush`, always active, not user-controlled):**
Resamples raw device events into a canonical stream at a fixed screen-space step (`_baselineCanonicalStepScreenPx()`, `0.5px`–`2px` depending on zoom, `2335-2345`), with corner-preserving emission (`_baselineIsCorner`, `2355`), duplicate rejection, and a 32ms max-gap fallback. This is **not** the user-facing "Stabilization" slider — it runs unconditionally, at any stabilization amount including 0%, and exists purely to normalize sampling density/rate.

**Stage B — User stabilizer (`_stabilizePoint`, `2367`, driven by the Stabilization slider):**
- Source: `window._tsStabilization` (0–1), bound from a DOM range input in `brush-presets.js:184-186`.
- Internal amount mapping: `_stabilizationAmount()` (`2249`) clamps to `[0,1]`, and **treats 0 as "1" (max) via `amount<=0?1:amount`** — but only *after* the bypass check in `_stabilizePoint` itself, see below. `_stabilizerStrength(amount)` (`2286`) applies a smoothstep curve.
- No deadzone. No moving-average/window buffer. This is a **first-order exponential low-pass filter** (`alpha=1-exp(-dt/tau)`, `2389`) with a **spatial leash** clamp (`2393-2402`, `_stabilizerMaxLagCanvas`) that prevents the filtered point from lagging the raw point by more than a zoom-scaled screen-pixel bound (`0.20px`–`16px` depending on amount, `2267-2268`). This is explicitly *not* a lazy-brush/leash-drag tool — every real sample advances the filter immediately (per code comment at `2255-2260`); there is no velocity/inertia term.
- Distance/time/count basis: **time-based** (`dt` from event timestamps, clamped `0.00025s–0.05s`, `2380`), not sample-count or pure-distance based, though the leash bound is spatial.
- Velocity-dependent: yes — `_stabilizerEffectiveTau` (`2294`) shortens the effective time-constant as `_stabilizerSpeed` rises, so fast strokes/reversals aren't pulled inward as much.
- Zoom-dependent: yes — the leash (`_stabilizerMaxLagCanvas`, `2304`) converts a fixed screen-pixel bound into canvas space via `/zoom`.
- State/buffers maintained: `_stabilizerX/Y` (filtered position), `_stabilizerTargetX/Y` (raw target), `_stabilizerRawX/Y`, `_stabilizerSpeed`, `_stabilizerLastSampleT/AdvanceT/InputWallT`, an idle-driven `requestAnimationFrame` loop (`_stabilizerStep`, `2457`) that keeps advancing the filter toward the target even without new input, until convergence (see §8, hold behavior).
- Reset at stroke start: `_resetStabilization(x,y,t)` (`2309`), called from `_brushPointerDown` (`4158`).

**Is stabilization active at UI 0%?** Ambiguous by design/comment vs. actual code path:
- `_stabilizationAmount()` maps `0 → 1` (i.e. "no explicit value" defaults to full amount) — but this only matters for the `amount` passed *into* the tau/leash math.
- However, `_stabilizePoint` has an **explicit true-bypass branch** at `2369`: `if(amount<=0){ ...return{x,y}; }`. Since `_stabilizationAmount()` never actually returns `≤0` (it coerces `0→1`), **this bypass branch is currently dead code** — at UI-slider 0%, `_stabilizationAmount()` returns `1` (max stabilization), not `0`. **This is worth flagging as a probable existing bug/mismatch, not something to "fix" during migration** — the safest interpretation is that "true 0% bypass" may not be reachable through the UI as-is; only Stage A (the always-on conditioner) would be active in what the user perceives as "no stabilization." Confirm against `bindRange` before assuming intended behavior.

**Function boundaries requested:**
- Receives raw canvas point: `_stabilizePoint(x,y,t)` — but `x,y` here are *already conditioned* (post `_baselineConditionerPush`), not the truly raw `getPos()` output.
- First changes it: `_baselineConditionerPush` (resampling) is technically first; the first *smoothing* transform is `_stabilizePoint`.
- Final stabilized `{x,y}`: return value of `_stabilizePoint` (`2407`), further passed through `_curveAddPoint`'s optional low-zoom Savitzky–Golay reconstruction (`3095`) before becoming actual curve control points — so geometrically the *truly final* position used for dab placement can differ slightly again after `_curveBaselineEmit`/SG smoothing at low zoom.

---

## 5. Pressure pipeline

- **Raw pressure reader:** `_getPressure(e)` (`3582`). Pen: trusts `e.pressure>0`; treats a reported `0` while `drawing` as a driver glitch and holds `_lastKnownPressure` instead of snapping (`3594-3598`). Also **rate-limits jump size** via `_MAX_PRESSURE_JUMP=0.09` per raw sample (`3579`, applied `3602-3605`), except on the stroke's first sample (`_strokeFirstSample`, snaps immediately, `3599`).
- **Pressure curve:** `_getPressureCurve(setting)` (`3664`) reads a per-setting (`size`/`opacity`/etc.) curve selector from the DOM (`ts-<setting>-pressure-curve`), resolving to one of `PRESSURE_CURVES` (`linear/soft/hard/s`, `3678`) or a custom Bézier (`window._tsCustomPressureCurves`). Applied via `_applyPressureCurve` (`3715`) → `_evalPressureCurveY`/`_bezierPointAt`.
- **Minimum size/flow:** `_getMinSize()` / `_getMinFlow()` (`3659-3660`), read from `ts-min-size` / `ts-min-flow` DOM inputs.
- **Size-pressure toggle:** `_getSizeControl()` (`3656`), default `'pressure'`.
- **Opacity/flow-pressure toggle:** `_getOpacityControl()` (`3658`, default `'pressure'`) / `_getFlowControl()` (`3657`, default `'off'`).
- **Pressure smoothing/buffering:** exponential smoothing, `_smoothedPressure` with `_PRESSURE_SMOOTH=0.25` (`3568,3579` region, applied at `3730` inside `_resolveControl('pressure',...)`). Distinct from the jump-clamp in `_getPressure`.
- **Release-tail filtering:** handled via end-of-stroke taper machinery (`_getEndTaper`, `_beginEndTaperCapture` at `382`) rather than pressure filtering per se; also `_stabilizerFinalize`/`_flushCurveTail` ensure the final true pointer-up position/pressure is emitted (§9).
- **Mouse pressure:** constant `1.0` (`3615`) — the stroke-start taper (`_strokeTaperFactor`, `2642`, distance-driven not pressure-driven) is what gives mice a natural-feeling stroke-start ramp despite constant pressure.
- **Where pressure becomes radius/alpha:** `_getEffectiveBrushParams(e)` (`3885`) → `_computeEffectiveParams` (`3746`) → resolves `r` (radius) and `alpha`, consumed by `_stampDab` (`2514`, calls `_getEffectiveBrushParams` at `2517`).

**Answers:**
- Position and pressure synchronization: pressure travels **attached to each conditioned sample** (`_baselineSampleFromEvent`/`_baselineConditionerPush` carries `.pressure` alongside `.x/.y`), and both are consumed together by `_stabilizerSetSampleContext`/`_curveAddPoint`. However, **stabilization filters position only** — `_stabilizerEmit` (`2416-2422`) explicitly notes "Stabilization owns position only. Pressure remains the raw pressure already used by the normal reconstruction/interpolation pipeline," using `_stabilizerTargetPressure` (the latest raw/conditioned pressure, unfiltered) rather than a filtered pressure value.
- Can position and pressure advance at different cadences? Yes, structurally: pressure updates are keyed to whichever conditioned sample last set `_stabilizerTargetPressure`, while position is continuously advanced by the RAF-driven stabilizer loop even between new samples (see §8) — meaning during a hold-still gap, position converges toward a fixed target while pressure stays pinned at the last sampled value (not itself animated).
- Functions that must not be replaced/altered casually during migration: `_getPressure` (device-quirk handling is tablet/driver-specific tribal knowledge), `_applyPressureCurve`/`PRESSURE_CURVES` (user-visible curve shapes, must stay pixel-identical to the Tool Settings panel preview which shares this exact table), and `_MAX_PRESSURE_JUMP` clamp semantics (anti-glitch behavior tuned per comment history).

---

## 6. Curve and brush geometry

Real-project equivalents of the requested primitives:

| Prototype-style name | Real-project function | Location |
|---|---|---|
| `feedPoint()` | `_curveAddPoint` (outer) / `_curveAddReconstructedPoint` (inner, real curve feed) | `3095` / `3030` |
| `drawQuadCurve()` | `_stampQuadCurve` | `2919` |
| segment/dab generation | `_strokeSegment`, `_strokeSegmentProfile`, `_walkDabArc` | `2745`, `2738`, `2706` |
| spacing | `_effectiveSpacingFrac`, `_effectiveDabStep`, `_computeSpacingRadius` | `2653`, `2673`, `3895` |
| interpolation | midpoint-quadratic construction inside `_curveAddReconstructedPoint` (`3037-3038`: `startPt=(A+Bc)/2`, `endPt=(Bc+C)/2`) | `3030` |
| long-jump subdivision | `_walkDabArc` (arc-length dab walking along the quad curve) + `_quadArcTable`/`_quadPointAtLength` (`2860`, `2897`) | `2706` |

**Trace:** stabilized position + pressure → `_curveAddPoint` → (optional SG reconstruction at low zoom, `3054-3107`) → `_curveAddReconstructedPoint` builds a **3-point rolling buffer** (`_curveP0/_curveP1`, new sample `C`) and emits a **midpoint-quadratic Bézier segment** per new point (classic "quadratic through midpoints" curve smoothing, same family as the Catmull-Rom-via-midpoints technique many painting apps use) via `_stampQuadCurve` → which arc-length-walks the quad curve and calls `_queueDab` once per spacing interval → `_drawDabNow`/AA rasterizers → `_strokeCanvas`.

**Answers:**
- Geometry type: **midpoint quadratic Bézier segments**, not raw polyline, not full cubic Bézier, not simple stamping-only. (Cubic Bézier math (`_bezierPointAt`, `3686`) exists in the file but is used for **pressure-curve evaluation**, not stroke geometry.)
- Event-driven or distance-driven: **distance-driven** for dab placement (arc-length walk with adaptive step ~12% of current effective diameter, `2623-2629` comment), independent of how many pointer events triggered a given curve segment.
- Spacing/tip shape applied: spacing in `_effectiveSpacingFrac`/`_effectiveDabStep`; tip shape (procedural round vs. custom tip image) resolved inside `_getEffectiveBrushParams`/`_stampDab` and rasterized in `_buildTipStamp`/`_buildSoftRoundMask`/`_drawUnifiedTipStamp` (`994`, `963`, `1238`).
- Final geometry entry point: `_queueDab(dabParams)` (`2200`) is the single funnel through which every dab — from pointerdown's first stamp, movement-driven curve walking, and airbrush timer stamps — enters rasterization.

---

## 7. Rendering destination

- **Live/in-progress stroke buffer:** `_strokeCanvas`/`_strokeCtx` (offscreen `<canvas>`, allocated per-stroke in `_ensureStrokeCanvas`, `392`). All dabs during a stroke are painted here, **not** onto `activeC`, so stroke-level Opacity/Flow can be applied once at commit (comment at `484-497`).
- **Live preview compositing:** `_getLiveStrokePreview()` (`506`) blends `activeC` + `_strokeCanvas` (at `brushOpacity`) into `_strokePreviewCanvas`, which `recomposite()` substitutes in place of `activeC` for the active layer while `_inStroke` is true (`panels.js:390-394`).
- **Commit:** `_commitStrokeCanvas()` (`453`) — draws the (optionally textured/selection-clipped) `_strokeCanvas` onto `ctx` (== `activeC`'s 2D context) using `_brushPaintCompositeOperation()` for blend mode (`414`), at `globalAlpha=brushOpacity`. For eraser tool, `_stampDab` sets `composite:'erase'` per-dab instead (rasterized with erase semantics, not via this blend-mode composite path — see `2523`).
- **Active layer / active frame routing:** the true persisted destination is **not** `activeC` — `activeC` is a **working/scratch canvas for the currently-selected layer+frame**. Persistence into the actual layer/frame data model happens via `saveActiveToKey()` (`panels.js:300`), called after `_commitStrokeCanvas()` at every stroke-end path (`4443`, `4459`, `4333`, `3492`). This copies `activeC` into `layers[curLayer].frames[curFrame]` (a per-key canvas), or into an "extended" (larger-than-canvas) frame record if one exists.
- **Smart-raster vs normal raster:** detected via `layers[curLayer].type==='smart-raster'`. When true and an advanced palette style is active, `_commitStrokeCanvas` additionally calls `commitSmartRasterBrush(src,styleId,brushOpacity,ownershipDirtyRect,ownershipBefore,brushBlendMode)` (`474-478`, defined in `src/smart-raster/`) to update style/palette ownership metadata over the dirty region — this is **layered on top of**, not instead of, the normal pixel commit.
- **Style/palette mapping:** `activeAdvancedStyleIdForPainting()` (external function) resolves the active style id used both for eraser color-mode routing (`336-340`) and smart-raster ownership commit (`462-463`).
- **WebGPU/WebGL/Canvas2D:** Canvas2D only — every context obtained in this file is `getContext('2d', ...)` (e.g. `400`, `268` in core-state.js). No WebGL/WebGPU path exists for brush rendering.
- **Blend mode:** `_brushPaintCompositeOperation()` (`414`) maps a `window.brushBlendMode` string to a canvas `globalCompositeOperation`, only for `brush`/`line`/`curve` tools (`_usesBrushPaintPipeline()`, `413`); eraser always uses erase semantics regardless of blend mode setting.
- **Eraser behavior:** routed at the dab level (`composite:'erase'` in `_stampDab`, `2523`), plus a distinct **color eraser** sub-mode (`_beginColorEraserStroke`/`_captureColorEraserDab`/`_endColorEraserStroke`, `332-392`) for smart-raster style-aware erasing.
- **Antialiasing:** dab-level AA via `_dabAA`/`_dabAACpu`/`_dabAAGpu`/`_dabAATinyCoverage` (`1606,1357,1258,1430`) vs. `_dabAliased` (`1685`) for hard-edge mode; separately, line/curve tools have their own AA quality settings (`_normalizeLineAAQuality`, `273`).
- **Final compositing into the document:** `recomposite(li,fi,dirtyRect)` (`panels.js:337`) — draws every visible layer bottom-to-top into `compCtx`/`artworkCompositeCtx`, substituting the live-stroke preview for the active layer mid-stroke, and the plain `activeC` once no stroke is in progress.

**Answers:**
- True stroke destination: **`_strokeCanvas`** during the stroke; **`activeC`** at commit; **`layers[curLayer].frames[curFrame]`** (persisted key canvas) after `saveActiveToKey()`.
- In-progress stroke buffer exists: yes (`_strokeCanvas`, offscreen, cleared/reused per-stroke).
- Stroke committed permanently: at `saveActiveToKey()` — called immediately after `_commitStrokeCanvas()` on every completion path (normal pointerup, `_endStroke` cancellation-that-still-had-ink, line/curve commit).
- What must be preserved: the `_strokeCanvas → activeC → layers[...].frames[...]` three-tier flow, the `extended`-frame (canvas-larger-than-document) mechanism in `saveActiveToKey`/`getExtendedLayerFrame`, and the smart-raster ownership commit hook — a migrated engine that writes straight to `activeC` without going through an equivalent `_strokeCanvas`+opacity-composite stage would break stroke-level Opacity/Flow, and skipping `saveActiveToKey` would break animation-frame persistence entirely.

---

## 8. Hold behavior

Two independent hold mechanisms exist:

**A. Stabilizer catch-up RAF (always relevant whenever Stabilization > effectively-0):**
`_stabilizerSchedule()` (`2328`) arms `requestAnimationFrame(_stabilizerStep)` whenever the stabilizer is active and no RAF is already pending. `_stabilizerStep` (`2457`):
- Waits out an **18ms idle delay** (`_STABILIZER_IDLE_DELAY_MS`, `2269`) after the last real input before beginning to "catch up" — this avoids animating on every single high-frequency sample.
- Then repeatedly calls `_stabilizerAdvance(dt,now)` (`2425`), each call moving `_stabilizerX/Y` toward the (unchanging, since pen is still) target by the exponential filter, **emitting a real dab** via `_stabilizerEmit` (`2416`) every tick — so **yes, the brush tip visibly catches up to a stationary pen**, and rendering (`_scheduleRecomposite`) continues each tick until convergence (`gapCanvas` below `_STABILIZER_EPS_SCREEN_PX=0.30px`, `2438`).
- Pressure does **not** continue changing during this catch-up (`_stabilizerTargetPressure` stays pinned to the last sampled value; `_stabilizerEmit` reuses it unchanged).
- Stop condition: convergence within epsilon, or (during finish) invoking the pending finalize callback.

**B. Airbrush continuous spray (only when `window._brushAirbrush && window._brushContinuousSpraying`):**
`_startAirbrushSpray()` (`2593`), called from `_brushPointerDown` (`4245`) if airbrush+continuous-spray flags are set. Runs a `setInterval` (not RAF) at `_airbrushIntervalMs()` (`6ms–50ms` depending on `window._tsAirbrushRate`, `2585`). Each tick:
- Stops itself if `drawing` became false or airbrush flags flipped off.
- If the pointer has actually moved since last tick, just updates its tracked position/time and returns (movement-driven dabs "own" deposition).
- Only stamps an extra dab at the last known position if the stroke has been still for **at least 2 spray-interval durations** (`2610`), avoiding double-painting the same spot as both movement dabs and timer dabs.
- Session ownership: implicit via the `drawing` flag and closure over `lx/ly` — no explicit session-id check inside the interval callback (unlike the stabilizer's `_activeStrokeSession` check). This is a narrower risk surface than the stabilizer path since `_stopAirbrushSpray()` is called from every stroke-ending function (`_pointerEndStroke` at `4410`, `_endStroke` at `3490`).

**If there is no hold behavior for a given configuration** (Stabilization "effectively 0" per §4's caveat, and airbrush off): the brush tip simply stays wherever the last conditioned sample placed it; no RAF/timer runs; rendering does not continue on its own.

---

## 9. Pointer-up and finish behavior

- **Owning function:** `_pointerEndStroke(e)` (`4406`), invoked from the `pointerup` listener (`4490-4495`), guarded by `_strokeCompletionStarted` (idempotency) and pointer-id ownership.
- **Final pending-sample flush:** `_baselineConditionerPush(..., {force:true})` (`4436`) forces emission of the true final raw sample even if it didn't cross the canonical-step threshold.
- **Stabilizer catch-up on release:** if stabilization is active, `drawing=false` is set immediately but completion is **deferred** via `_stabilizerFinalize(finalRaw.x, finalRaw.y, ..., callback)` (`4439`, definition `2468`) — the stabilizer keeps animating toward the true final point (reusing the same RAF loop as mid-stroke hold-catch-up) and only calls the passed callback (which does `_flushCurveTail`→`_flushStrokeTail`→`_commitStrokeCanvas`→`saveActiveToKey`→`_finalizePointerEndStroke`) once fully converged. If stabilization is inactive, all of that runs **synchronously** in the same `pointerup` handler (`4448-4460`).
- **Endpoint freezing / final geometry emission:** `_flushCurveTail(e)` (`3114`, seen at `4440`/`4456`) completes the SG-reconstruction window and emits the true final raw pen position exactly, so the stroke visibly ends under the pen with no perceptible lag (comment `3108-3113`).
- **Taper/release envelope:** `_getEndTaper()`/taper-mode logic (`3662-3663`), captured via `_beginEndTaperCapture()` at stroke start (`382`, called `4206`) and applied during dab param resolution — not traced line-by-line here but confirmed present and wired into `_computeEffectiveParams`.
- **Stroke commit:** `_commitStrokeCanvas()` — same function used by every completion path.
- **Undo finalization:** none additional here — undo was already pushed at stroke **start** (`pushUndo()` at `4190`), so pointer-up doesn't push a second undo entry; only `saveActiveToKey()` persists final pixels.
- **Cleanup:** `_finalizePointerEndStroke(e)` (`4463`) resets `_pendingDabs`, `_curveP0/_curveP1`, `_strokeSegCarryOver`, `_activeStrokePointerId`, `_strokeOwnerLayer/Frame`, plus telemetry/profiler teardown calls.
- **Pointer capture release:** `activeC.releasePointerCapture(e.pointerId)` (`4494`), guarded by `hasPointerCapture` check.

**Answers:**
- Which function owns finish: `_pointerEndStroke`, with `_finalizePointerEndStroke` as the common tail shared with cancellation paths.
- Can async callbacks continue after pointer-up? Yes — the stabilizer-finalize RAF chain explicitly continues running *after* `pointerup` fires, until convergence, before the commit actually happens.
- Stale callbacks prevented via: `_activeStrokeSession` id captured as `ownerSession` at `_stabilizerFinalize` call time (`2469`) and checked inside the finalize callback (`2475`) — if a new stroke has started (bumping `_activeStrokeSession`) before the old finalize fires, the old callback silently no-ops (`_traceStrokeLifecycle('stabilizer-finalize-rejected', ...)`).
- New stroke before previous finish completes: `_brushPointerDown` explicitly detects this (`4110-4117`) and calls `_stabilizerAccelerateToCompletion()` (`2481`) to synchronously fast-forward (up to 32 simulated 1/30s steps) the old stroke's convergence and fire its commit callback *before* the new stroke's own initialization proceeds — preventing the old stroke's delayed commit from landing on top of the new one's canvas state.
- `pointercancel`: routed to `_endStroke(e.pointerId)` (`4496`), which is the **cancellation** variant (§10) — it does flush and commit whatever ink already exists (`3492`) rather than discarding it, but does *not* run the stabilizer catch-up/finalize convergence (`_stabilizerActive&&!_stabilizerFinishing` → immediate `_stabilizerCancel()` at `3485`, discarding any not-yet-converged tail).

---

## 10. Tool switching and cancellation

| Trigger | Handler | Effect |
|---|---|---|
| Escape (curve tool bending) | `document.keydown` listener (`4482-4487`) | `_cancelCurveTool()` (`4328`) — resets curve gesture, recomposites, discards uncommitted preview |
| Enter (curve tool bending) | same listener | `_commitCurveTool(e)` (`4329`) — commits the curve |
| `tool-changed` custom event, new tool ≠ curve | `4489` | `_cancelCurveTool()` if a curve gesture was active |
| Right-click / contextmenu while curve gesture active | `4488` | `_cancelCurveTool()` |
| Frame/layer switch | `window.finishActiveDrawingBeforeArtworkChange` (`3515`), called from `loadFrame()` (`panels.js:315`) | If a stroke/line/eraser session is active: fast-forwards stabilizer if needed, then calls `_endStroke(_activeStrokePointerId)` targeted at the *stroke's owner* layer/frame (not necessarily the one being switched away from), then restores `curLayer/curFrame` to the destination. Guarded by `_endingForArtworkChange` flag. |
| Tab hidden / window blur | `visibilitychange`/`blur` listeners (`3542`, `3556`) | `_endStroke()` (no pointerId filter — ends unconditionally) |
| Tab shown again | `visibilitychange` else-branch (`3553`) | `loadFrame(curLayer,curFrame)` — defensively reloads from the persisted key in case the browser discarded `activeC`'s backing store while backgrounded |
| Transform tool start | `_brushPointerDown` guard (`4097`) blocks *new* strokes when `tool==='transform'`, but does not itself cancel an in-progress brush stroke — that would need to come from whatever sets `tool='transform'` calling `finishActiveDrawingBeforeArtworkChange` or an equivalent; **not found inside brush-engine.js** (external code's responsibility — flagged as an open question, §17). |
| Modal close | No direct hook found in this file. |
| New pointer interaction (second stroke starts) | `_brushPointerDown` (`4114-4117`) | `_endStroke(_activeStrokePointerId)` for the old stroke, then proceeds with new stroke init |

- **Cancellation functions:** `_endStroke(pointerId)` (`3482`) — the general-purpose "stop now" path; `_cancelCurveTool()`/`_resetCurveToolGesture()` for curve-specific state; `_cancelLinePreview()` for RAF-based line preview.
- **Pending RAF/timer cancellation:** `_endStroke` cancels stabilizer RAF (`3485`, unless finishing), `_stopAirbrushSpray()` clears the `setInterval`; `_completePostStrokePresentation`/`finishActiveDrawingBeforeArtworkChange` cancel `_recompRAFHandle`.
- **Partial stroke pixels:** `_endStroke` **does** commit whatever was drawn so far if `drawing` was true (`3492`) — i.e. mid-gesture cancellation of an actual painting stroke keeps the ink; only the **line/curve drag preview** (uncommitted scratch, never touched the layer) is discarded on cancellation (`3493-3501`).
- **Undo behavior on cancellation:** since `pushUndo()` already fired at stroke start, a cancelled-but-partially-painted stroke still has exactly one undo entry (correct, no double-push); a cancelled line/curve drag that never committed never pushed undo at all (also correct — `pushUndo()` for line/curve happens at `_pointerEndStroke`'s line-commit branch, `4419`, or `_commitCurveTool`, `4331`, not at gesture start).

---

## 11. Undo/history integration

- **When captured:** `pushUndo()` — `tools-color.js:171` — called once per brush/eraser stroke at `_brushPointerDown` (`4190`, before the first dab), and once per **committed** line/curve at their respective commit points (`4419`, `4331`). Fill tool pushes its own undo inline (`4163`) before flood-filling.
- **What is captured:** `_currentUndoSnapshot()` (`tools-color.js:157`) — a full snapshot object: `{snap, styleBundle, extendedSnapshot, frame, layer, layerType, selectionSnapshot}`.
  - **Bitmap layers:** `snap` = a full clone of `activeC` at that instant (`mkLayerCanvas()` + `drawImage`, `166-167`) — i.e. a **whole-canvas raster copy**, not a diff/patch.
  - **Smart-raster layers:** `snap:null`; instead `styleBundle = getStyleFrameBundle(curLayer,curFrame)` — style/palette metadata bundle instead of (or in addition to conceptually) raw pixels.
  - Also captures `extendedSnapshot` (out-of-bounds canvas region, if the frame has one) and a `selectionSnapshot` (pixel selection state) unconditionally.
- **Layer/frame/style metadata included:** yes — `frame`, `layer`, `layerType` (`'smart-raster'` vs `'bitmap'`) are all embedded, so `_restoreUndoAction` (`tools-color.js:203`) can route to `restoreSmartRasterUndo` vs `restoreBitmapUndo` correctly and switch layer/frame back if needed (`178-179`, `194-195`).
- **One-undo-per-stroke:** confirmed — no code path pushes a second undo entry between stroke start and stroke end; `_pointerEndStroke`/`_endStroke`/`_commitCurveTool` never call `pushUndo()` again.
- **Preview vs. committed history:** only the committed result is ever snapshotted (undo is pushed *before* any pixels change, capturing the pre-stroke state — standard "push before mutate" undo, not diff-of-final-result). Live preview (`_strokeCanvas`/`_strokePreviewCanvas`) never touches the undo stack.
- **Memory model:** `undoStack` capped at 40 entries (`tools-color.js:173`), each holding a **full-canvas bitmap clone** for bitmap layers — i.e. memory cost scales with canvas size × 40, not with stroke complexity. `redoStack` cleared on every new undo push (standard linear-history semantics).

Per the task instructions, this audit does **not** recommend replacing this undo model with the prototype's full-texture undo stack — noted here only as context: the existing model is a **pre-stroke full-canvas snapshot**, which is architecturally different from (and must not be silently swapped for) whatever undo model the prototype file uses.

---

## 12. Settings and presets

Two distinct sourcing patterns coexist:

1. **Direct DOM reads, cached per-stroke-lazily:** e.g. `_getSizeControl`, `_getFlowControl`, `_getOpacityControl`, `_getMinSize`, `_getMinFlow`, `_getTaperMode`, `_getStartTaper`, `_getEndTaper` (`3656-3663`) — each caches its `document.getElementById(...)` **element reference** in a module-level `let` (e.g. `_elSizeControl`) the first time it's called, but still reads `.value` fresh on every call, explicitly to avoid per-dab `getElementById` cost while keeping live-slider-drag responsiveness (comment `3642-3654`). Pressure-curve selectors (`_getPressureCurve`, `3664`) call `getElementById` fresh every time (not cached) — a minor inconsistency worth noting but not urgent.
2. **`window._ts*` global state, written by `brush-presets.js`'s `bindRange`/binding helpers:** e.g. `window._tsStabilization` (`brush-presets.js:184-186`), `window._tsScatterEnabled/Amount/Count/BothAxes`, `window._tsAirbrushRate`, `window._tsCustomPressureCurves`, `window._tsBrushAngle`, `window._tsSpacing`. These are read directly as globals inside brush-engine.js (e.g. `2251`, `2526-2527`, `2586`, `2657`) rather than via getter functions.
3. **Non-`ts-` prefixed direct globals:** `brushFlow`, `brushOpacity`, `brushHardness`, `color`, `window.brushBlendMode`, `window.brushTipCanvas`/`brushTipVersion`/`brushTipSizeJitter`/`brushTipAngleJitter`/`brushTipRoundnessJitter`/`brushTipRoundness`/`brushTipMinimumRoundness`, `window._brushAirbrush`, `window._brushContinuousSpraying`, `toolSizes[tool]` (via `getBrushSize()`, `32`) — presumably set by `brush-presets.js`/preset-loading code (not traced line-by-line here, out of brush-engine.js scope) but consumed as ambient globals throughout brush-engine.js.

**Source of truth:** there is **no single unified brush-settings state object** read by the engine — it's a mix of live DOM values (pattern 1), a `window._ts*` namespace (pattern 2), and bare globals (pattern 3), all read directly at dab-resolution time rather than snapshotted once per stroke. This means live mid-stroke edits to most sliders take effect immediately (which may be desired UX) but also means **there is no single settings object to hand off cleanly to a new engine** — any migration must either replicate this exact multi-source read pattern or introduce an adapter that gathers all of these into one settings object per dab/segment.

**Answers:**
- Must continue working: at minimum everything enumerated above — size, opacity/flow (+ pressure toggles per-control), hardness (implied via tip/mask generation, not traced line-by-line), antialiasing mode, airbrush + continuous-spray, pressure curve (per-setting + custom), stabilization, spacing, scatter, shape-dynamics jitter (size/angle/roundness), blend mode, min size/flow, taper mode/start/end, brush preset (tip image + associated jitter settings).
- Not wired into the old engine: cannot be conclusively determined from static reading alone — tilt/twist are **captured** (`_baselineSampleFromEvent`) but no downstream consumer of `tiltX/tiltY/twist` was found in brush-engine.js, suggesting tilt-based dynamics (if any exist in the prototype) are **not currently wired** in the real engine. Flagged as "needs manual testing" (§15) rather than asserted as fact.
- Live-DOM vs. state-object reads: see the three patterns above — worth deciding, as part of migration seam design, whether the new engine should read the same live sources or be handed a resolved settings object per stroke.

---

## 13. Architecture diagram

```
pointerdown (activeC / canvasArea)
  │
  ▼
_brushPointerDown
  ├─ guard checks (tool, group, pan/zoom/transform, hidden/locked layer)
  ├─ finish/end any still-active previous stroke (session-safe)
  ├─ currentPressure = _getPressure(e)
  ├─ p = getPos(e)                          ── coordinate conversion
  ├─ _resetStabilization(p.x,p.y,t)         ── stabilization reset
  ├─ _updateVelocity(...)
  ├─ [fill tool: pushUndo → ensureKey → floodFill → saveActiveToKey → recomposite → RETURN]
  ├─ activeC.setPointerCapture(pointerId)
  ├─ pushUndo()                              ── undo capture (once)
  ├─ ensureKey()                             ── layer/frame routing
  ├─ _beginColorEraserStroke / _beginEndTaperCapture / selection-scope capture
  ├─ drawing=true; _resetCurve(...)          ── curve/spacing init
  ├─ _ensureStrokeCanvas(); _inStroke=true   ── render destination init
  ├─ _stampDab(p.x,p.y,e)                    ── brush geometry → first dab
  │     └─ _getEffectiveBrushParams → _queueDab → _drawDabNow → _strokeCanvas
  └─ _scheduleRecomposite({firstDab:true})   ── renderer/compositor
        └─ recomposite(curLayer,curFrame,rect) → visible canvas

pointermove / pointerrawupdate
  │
  ▼
_handleMoveEvent
  ├─ coalesced/raw sample expansion (getCoalescedEvents)
  ├─ per sample: getPos → coordinate conversion
  ├─ _getPressure → pressure pipeline (curve/smoothing applied later, per-dab)
  ├─ _baselineConditionerPush → canonical resampling (always-on)
  ├─ _stabilizePoint → user stabilization (slider-driven)
  ├─ _updateVelocity
  ├─ _curveAddPoint → curve/path reconstruction (midpoint quad, +SG at low zoom)
  │     └─ _stampQuadCurve → _walkDabArc → _queueDab (brush geometry/dabs)
  │           └─ _drawDabNow → _strokeCanvas (render destination, offscreen)
  └─ _scheduleRecomposite → recomposite() → visible canvas (live preview blend)
```

**Hold path (separate):**
```
pen stationary, still down
  │
  ├─ Stabilizer catch-up: _stabilizerStep (RAF loop, 18ms idle delay)
  │     → _stabilizerAdvance → _stabilizerEmit → _curveAddPoint → dabs → _scheduleRecomposite
  │     (continues until converged within epsilon; pressure stays pinned)
  │
  └─ Airbrush spray (if enabled): setInterval timer
        → if truly still for ≥2 intervals → _stampDab at last position → _scheduleRecomposite
```

**Finish path (separate):**
```
pointerup
  │
  ▼
_pointerEndStroke
  ├─ force-flush final conditioned sample
  ├─ IF stabilizer active: _stabilizerFinalize(...) → defers to async RAF convergence
  │     └─ on converge: _flushCurveTail → _flushStrokeTail → _commitStrokeCanvas
  │           → _restoreSelectionScopePixels → _cleanupErasedSmartOwnership
  │           → saveActiveToKey → _finalizePointerEndStroke
  ├─ ELSE (no active stabilizer): same sequence runs synchronously inline
  └─ _finalizePointerEndStroke: telemetry teardown, state reset,
       releasePointerCapture (in the pointerup listener wrapper)
```

---

## 14. Ownership map

| Responsibility | File | Function(s) | State owned |
|---|---|---|---|
| Pointer input | `brush-engine.js` | `_brushPointerDown`, `_handleMoveEvent`, `_pointerEndStroke`, pointerrawupdate/pointermove listeners | `_activeStrokePointerId`, `drawing`, `_inStroke`, `_strokeCompletionStarted` |
| Coordinate conversion | `brush-engine.js` | `getPos` | none (pure function of pan/zoom/rotation/flip globals) |
| Sample conditioning | `brush-engine.js` | `_baselineConditionerPush`, `_baselineSampleFromEvent`, `_baselineEmit` | `_baselineConditionerState` |
| Stabilization | `brush-engine.js` | `_stabilizePoint`, `_stabilizerAdvance`, `_stabilizerStep`, `_stabilizerFinalize`, `_resetStabilization` | `_stabilizerX/Y`, `_stabilizerTargetX/Y`, `_stabilizerSpeed`, `_stabilizerRAF`, `_stabilizerFinishing`, `_stabilizerFinalizeCB` |
| Pressure | `brush-engine.js` | `_getPressure`, `_applyPressureCurve`, `_resolveControl` | `currentPressure`, `_smoothedPressure`, `_lastKnownPressure`, `_prevRawPressure` |
| Curve generation | `brush-engine.js` | `_curveAddPoint`, `_curveAddReconstructedPoint`, `_stampQuadCurve` | `_curveP0/P1`, `_curvePr0/Pr1`, `_curveBaselineSamples` |
| Spacing/dabs | `brush-engine.js` | `_effectiveSpacingFrac`, `_walkDabArc`, `_queueDab`, `_stampDab` | `_strokeSegCarryOver`, `_strokeDistSoFar`, `_strokeDabCount`, `_pendingDabs` |
| Rendering | `brush-engine.js` (+ `panels.js` compositor) | `_drawDabNow`, `_ensureStrokeCanvas`, `_commitStrokeCanvas`, `_getLiveStrokePreview`, `recomposite` | `_strokeCanvas`, `_strokePreviewCanvas`, `_frameDirty`/`_strokeDirty` |
| Layer/frame routing | `core-state.js` / `panels.js` | `curLayer`/`curFrame` (globals), `ensureKey`, `saveActiveToKey`, `loadFrame` | `layers[i].frames[j]`, `activeC` contents |
| Hold | `brush-engine.js` | `_stabilizerStep`/`_stabilizerAdvance` (catch-up), `_startAirbrushSpray`/interval callback | `_stabilizerRAF`, `_airbrushTimer` |
| Finish | `brush-engine.js` | `_pointerEndStroke`, `_finalizePointerEndStroke`, `_flushCurveTail`, `_flushStrokeTail` | `_strokeCompletionStarted`, `_activeStrokeSession` |
| Undo | `tools-color.js` | `pushUndo`, `_currentUndoSnapshot`, `_restoreUndoAction` | `undoStack`, `redoStack` |
| Cancellation | `brush-engine.js` | `_endStroke`, `_cancelCurveTool`, `finishActiveDrawingBeforeArtworkChange` | same stroke-session state as "Finish" row |

---

## 15. Migration classification

| Subsystem | Classification | Reasoning |
|---|---|---|
| `getPos` (coordinate conversion) | **Preserve** | Encodes exact inverse of the app's pan/zoom/rotation/flip transform chain; a prototype using a different (e.g. simpler) canvas setup would need this exact math, not a generic replacement. |
| `_getPressure` (raw pressure read + jump-clamp) | **Preserve**, with test | Tablet-driver-quirk handling (0-while-drawing, jump clamp) is empirical tuning; replacing risks reintroducing known bugs the comments describe as already fixed. |
| Sample conditioner (`_baselineConditionerPush`) | **Adapt/bridge** | This "always-on" resampling stage is conceptually orthogonal to whatever the prototype's stabilization does. If the prototype has its own resampling, the two must not stack; if not, this stage likely still needs to feed the prototype's stabilizer at the seam (§16). |
| User stabilizer (`_stabilizePoint`/RAF catch-up) | **Replace with prototype behavior** (that's the explicit goal), but only after the 0%-bypass discrepancy in §4 is resolved/understood | This is the subsystem the whole migration exists to swap. Must decide whether the prototype reproduces the spatial-leash-plus-EMA model, a different lazy-brush model, or something else, and whether the always-on conditioner (previous row) stays. |
| Pressure curve/smoothing (`_applyPressureCurve`, `_resolveControl`) | **Preserve** | User-facing, matched 1:1 with a UI curve-preview widget in `brush-presets.js`; a mismatch would be immediately visible to users. |
| Curve/geometry (`_stampQuadCurve`, midpoint reconstruction, arc-walk spacing) | **Needs manual testing before decision** | Whether the prototype's stabilization output is even compatible with a midpoint-quadratic consumer (vs. expecting raw polyline points) determines whether this stays untouched or needs adapting. |
| Dab rasterization (`_dabAA*`, tip stamping, texture, scatter, jitter) | **Preserve** | High-value, complex, unrelated to stabilization; touching this is out of scope and high-risk for a stabilization-focused migration. |
| Stroke canvas / commit (`_strokeCanvas`, `_commitStrokeCanvas`, opacity/blend compositing) | **Preserve** | Central to correct stroke-level Opacity/Flow and blend-mode behavior; any new stabilizer must still feed dabs into this exact pipeline. |
| Layer/frame/cel routing (`ensureKey`, `saveActiveToKey`, `curLayer`/`curFrame`) | **Preserve** | Explicitly called out in the task as must-preserve; nothing about stabilization needs to touch this. |
| Undo/history (`pushUndo`, snapshot model) | **Preserve — do not replace with prototype's undo model** | Explicit instruction; also architecturally full-canvas-snapshot vs. whatever the prototype does, incompatible without a real design exercise. |
| Smart-raster ownership commit (`commitSmartRasterBrush`) | **Preserve** | Palette/style metadata correctness is layer-type-specific and unrelated to stabilization; must keep firing exactly where it does today (post-commit, pre-`saveActiveToKey`). |
| Airbrush continuous spray | **Adapt/bridge** | Currently keyed off `lx/ly` (raw last-stabilized position) and `drawing` flag; if the seam changes how "current position" is tracked, this timer's read of `lx/ly` needs to keep working or be rewired to the new position source. |
| Tool switching / frame-change cancellation (`finishActiveDrawingBeforeArtworkChange`, `_endStroke`) | **Preserve** | Session-id staleness protection (`_activeStrokeSession`) is exactly the kind of subtle correctness mechanism a migration could accidentally bypass; the new stabilizer's own async completion (if any) must integrate with this same session-check pattern (§16, §17). |
| Settings sourcing (`window._ts*` / DOM reads) | **Adapt/bridge** | Not itself something to replace, but the seam design must decide whether the new stabilizer reads `window._tsStabilization` directly (matching existing pattern) or receives a resolved value — needs an explicit decision, not silent copying. |

---

## 16. First safe migration seam

**Determined seam (not the naive guess in the prompt — the actual seam the code supports):**

```
real project: conditioned sample stream
   { x, y, pressure, time, event }  — output of _baselineConditionerPush /
                                       _baselineSampleFromEvent, i.e. what
                                       currently feeds _stabilizerSetSampleContext
                                       + _stabilizePoint
        │
        ▼
   [NEW stabilization/pressure processing]  ← prototype logic goes here
        │
        ▼
   filtered { x, y } (+ pressure passthrough, per existing
   "stabilization owns position only" contract at _stabilizerEmit)
        │
        ▼
real project: _updateVelocity → _curveAddPoint → _stampQuadCurve → dabs
```

This differs from the prompt's suggested seam ("raw canvas-space position") in one important way: the **always-on sample conditioner** (`_baselineConditionerPush`) sits *between* raw `getPos()` output and the current stabilizer, and is not itself a "stabilization" concern — it's a canonical-resampling/anti-duplicate stage that likely still needs to run regardless of which stabilizer consumes its output, since downstream curve reconstruction (`_curveAddPoint`'s corner-preserving SG smoothing) implicitly assumes a reasonably dense, deduplicated sample stream. Inserting a new stabilizer *before* the conditioner (i.e. truly at raw `getPos()` output) risks feeding it duplicate/zero-distance samples the conditioner exists to filter out.

- **Exact input contract:** a conditioned sample `{x, y, screenX, screenY, pressure, tiltX, tiltY, twist, time, pointerId, event}` (shape defined by `_baselineSampleFromEvent`, `2349`), delivered once per emitted canonical sample (0..N per raw pointer event, per `_baselineConditionerPush`'s emission rules).
- **Exact output contract:** `{x, y}` canvas-space float — must preserve `_stabilizerEmit`'s existing convention that **pressure is not filtered by this stage** (`2419-2420`); if the prototype's stabilizer also smooths pressure, that's a deliberate behavior change requiring explicit sign-off, not an incidental one.
- **Files/functions involved:** `brush-engine.js` — replace/wrap `_stabilizePoint` + the `_stabilizerAdvance`/`_stabilizerStep`/`_stabilizerFinalize`/`_stabilizerSchedule` RAF-catch-up machinery. Everything upstream (`getPos`, `_getPressure`, `_baselineConditionerPush`) and everything downstream (`_updateVelocity`, `_curveAddPoint` onward) stays untouched.
- **State that must be initialized/reset:** equivalent of `_resetStabilization(x,y,t)` must still be called from `_brushPointerDown` at stroke start, and the new stabilizer must expose an equivalent of `_stabilizerFinalize(...)`/`_stabilizerAccelerateToCompletion()` so `_pointerEndStroke` and `finishActiveDrawingBeforeArtworkChange` can still force-converge-and-commit synchronously when needed (§9, §10) — **this async-finalize contract is the single most load-bearing piece of the seam** and is easy to underestimate.
- **Systems that remain untouched:** sample conditioning, curve reconstruction/geometry, dab rasterization, stroke canvas/commit, layer/frame routing, undo, settings sourcing, tool-switching cancellation logic (aside from whatever new finalize hook it needs to call).
- **Risks:** see §17.

---

## 17. Compatibility risks

All of the following are grounded in the code read above, not speculative:

- **Duplicate stabilization / conditioning:** if the prototype's stabilizer includes its own resampling or duplicate-rejection, and it's inserted *in addition to* (rather than *instead of*) `_baselineConditionerPush`, samples could be double-filtered, changing feel unpredictably. (§16)
- **Async finalize contract gap:** the real engine relies on `_stabilizerFinalize`/`_stabilizerAccelerateToCompletion` being callable synchronously-to-completion from `_pointerEndStroke`, `_endStroke` (conditionally), and `finishActiveDrawingBeforeArtworkChange`. If the new stabilizer doesn't expose an equivalent "finish now, synchronously if needed" API, tool-switch/frame-switch/tab-blur cancellation could leave a stroke's tail un-flushed, silently dropping the last few pixels of a stroke or leaving stray async work targeting a now-wrong layer/frame. (§9, §10, §16)
- **Stale-session callbacks:** the existing `_activeStrokeSession` check (`2469`, `2475`) is what prevents a slow-converging old stroke's finalize callback from committing into whatever layer/frame is now active. A replacement stabilizer's async completion must be wired into this same (or an equivalent) session-guard, or a race between "new stroke starts" and "old stroke's stabilizer finally converges" could paint into the wrong layer/frame — a real, demonstrated failure mode this code explicitly guards against today. (§9)
- **Pressure/position cadence mismatch:** the current contract is "stabilization owns position only, pressure passes through raw" (`2419-2420`). A prototype stabilizer that also smooths pressure would be a silent behavior change unless explicitly decided. (§5, §16)
- **0%-stabilization semantics:** `_stabilizationAmount()`'s `amount<=0?1:amount` coercion (`2252`) means the documented "true bypass" branch in `_stabilizePoint` (`2369`) may be unreachable via the UI slider today. Migrating without resolving this ambiguity risks either (a) preserving a possible existing bug, or (b) "fixing" it and unintentionally changing behavior at the 0% setting that users currently rely on. Needs a decision, not a guess. (§4)
- **Airbrush timer reading stale position semantics:** `_startAirbrushSpray`'s interval callback reads module-level `lx/ly` directly (`2596-2613`), not through any stabilizer accessor. If the seam changes how "current stabilized position" is exposed/named, this timer must be updated in lockstep or airbrush-while-held will silently stop tracking the real brush position.
- **Two RAF loops owning hold/finish:** currently only the stabilizer has a hold/finish RAF loop (`_stabilizerStep`); `_scheduleRecomposite` has its own separate RAF coalescing. If the prototype's stabilizer brings its own independent RAF loop that isn't unified with `_stabilizerSchedule`'s idle-delay/convergence logic, there's a real risk of **two competing RAF loops** both trying to own "does the brush keep moving while held," producing double-emission or fighting update rates — exactly the kind of bug the file's own comments describe having fixed once already (pointermove vs. pointerrawupdate double-stamping, §3).
- **Coordinate-unit mismatch:** all downstream geometry (`_curveAddPoint`, `_stampQuadCurve`, dab radius math) operates in canvas-space float pixels **post-`/zoom`**. If the prototype's stabilization file operates in raw screen pixels (common for standalone demos), inserting it at this seam without converting units would silently corrupt spacing/curve shape at any zoom level other than 100%. Must confirm the prototype's coordinate convention before wiring in.
- **Low-zoom SG reconstruction feeding on stabilizer output:** `_curveAddPoint`'s Savitzky–Golay smoothing (`_curveSubpixelConditioning`, active when `zoom<1`) assumes a dense, evenly-conditioned sample stream from upstream. A new stabilizer that changes sample density/timing characteristics (e.g. emits fewer or burstier points) could interact with this smoothing in ways not exercised by the current stabilizer — flagged as needing manual low-zoom testing, not assumed safe.
- **Tool switch mid-stroke via `tool='transform'` etc.:** no direct call from a generic "tool changed" path into `finishActiveDrawingBeforeArtworkChange`/`_endStroke` was found in brush-engine.js for the general tool-switch case (only the curve-tool-specific `tool-changed` listener and the transform-tool pointerdown guard exist). Whether some other file calls the cancellation hooks on tool switch is unconfirmed from this file alone — flagged as an open question (§18) rather than a known risk, since asserting it would go beyond what this file's code supports.

---

## 18. Final deliverable

### 1. Exact current real-project brush call path
See §2 (movement) and §13 (full diagram) above — summarized: `pointerdown → _brushPointerDown → getPos/_getPressure → _resetStabilization → _stampDab(first dab) → _scheduleRecomposite`, then per-move `_handleMoveEvent → getPos/_getPressure → _baselineConditionerPush → _stabilizePoint → _updateVelocity → _curveAddPoint → _stampQuadCurve → _queueDab → _drawDabNow(_strokeCanvas) → _scheduleRecomposite(recomposite)`, then `pointerup → _pointerEndStroke → [stabilizer finalize, sync or async] → _flushCurveTail/_flushStrokeTail → _commitStrokeCanvas(activeC) → saveActiveToKey(layers[...].frames[...]) → _finalizePointerEndStroke`.

### 2. Ownership table
See §14.

### 3. Preserve/replace/adapt classification
See §15.

### 4. Recommended first migration seam
Conditioned-sample-stream → new stabilizer → filtered `{x,y}` (pressure passthrough) → existing `_updateVelocity`/`_curveAddPoint` onward. See §16 for exact contracts.

### 5. Ordered migration plan with small, reversible commits
1. **Instrumentation-only commit:** add a feature flag (e.g. `window._newStabilizerEnabled`) that, when false (default), changes nothing — establishes the toggle point without touching behavior.
2. **Introduce the new stabilizer as a pure function alongside the old one**, not replacing it — e.g. `_newStabilizePoint(x,y,t)` implemented from the prototype, callable but unused, so it can be unit-tested in isolation (feed it known input sequences, inspect output) before it touches the live pipeline.
3. **Wire the flag into `_stabilizePoint`'s call site only** (`4376`/`4451` and equivalents), branching to the new implementation when enabled, while leaving `_stabilizerAdvance`/`_stabilizerStep`'s hold/finalize RAF loop as a stub that must still be implemented before this commit is usable — i.e. this commit alone will break hold/finish behavior and should be tested only with Stabilization-driven hold disabled/mocked, or should ship together with commit 4.
4. **Implement the new finalize/accelerate-to-completion contract** (equivalent of `_stabilizerFinalize`/`_stabilizerAccelerateToCompletion`/`_stabilizerCancel`) and wire it into the exact same call sites (`_pointerEndStroke`, `_endStroke`, `finishActiveDrawingBeforeArtworkChange`) behind the same flag, preserving the `_activeStrokeSession` staleness check.
5. **Manual test pass** (see item 6) with the flag on, across a real device/pen, at multiple zoom levels and Stabilization slider values, including 0% (resolving the §4/§17 ambiguity explicitly as part of this step, not deferring it).
6. **Resolve the 0%-bypass semantics decision** (§4/§17) as its own reviewed commit, independent of the stabilizer swap, since it's a pre-existing question not introduced by migration.
7. **Flip the default flag on**, keep the old path available behind the flag (flipped off) for at least one release cycle as a rollback lever.
8. **Remove old stabilizer code** only after the flag has been on-by-default with no regressions reported for a full cycle.

### 6. Manual tests required after each commit
- Draw at Stabilization 0%, 50%, 100%, at zoom 25%, 100%, 400%.
- Hold pen stationary mid-stroke; confirm catch-up animates to the pen tip and pressure doesn't drift.
- Lift pen immediately after a fast flick; confirm no visible truncation or overshoot at the endpoint.
- Start a new stroke immediately after releasing the previous one (fast double-tap); confirm no stray marks from a stale finalize callback.
- Switch layer/frame/tool mid-stroke; confirm the in-progress stroke commits correctly to its **origin** layer/frame, not the destination.
- Background the tab mid-stroke (alt-tab); confirm the stroke commits and reappears correctly on return.
- Test airbrush-continuous-spray with the new stabilizer enabled; confirm spray still tracks real position and doesn't double-stamp.
- Test mouse (no pressure) and pen (with pressure) on the same settings; confirm stroke-start taper still functions for mouse.
- Test smart-raster layer strokes; confirm palette/style ownership metadata still commits correctly.
- Confirm undo captures exactly one entry per stroke, with correct pre-stroke pixel state, across all of the above scenarios.

### 7. Open questions the code alone cannot answer
- Is the `_stabilizationAmount()` `0→1` coercion (§4) an intentional design choice (e.g. "0% still means light default stabilization") or a genuine bug where the UI's "0%" doesn't produce the documented true-bypass code path? This needs a decision from whoever owns brush feel, not a code-only answer.
- Does any code **outside** `brush-engine.js` call `finishActiveDrawingBeforeArtworkChange`/`_endStroke` on generic tool switching (e.g. switching to eyedropper, selection tools, etc. mid-stroke)? Only the curve-tool-specific and transform-tool cases were found inside this file; confirming full tool-switch cancellation coverage requires searching the tool-switching code itself (`setTool` and callers), which is outside this file and outside this audit's traced scope.
- What exactly does the prototype's stabilization file (`stabilizationtest.html`, explicitly not read for this audit per the task's "do not copy from it yet" instruction) expect as input/output units and cadence? The seam contract in §16 is derived entirely from the real project's side; confirming compatibility requires a matching read of the prototype file as a *separate*, explicitly-scoped follow-up task.
- Does `tiltX/tiltY/twist` data (captured but apparently unconsumed downstream in this file) feed anything in a different file (e.g. brush-presets.js tip-rotation-by-tilt features)? Not traced — flagged rather than assumed absent.
