# MIGRATION_PLAN.md
## Integrating the Prototype (`stabilizationtest.html`) into the Real Project (`behind-me-is-winner`)

**Status:** Planning document only. No source code was modified or written to produce this plan. No implementation code is included below.

**Sources:**
- `ARCHITECTURE.md` — audit of the prototype (`stabilizationtest.html`, commit `0d0327d`, 2026-08-04)
- `BRUSH_PIPELINE_AUDIT.md` — audit of the real project's brush **stroke/stabilization** engine (`src/brush/brush-engine.js` + supporting files). This document does **not** cover the real project's cursor/preview overlay — it is exclusively about the stroke pipeline.
- Direct read of the real project's cursor/preview code (`src/brush/brush-size-cursor.js`, `src/brush/brush-size-drag.js`, `src/ui/cursor-prefs.js`), performed specifically to scope Deliverable B below, since no existing audit document covers it.

**Revised scope statement:** This migration has **two independent deliverables**, sourced from two functionally unrelated parts of the prototype:

1. **Deliverable A — Position stabilization.** Prototype-derived stroke-position smoothing behavior (moving-average FIFO → real-project EMA/leash model). This is the subject of the original audits and the plan below in §A.1–§A.10.
2. **Deliverable B — Animated brush-dab preview.** Prototype-derived live brush-tip preview behavior (the `#sizePreview` canvas driven by `showSizePreview()`/`updateBrushPreview()`), ported against the real project's existing `#brush-cursor-overlay` (`brush-size-cursor.js`). This is a **separate, later addition** to this plan, covered in §B.1–§B.10, added because the prototype also contains a distinct animated-dab-preview mechanism with no prior audit coverage.

**An earlier version of this document stated that stabilization was the only thing being migrated. That statement is corrected here: the animated brush-dab preview is a second, independent deliverable, added below.** The two deliverables are unrelated in code (different files/functions in both the prototype and the real project — see §B.1), must ship in **separate commits/phases**, and must remain **independently switchable** via separate feature flags. Together they still leave the following real-project subsystems untouched: the brush rasterizer, spacing/dab placement, brush tips and dynamics, pressure curves, rendering backend, undo/history, layers/frames/cels, and presets/Tool Settings (beyond reading existing values).

WebGPU is not part of either deliverable. Nothing in the prototype's animated-dab-preview code (§B.1) uses or depends on the WebGPU renderer — it is drawn with plain Canvas2D `arc()`/`fill()`/`stroke()` calls onto an independent `<canvas>` element, entirely separate from the prototype's WebGPU stroke pipeline.

---

# Deliverable A — Position Stabilization

---

## A.1–A.2. Subsystem Comparison Table

| Subsystem | Prototype | Real Project | Migration Decision | Reason |
|---|---|---|---|---|
| Pointer input entry | `pointerdown`/`pointermove`/`pointerenter`/`pointerleave` on `stageWrap`; `pointerup`/`pointercancel` on `window` | `pointerdown` on `activeC` + forwarding listener on `canvasArea`; `pointerup`/`pointercancel` handled with pointer-id/session guards | **Different** | Real project's listener topology and guard conditions (layer group, pan, zoom-drag, transform tool, locked/hidden layer) have no prototype equivalent and are irrelevant to it; not part of the seam. |
| `pointerrawupdate` | Not used anywhere in the prototype | Used exclusively for pen input when supported (`_hasRawUpdate`), with explicit double-stamp avoidance vs. `pointermove` | **Incompatible / Real project is ahead** | The real project already solved a problem (double-stamping) the prototype never encountered because it never used this event. Do not import prototype input handling here. |
| `getCoalescedEvents()` | Used in `processStrokeBatch` and `moveStroke`, both with the same repeated inline pattern | Used once, centrally, inside `_handleMoveEvent` | **Similar (concept), different (implementation)** | Same purpose (recover sub-frame samples). Real project's centralization is already superior to the prototype's duplicated pattern (which its own audit flags as a cleanup item). Nothing to migrate. |
| Coordinate conversion | `screenToCanvas()` — client → backing-store space, includes pan, zoom, and a fixed `SS=4` supersample factor | `getPos()` — client → canvas-space float pixels, inverts flip/pivot, rotation, pan, zoom | **Different, both load-bearing, not interchangeable** | Real project has no supersampled backing store and has flip/rotation the prototype does not. A new stabilizer must consume/produce **real-project canvas-space float pixels** (post-`getPos`), not prototype backing-store pixels. |
| Stabilization (core algorithm) | Fixed-length moving-average FIFO (arithmetic mean) over `smoothBuf`; window size 1–100 samples, count-based | Two-stage: (A) always-on sample conditioner (canonical resampling, not user-controlled) + (B) first-order exponential low-pass filter with a spatial "leash" clamp, time-based (`dt`), velocity-dependent tau | **Incompatible algorithms — this is the crux of the migration** | Different smoothing families entirely (count-windowed mean vs. time-based EMA+leash). This is explicitly what the migration exists to change, but the two cannot simply be swapped 1:1 without redesigning how "amount," "window," and "leash" map to each other. |
| Pressure pipeline | `getPressure()` → `pressureInfluence(p^1.2)` → `pressureRadius()`; `pressureAlpha()` deliberately ignores pressure and returns global opacity | `_getPressure()` with driver-glitch handling + jump-rate clamp (`_MAX_PRESSURE_JUMP=0.09`); per-setting pressure curves (linear/soft/hard/s/custom Bézier) feeding both size and (optionally) opacity/flow | **Different, real project is materially more capable** | Real project's pressure pipeline is a full user-configurable curve system tied to a live UI preview widget; prototype's is a single hardcoded easing exponent with opacity explicitly excluded. Prototype's pressure math is not a superset or subset — do not migrate. |
| Pressure buffering | Separate `pressureBuf` moving-average FIFO, same window length as position, kept in lockstep with position via advance-count diagnostics | Stabilization explicitly does **not** filter pressure — `_stabilizerEmit` passes raw/conditioned pressure through unfiltered; separate exponential smoothing (`_smoothedPressure`) exists but is architecturally distinct from position stabilization | **Incompatible contract** | Real project's documented contract is "stabilization owns position only." Migrating the prototype's pressure-buffering behavior wholesale would silently change this contract and is flagged as a compatibility risk in both audits' spirit — requires an explicit decision, not a silent import. |
| Release pressure | Two frozen/constant-value strategies (Stationary Hold vs. Moving Release) fed into the *same* `pushPressureBuf()` FIFO used live | Real project uses taper machinery (`_getEndTaper`/`_beginEndTaperCapture`) plus `_flushCurveTail` to emit the true final raw position/pressure — a different mechanism entirely, not a moving-average tail | **Different mechanisms, do not conflate** | These solve a similar user-facing problem (natural stroke ends) via unrelated implementations. Real project's taper system is untouched by this migration; only the *position* catch-up concept (see Hold/Finish below) is relevant to the seam. |
| `feedPoint()` equivalent | `feedPoint(all, raw, pressure)` — single choke point turning one stabilized sample into geometry | `_curveAddPoint()` → `_curveAddReconstructedPoint()` — functionally equivalent choke point, plus an optional low-zoom Savitzky–Golay reconstruction stage the prototype has no equivalent of | **Similar concept, real project is a superset** | Both are the single entry point from "stabilized sample" to "curve geometry." Real project's version is strictly more sophisticated. Not replaced; it is the **downstream consumer** the new stabilizer must feed into unchanged. |
| Curve generation | Midpoint-quadratic Bézier, tessellated into a fixed `steps=10` per call | Midpoint-quadratic Bézier (same family/technique), via a 3-point rolling buffer and arc-length walk, not a fixed step count | **Similar technique, different implementation** | Same mathematical family (this is reassuring for the seam — the new stabilizer's output shape should be compatible with the consumer), but real project's version is distance-driven, not sample-count-driven. Preserve real project's version; do not import the fixed-`steps=10` scheme. |
| Spacing | No explicit fixed-distance spacing model; tessellation density is driven by `steps=10` and long-jump subdivision only | Explicit spacing model — `_effectiveSpacingFrac`/`_effectiveDabStep`, arc-length dab walk at ~12% of effective diameter | **Missing in prototype** | Prototype has no equivalent concept; nothing to migrate. Real project's spacing system is preserved untouched and is a downstream consumer of stabilizer output only. |
| Geometry generation | `circleVerts()`/`segmentVerts()` — bounding-box quads with per-vertex shape params for a GPU fragment shader (analytic SDF) | `_walkDabArc` → `_queueDab` → `_drawDabNow`/AA rasterizers, tip stamping with scatter/jitter/texture, all Canvas2D | **Incompatible — different rendering targets entirely** | Prototype geometry is GPU-vertex-shader-oriented; real project geometry is CPU/Canvas2D dab-rasterization-oriented. This subsystem is not part of the migration in either direction. |
| Rendering | WebGPU (3 pipelines: stroke-mask → composite → blit), 4× supersampled backing store, WASM module (confirmed unused) | Canvas2D only — `getContext('2d', ...)` everywhere; no WebGL/WebGPU path exists | **Incompatible / out of scope** | The real project has no WebGPU renderer to receive prototype rendering code, and adopting WebGPU rendering is a project-scale decision far outside a stabilization migration. Prototype rendering subsystem is **not migrated**. |
| WebGPU | Primary rendering path in the prototype | **Does not exist** in the real project | **Missing in real project — out of scope for this migration** | Confirmed by direct inspection of both documents; introducing WebGPU rendering is a separate initiative, not a consequence of migrating stabilization. |
| Canvas2D fallback | A deliberately minimal degraded-mode path (no stabilization, no pressure taper, `getImageData`/`putImageData` undo) used only when WebGPU is unavailable | Canvas2D is the **only** rendering path and is fully featured (spacing, scatter, jitter, tip textures, AA, blend modes, smart-raster) | **N/A — not comparable as "fallback," real project's Canvas2D is the primary and only path** | The prototype's Canvas2D code is explicitly the *weaker* of its two renderers and is not a useful reference for the real project's Canvas2D pipeline, which is already comprehensive. Nothing to migrate from prototype's fallback. |
| Hold behavior | `strokeHoldTick` — persistent per-frame RAF loop from `beginStroke` to `endStroke` entry; wall-clock-driven catch-up (`ticksPerMs`) toward a 350ms convergence target | `_stabilizerStep` — RAF loop armed by `_stabilizerSchedule()`, 18ms idle delay before catch-up begins, converges within an epsilon (`0.30px`); **separate** airbrush spray mechanism (`setInterval`, not RAF) | **Similar concept, different implementation, real project has an additional concern (airbrush) prototype lacks** | Both implement "brush tip visibly catches up while pen is held still" via RAF loops with session guards — conceptually compatible. The new stabilizer must expose an equivalent catch-up hook that plugs into (or replaces) `_stabilizerStep`, and must not introduce a *second* independent RAF loop alongside the existing one (flagged as a real risk in the real-project audit). |
| Finish behavior | `endStroke`/`finishTick` — two finish modes (Stationary Hold / Moving Release), accelerating catch-up rate, session-guarded via `activeStrokeSessionId` | `_pointerEndStroke` → `_stabilizerFinalize` (async RAF convergence) or synchronous inline path if stabilization inactive; `_stabilizerAccelerateToCompletion` fast-forwards a still-converging old stroke when a new one begins | **Similar concept, real project's contract is more precisely specified** | Both defer final commit until the stabilized output converges to the true endpoint, and both guard against a stale finish corrupting new state. The real project's `_stabilizerFinalize`/`_stabilizerAccelerateToCompletion` contract is the one the new stabilizer must implement equivalents of — not the prototype's `finishTick`. |
| Session ownership | `activeStrokeSessionId`/`nextStrokeSessionId`, monotonic tokens, checked inside `strokeHoldTick` and `finishTick` before acting; documented "REQUIRED OWNERSHIP RULE" | `_activeStrokeSession`, monotonic, captured as `ownerSession` at finalize-schedule time and checked inside the finalize callback; also used by `_brushPointerDown` to detect and fast-forward a still-live prior stroke | **Similar / compatible pattern** | Both projects independently arrived at the same guard pattern for the same reason (preventing stray async callbacks from corrupting a newer stroke's state). This is good news for the migration: the new stabilizer's async completion must be wired into the **real project's** `_activeStrokeSession` mechanism, not a newly introduced one. |
| Undo | GPU texture stack (WebGPU) or `ImageData` snapshots (Canvas2D fallback), `MAX_UNDO=12`, full-resolution copies, no delta/compression | Pre-stroke full-canvas bitmap clone (bitmap layers) or style/palette bundle (smart-raster layers), captured once via `pushUndo()` at stroke *start*, `MAX_UNDO`-equivalent of 40 entries, plus selection/extended-frame snapshot data | **Incompatible — explicitly not to be migrated** | Both source documents are explicit on this point from their own sides: the real-project audit states the prototype's full-texture undo model must not silently replace the real project's pre-stroke snapshot model. This is a hard boundary for the migration. |
| Layer/frame routing | **None** — the prototype is a single flat canvas with no layer or frame concept | `curLayer`/`curFrame` globals, `ensureKey`, `saveActiveToKey` persisting into `layers[i].frames[j]`, extended (larger-than-canvas) frame support, smart-raster ownership commit | **Missing in prototype** | Nothing to compare or migrate; this entire subsystem belongs to the real project alone and must remain untouched by a stabilization-focused change. |
| Brush settings | A handful of globals: `tool`, `color`, single `size`/`opacity` value, `stabilization`, `leashEnabled`, `zoom`/`pan` | Three coexisting sourcing patterns: cached DOM reads, `window._ts*` globals (bound via `brush-presets.js`), and bare globals (`brushFlow`, `brushOpacity`, tip-jitter fields, etc.) — no single unified settings object | **Missing/simpler in prototype** | Prototype has no equivalent of the real project's multi-source settings system. Not migrated; the new stabilizer must be designed to read whichever of these sources holds the "Stabilization amount" (currently `window._tsStabilization`), matching the existing sourcing pattern rather than inventing a new one. |
| Brush presets | **None** — no preset system exists in the prototype | Full preset system (`brush-presets.js`): tip image, jitter (size/angle/roundness), scatter, spacing, pressure curves, taper mode, airbrush rate, blend mode | **Missing in prototype** | Not part of the migration in any direction; flagged here only to confirm the prototype has nothing that could conflict with or need reconciling against the real project's preset system. |
| Zoom compensation | Two mechanisms: `zoomSmoothingFactor()` scaling a jitter floor, and a separate hidden stabilization-amount floor blended via `computeInternalStabilization()`, both operating on a *count-based* moving-average window | Leash bound (`_stabilizerMaxLagCanvas`) converts a fixed screen-pixel bound to canvas space via `/zoom`; separately, low-zoom (`zoom<1`) Savitzky–Golay reconstruction inside `_curveAddPoint` assumes a dense, evenly-conditioned sample stream | **Different mechanisms solving a related but not identical problem** | Prototype's zoom compensation is tuned for a count-based window (window size in *samples*); real project's leash is tuned for a spatial bound in *pixels*, and separately has a downstream smoothing stage with its own density assumptions the new stabilizer could disturb. Cannot be ported directly — needs to be redesigned against the real project's EMA/leash model and validated against the SG reconstruction stage at low zoom. |
| Debug systems | Substantial: always-on per-stroke console logging (`[stabilizer cadence]`, `[stabilizer finish]`), a confirmed-unused embedded WASM module, an unreachable `#fatal` overlay, a global `window.setStabDebugConstantPressure` debug hook, a production-facing leash visualization overlay | Not comprehensively audited in `BRUSH_PIPELINE_AUDIT.md` beyond passing references to "telemetry/profiler teardown calls" in `_finalizePointerEndStroke` | **Prototype debug surface should not be migrated; real project's is unknown/out of scope** | The prototype audit itself recommends removing or gating nearly all of its debug systems before any integration (§13, §19.3, §20.3 of `ARCHITECTURE.md`). None of this should cross into the real project. The leash *visualization* is a UI feature question, not part of the stabilization algorithm seam, and is deferred (see §A.9 below). |

---

## A.3. Subsystem Classification (KEEP / REPLACE / ADAPT / REMOVE / LEAVE FOR LATER)

| Subsystem | Classification | Explanation |
|---|---|---|
| Real project: `getPos` (coordinate conversion) | **KEEP** | Encodes the exact inverse of the real project's pan/zoom/rotation/flip chain, which the prototype does not have. No prototype equivalent could replace it correctly. |
| Real project: `_getPressure` (raw read + jump clamp) | **KEEP** | Tablet-driver-quirk handling is empirical, project-specific tuning per the real-project audit; the prototype's pressure reader has no equivalent hardening and is not a suitable replacement. |
| Real project: sample conditioner (`_baselineConditionerPush`, Stage A) | **KEEP** | Always-on, not user-controlled, and structurally unrelated to which stabilization algorithm runs downstream. The prototype has no equivalent stage at all. |
| Real project: user stabilizer (`_stabilizePoint` + RAF catch-up machinery) | **REPLACE** (with new logic informed by the prototype's algorithm) | This is the explicit target of the migration. Not a literal port of the prototype's moving-average FIFO — that algorithm has known contraction limitations (see prototype §16) and a different pressure contract. The replacement must implement the real project's existing seam contract (§16 of `BRUSH_PIPELINE_AUDIT.md`), informed by, but not identical to, the prototype's approach. |
| Real project: pressure curve/smoothing (`_applyPressureCurve`, `_resolveControl`, `_smoothedPressure`) | **KEEP** | User-facing, matched 1:1 with a live UI curve-preview widget. The prototype's pressure model (fixed easing exponent, opacity-ignoring) is not a superset and must not silently override this. |
| Real project: curve/geometry (`_stampQuadCurve`, midpoint reconstruction, arc-walk spacing) | **KEEP, pending manual verification** | Same curve family as the prototype (midpoint-quadratic), which is reassuring for compatibility, but must be manually tested against the new stabilizer's output density/timing before being declared unaffected. |
| Real project: dab rasterization (AA/aliased, tip stamping, scatter/jitter/texture) | **KEEP** | Unrelated to stabilization; explicitly out of scope per the real-project audit. |
| Real project: stroke canvas / commit (`_strokeCanvas`, `_commitStrokeCanvas`, opacity/blend compositing) | **KEEP** | Central to correct stroke-level Opacity/Flow; the new stabilizer only feeds this pipeline, it does not touch it. |
| Real project: layer/frame/cel routing | **KEEP** | Explicitly out of scope for a stabilization migration in both audits. |
| Real project: undo/history | **KEEP** | Explicitly not to be replaced with the prototype's model (stated directly in `BRUSH_PIPELINE_AUDIT.md` §11 and §17). |
| Real project: smart-raster ownership commit | **KEEP** | Layer-type-specific correctness logic, unrelated to stabilization. |
| Real project: tool-switching / frame-change cancellation (`finishActiveDrawingBeforeArtworkChange`, `_endStroke`) | **KEEP**, with **ADAPT** at the integration point | The session-staleness pattern itself is preserved; the new stabilizer's async completion must be wired into it (a genuine integration touch-point, not a rewrite). |
| Real project: airbrush continuous spray | **ADAPT** | Currently reads `lx/ly` (raw last-stabilized position) directly. If the new stabilizer changes how "current position" is exposed, this timer callback must be updated in lockstep or it will silently stop tracking the real brush position. |
| Real project: settings sourcing (`window._ts*` / DOM reads) | **ADAPT** | Not replaced, but the new stabilizer must explicitly decide whether it reads `window._tsStabilization` directly (matching the existing pattern) or receives a resolved value — this needs a decision, not a silent copy. |
| Prototype: moving-average FIFO algorithm (`pushSmoothBuf`, `pushPressureBuf`) | **LEAVE FOR LATER as literal code — informs REPLACE above** | Not ported wholesale (different algorithm family, different pressure contract, known contraction limitation per prototype §16). Its *design lessons* (session ownership, hold catch-up, finish catch-up, zoom compensation) inform the new implementation, but the arithmetic-mean-FIFO mechanism itself is not what gets written into the real project. |
| Prototype: zoom compensation (`zoomSmoothingFactor`, hidden stabilization floor) | **LEAVE FOR LATER** | Tuned for a count-based window model that won't exist in the replacement (which must live inside the real project's EMA/leash model). Needs a fresh design pass against the real project's actual zoom-compensation surface (the leash bound + low-zoom SG reconstruction), not a port. |
| Prototype: WebGPU renderer, WASM module, Canvas2D fallback | **REMOVE (from consideration)** | No real-project equivalent exists to receive this code; out of scope entirely. Not part of this migration in any form. |
| Prototype: debug logging, `#fatal` overlay, `window.setStabDebugConstantPressure` | **REMOVE (from consideration)** | The prototype's own audit already recommends removing most of this before any integration; none of it should be introduced into the real project. |
| Prototype: leash visualization (`drawStabilizerLeash`) | **LEAVE FOR LATER** | A UI/feature decision, not an algorithm-seam decision. Whether the real project gains an equivalent leash overlay is a product question independent of the stabilization migration itself. |
| Prototype: `pointerrawupdate` (unused in prototype) | **N/A** | The real project already has this; nothing to adopt from the prototype here. |

---

## A.4. Compatibility Risks

All risks below are grounded directly in findings from the two source documents; none are speculative.

1. **Duplicate stabilization/conditioning.** The real project's sample conditioner (Stage A) runs unconditionally, independent of the stabilization slider. If the new stabilizer includes its own resampling/duplicate-rejection logic (plausible, given the prototype's own moving-average buffer implicitly does something similar) and it is inserted *in addition to* Stage A rather than *instead of* it, samples could be double-filtered, changing stroke feel unpredictably.

2. **Async finalize contract gap.** The real project relies on `_stabilizerFinalize`/`_stabilizerAccelerateToCompletion` being callable, and completing synchronously when forced, from three call sites: `_pointerEndStroke`, `_endStroke` (conditionally), and `finishActiveDrawingBeforeArtworkChange`. If the new stabilizer does not expose an equivalent "finish now, synchronously if needed" API, tool-switch, frame-switch, or tab-blur cancellation could leave a stroke's tail un-flushed — silently dropping the last few pixels of a stroke, or leaving stray async work targeting a now-wrong layer/frame.

3. **Stale-session callbacks.** The real project's `_activeStrokeSession` check is what currently prevents a slow-converging old stroke's finalize callback from committing into whatever layer/frame is now active. A replacement stabilizer's async completion must be wired into this exact (or an equivalent) session-guard, or a race between "new stroke starts" and "old stroke's stabilizer finally converges" could paint into the wrong layer/frame — a real, previously-guarded-against failure mode, not a hypothetical one.

4. **Pressure/position cadence mismatch.** The real project's current contract is "stabilization owns position only; pressure passes through raw" (unfiltered). The prototype's stabilizer smooths *both* position and pressure via parallel FIFOs kept in lockstep. Migrating the prototype's pressure-buffering behavior would be a silent, unreviewed behavior change unless explicitly decided and signed off on.

5. **0%-stabilization semantics (real project, pre-existing).** The real project's `_stabilizationAmount()` coerces `0 → 1` (max), meaning the documented true-bypass branch in `_stabilizePoint` may be unreachable via the UI slider today. Migrating without resolving this ambiguity risks either silently preserving a possible existing bug, or "fixing" it and unintentionally changing behavior at the 0% setting users currently rely on. This is a pre-existing question, not introduced by the migration, but it must be resolved as part of it since the new stabilizer will need well-defined 0% behavior.

6. **Airbrush timer reading stale position semantics.** `_startAirbrushSpray`'s interval callback reads module-level `lx`/`ly` directly, not through any stabilizer accessor. If the seam changes how "current stabilized position" is exposed or named, this timer must be updated in lockstep, or airbrush-while-held will silently stop tracking the real brush position.

7. **Two RAF loops owning hold/finish.** Currently only the real project's stabilizer owns a hold/finish RAF loop (`_stabilizerStep`); `_scheduleRecomposite` has its own, separate RAF coalescing. If the new stabilizer (following the prototype's `strokeHoldTick`/`finishTick` pattern) brings its own independent RAF loop that isn't unified with the real project's `_stabilizerSchedule` idle-delay/convergence logic, there is a real risk of two competing RAF loops both trying to own "does the brush keep moving while held," producing double-emission or fighting update rates.

8. **Coordinate-unit mismatch.** All downstream real-project geometry (`_curveAddPoint`, `_stampQuadCurve`, dab radius math) operates in real-project canvas-space float pixels, post-`/zoom`, with no supersample factor. The prototype operates in a 4×-supersampled backing-store space via `screenToCanvas`. Inserting new stabilization logic at the seam without correctly re-deriving it in the real project's coordinate convention (not the prototype's) would silently corrupt spacing/curve shape at any zoom level other than exactly matching values.

9. **Low-zoom Savitzky–Golay reconstruction feeding on stabilizer output.** The real project's `_curveAddPoint` applies SG smoothing when `zoom < 1`, and this assumes a dense, evenly-conditioned sample stream from upstream. A new stabilizer that changes sample density or timing characteristics (e.g., emits fewer or burstier points than the current EMA-based one) could interact with this downstream smoothing in ways not exercised by the current implementation. This needs manual low-zoom testing, not an assumption of safety.

10. **Tool-switch coverage outside `brush-engine.js` is unconfirmed.** No direct call from a generic "tool changed" path into `finishActiveDrawingBeforeArtworkChange`/`_endStroke` was found inside `brush-engine.js` for the general tool-switch case — only curve-tool-specific and transform-tool-pointerdown-guard cases exist. Whether some other file calls the cancellation hooks on arbitrary tool switches is unconfirmed from the audited file alone. This is a real gap in the audit's coverage, not a confirmed risk, and should be treated as an open question requiring code inspection outside the audited scope before the migration proceeds past the seam-swap stage.

11. **Undo/session/layer/frame ownership are the real project's, not the prototype's.** The real project's undo model, layer/frame model, and session-guard variable names are all architecturally distinct from the prototype's. Any part of the migration that reuses prototype naming, state shape, or assumptions (e.g., treating the canvas as flat/single-layer, assuming GPU texture undo) risks silently reintroducing prototype-only assumptions into a codebase that does not support them.

12. **WebGPU state — not applicable, but worth stating explicitly.** Since the real project has no WebGPU renderer, there is no WebGPU-state risk to manage during this migration. This is called out only to confirm it was considered and explicitly ruled out, not overlooked.

13. **Brush preset compatibility — not applicable in the algorithmic sense, but relevant to settings sourcing.** The prototype has no preset system to be incompatible with. The actual risk here is narrower: the new stabilizer must correctly read the real project's existing `window._tsStabilization` (or an equivalent single value the real project's UI already produces) rather than inventing a new settings surface that the existing preset system doesn't know about.

14. **Pointer capture — no direct incompatibility found, but not independently re-verified.** Both systems use `setPointerCapture`/pointer-id ownership gates. No conflict is identified in the source documents, but this was not exhaustively cross-checked against the new stabilizer's internals (which do not exist yet) and should be included in the manual test pass regardless.

---

## A.5. First Migration Seam

The seam recommended here is the real project's own — determined by comparing both architectures rather than assuming the real-project audit's suggested seam was self-evidently correct, and cross-checking it against what the prototype actually produces at an equivalent stage.

### Why not the prototype's own natural seam point?

The prototype's most self-contained function is `feedPoint()` — it is the single choke point that turns a stabilized `(x, y)` + pressure sample into rendered geometry. A naive approach might suggest inserting the new stabilizer immediately before whatever function is prototype-`feedPoint()`-equivalent in the real project, i.e., in front of `_curveAddPoint()`. That is in fact roughly where this seam lands — but the more important decision is what sits *immediately upstream* of the stabilizer, and the naive "raw canvas-space position" framing (i.e., seaming directly after `getPos()`) is wrong for the real project specifically:

### Determined seam

```
real project: conditioned sample stream
   { x, y, screenX, screenY, pressure, tiltX, tiltY, twist, time, pointerId, event }
     — this is the output of _baselineConditionerPush / _baselineSampleFromEvent,
       i.e. exactly what currently feeds _stabilizerSetSampleContext + _stabilizePoint
        │
        ▼
   [NEW stabilization/pressure-timing logic]
     — informed by the prototype's algorithm, but re-derived for the real
       project's coordinate space, pressure contract, and session/finalize model
        │
        ▼
   filtered { x, y }  (+ pressure passthrough — preserving the existing
   "stabilization owns position only" contract at _stabilizerEmit, unless
   that contract is explicitly and separately revised)
        │
        ▼
real project: _updateVelocity → _curveAddPoint → _stampQuadCurve → dabs
```

**Why not seam directly on raw `getPos()` output (the naive interpretation):** the real project's always-on sample conditioner (`_baselineConditionerPush`) sits between raw `getPos()` output and the current stabilizer. It is not a stabilization concern — it is a canonical-resampling/anti-duplicate stage that downstream curve reconstruction (the low-zoom SG smoothing) implicitly assumes a reasonably dense, deduplicated sample stream from. Inserting a new stabilizer *before* the conditioner would feed it duplicate/zero-distance samples the conditioner exists specifically to filter out. This is confirmed directly by the real-project audit's own seam analysis and is not contradicted by anything in the prototype document (the prototype has no equivalent conditioning stage at all, so it offers no counter-evidence either way).

### Exact function

There is no single existing function to "insert at" — the seam is a **replacement boundary** spanning:
- `_stabilizePoint(x, y, t)` — the per-sample filtering call
- `_stabilizerAdvance`/`_stabilizerStep`/`_stabilizerSchedule` — the RAF-driven hold/idle-catch-up machinery
- `_stabilizerFinalize`/`_stabilizerAccelerateToCompletion`/`_stabilizerCancel` — the async finish/cancellation contract
- `_resetStabilization` — the per-stroke reset call

### Expected input

A conditioned sample: `{x, y, screenX, screenY, pressure, tiltX, tiltY, twist, time, pointerId, event}`, in real-project canvas-space float pixels (post-`getPos`, post-conditioner), delivered once per emitted canonical sample (zero-to-N per raw pointer event).

### Expected output

`{x, y}` in the same real-project canvas-space float pixel convention. Pressure is **not** filtered by this stage under the existing contract — it passes through as the latest raw/conditioned value, matching `_stabilizerEmit`'s current behavior. Any change to this (i.e., adopting the prototype's pressure-smoothing behavior) is a separate, explicit decision (see Risk 4) and must not be an incidental side effect of the seam swap.

### Required state

- Per-stroke filtered position state (equivalent of `_stabilizerX/Y`, `_stabilizerTargetX/Y`).
- A velocity or rate-of-change signal if the new implementation is time/velocity-sensitive (both the prototype's velocity-paced finish and the real project's velocity-dependent tau rely on something equivalent).
- A hold/idle-catch-up RAF handle, unified with — not parallel to — the real project's existing `_stabilizerRAF`/`_stabilizerSchedule` mechanism (Risk 7).
- A finalize/finishing state flag equivalent to `_stabilizerFinishing`, and a pending-finalize callback slot equivalent to `_stabilizerFinalizeCB`.
- Integration with the real project's `_activeStrokeSession` id — captured at finalize-schedule time and checked before any async completion is allowed to act (Risk 3).

### Required initialization

An equivalent of `_resetStabilization(x, y, t)`, called from `_brushPointerDown` at stroke start, with the same responsibility: seed the filter so stabilization strength is meaningful from the very first sample rather than ramping up.

### Required cleanup

- Cancel any pending RAF/catch-up loop when a stroke is superseded (new pointerdown) or ends (pointerup/pointercancel/tab-blur), exactly as `_endStroke` currently cancels the stabilizer RAF unless a finish is legitimately in progress.
- Provide a synchronous "finish now" path (`_stabilizerAccelerateToCompletion` equivalent) so that `finishActiveDrawingBeforeArtworkChange`, tab-blur, and rapid double-stroke starts can force convergence without waiting out a full animation.

### Why this seam minimizes risk

- It is bounded by two subsystems the real-project audit classifies as **KEEP** on both sides (`_baselineConditionerPush` upstream, `_curveAddPoint` downstream) — neither needs to change, which shrinks the blast radius of the migration to exactly the stabilization/pressure-timing logic itself.
- It preserves the real project's existing async-finalize and session-ownership contracts rather than introducing a parallel or competing set of guarantees (Risks 2, 3, 7).
- It respects the real project's coordinate convention and pressure contract by construction, since it operates entirely downstream of `getPos()`/`_getPressure()` and upstream of `_curveAddPoint()` (Risks 4, 8).
- It does not require any decision about rendering, undo, layers, or presets — none of those subsystems are touched by, or need to know about, a change confined to this seam.

---

## A.6. Migration Order

Each phase is small, independently testable, and reversible via a feature flag. No phase combines large subsystems.

### Stabilization Phase 0 — Instrumentation-only

- **Goal:** Establish a toggle point without changing any behavior.
- **Files affected:** `brush-engine.js` (new flag declaration only).
- **Functions affected:** none functionally; a new flag (e.g., `window._newStabilizerEnabled`, default `false`) is introduced but not yet read anywhere that changes behavior.
- **Expected behavior:** Identical to pre-migration in every respect.
- **Tests:** Confirm the flag exists and defaults to off; confirm no behavior change anywhere in the app.
- **Rollback strategy:** Delete the flag declaration; trivial, no risk.

### Stabilization Phase 1 — New stabilizer as an inert, unit-testable pure function

- **Goal:** Implement the new stabilization algorithm as a standalone function (e.g., `_newStabilizePoint(x, y, t)`), callable but not wired into the live pipeline.
- **Files affected:** `brush-engine.js` (new function(s), no call-site changes).
- **Functions affected:** none of the existing call sites; purely additive.
- **Expected behavior:** No change to the running application. The new function can be exercised in isolation (feed known input sequences, inspect output) via a test harness or console.
- **Tests:** Unit-level only — feed synthetic sample sequences (straight line, held-still, sharp corner, fast flick) and verify the output shape and general smoothing behavior against expectations derived from the prototype's documented behavior, adapted for the real project's EMA/leash-based framing rather than a literal port of the moving-average math.
- **Rollback strategy:** Function is unused; delete it, no risk to the running app.

### Stabilization Phase 2 — Wire the flag into `_stabilizePoint`'s call site only (position, no hold/finish yet)

- **Goal:** Branch to the new implementation for live position filtering when the flag is on, while explicitly leaving hold/finish catch-up as a stub.
- **Files affected:** `brush-engine.js`.
- **Functions affected:** `_stabilizePoint`'s call sites (the two locations noted in the audit as feeding it) branch on the flag.
- **Expected behavior:** With the flag off (default), zero change. With the flag on, live-drawing position filtering uses the new algorithm, but **hold-while-stationary catch-up and release catch-up are not yet implemented** — this phase alone will visibly break "hold" and "finish" feel and must only be tested with those interactions disabled, mocked, or explicitly acknowledged as broken, or shipped together with Phase 3.
- **Tests:** Draw continuous strokes only (no holding still, no careful release timing) at multiple zoom levels and Stabilization slider values, flag on vs. off, comparing general smoothing feel qualitatively.
- **Rollback strategy:** Flip the flag off; old code path is untouched and fully functional.

### Stabilization Phase 3 — Implement the finalize/accelerate-to-completion contract

- **Goal:** Implement the new stabilizer's equivalent of `_stabilizerFinalize`/`_stabilizerAccelerateToCompletion`/`_stabilizerCancel`, and wire it into the exact same call sites as today: `_pointerEndStroke`, `_endStroke` (conditionally), `finishActiveDrawingBeforeArtworkChange`.
- **Files affected:** `brush-engine.js`.
- **Functions affected:** `_pointerEndStroke`, `_endStroke`, `finishActiveDrawingBeforeArtworkChange`, all behind the same flag.
- **Expected behavior:** With the flag on, releasing the pen (with the new stabilizer active) now correctly converges and commits, including the fast-forward path when a new stroke interrupts an old one still converging. The `_activeStrokeSession` staleness check must be preserved and must correctly reject a stale finalize callback.
- **Tests:** Lift pen immediately after a fast flick (no truncation/overshoot); start a new stroke immediately after releasing the previous one (no stray marks from a stale finalize callback); switch tool/layer/frame mid-stroke (stroke commits to its **origin** layer/frame, not the destination); background the tab mid-stroke (stroke commits correctly on return).
- **Rollback strategy:** Flip the flag off; both position and finish logic revert together to the existing implementation.

### Stabilization Phase 4 — Hold-while-stationary catch-up, unified with the existing RAF/idle-delay model

- **Goal:** Implement hold-while-stationary catch-up for the new stabilizer, explicitly unified with (not parallel to) the existing `_stabilizerSchedule`/`_stabilizerStep` idle-delay and convergence-epsilon pattern, to avoid the two-competing-RAF-loops risk (Risk 7).
- **Files affected:** `brush-engine.js`.
- **Functions affected:** the new stabilizer's RAF scheduling logic; no changes to `_scheduleRecomposite`'s independent RAF coalescing.
- **Expected behavior:** Holding the pen stationary mid-stroke visibly catches the brush tip up to the pen position, matching (in spirit, not necessarily in exact timing constants) both the prototype's `strokeHoldTick` behavior and the real project's existing `_stabilizerStep` behavior.
- **Tests:** Hold pen stationary mid-stroke at multiple Stabilization amounts and zoom levels; confirm catch-up animates smoothly and pressure does not drift during the hold (pressure must remain pinned per the existing "position only" contract unless Risk 4 has been separately, explicitly resolved).
- **Rollback strategy:** Flip the flag off; falls back fully to the existing `_stabilizerStep` implementation.

### Stabilization Phase 5 — Zoom compensation and low-zoom interaction validation

- **Goal:** Confirm or adapt the new stabilizer's zoom behavior against the real project's leash-bound zoom scaling and the downstream low-zoom Savitzky–Golay reconstruction (Risk 9), which was not designed against a new upstream sample density/timing profile.
- **Files affected:** `brush-engine.js` (new stabilizer's internal zoom handling only; `_curveAddPoint`'s SG stage is not modified).
- **Functions affected:** the new stabilizer's zoom-dependent internals.
- **Expected behavior:** Stroke quality at low zoom (`zoom < 1`) with the new stabilizer active is comparable to today's behavior — no new jitter, no new over-smoothing artifacts introduced by a sample-density mismatch with the SG stage.
- **Tests:** Draw slow curves, circles, and fast strokes at zoom 25%, 50%, 100%, 400%, comparing flag-on vs. flag-off for smoothness and responsiveness.
- **Rollback strategy:** Flip the flag off.

### Stabilization Phase 6 — Resolve the 0%-stabilization semantics decision

- **Goal:** Explicitly resolve, as a reviewed decision (not a guess), whether Stabilization slider at 0% should produce true bypass or the current coerced minimum behavior, and implement the new stabilizer's 0% behavior accordingly (Risk 5).
- **Files affected:** `brush-engine.js` (the new stabilizer's amount-mapping function only).
- **Functions affected:** the new stabilizer's equivalent of `_stabilizationAmount()`.
- **Expected behavior:** Documented, deliberate 0% behavior, independent of whatever the historical coercion bug/decision was.
- **Tests:** Draw at Stabilization 0% specifically, across zoom levels, with a decision-owner (whoever owns brush feel) explicitly signing off on the observed behavior as correct.
- **Rollback strategy:** This phase is a decision + small implementation change; can be reverted independently of Phases 1–5 since it only affects the amount-mapping function.

### Stabilization Phase 7 — Full manual validation pass with flag on

- **Goal:** Comprehensive manual test pass (see §A.7 below) across real device/pen hardware, all Stabilization amounts (including the resolved 0% behavior), multiple zoom levels, and all interaction patterns.
- **Files affected:** none (testing phase).
- **Expected behavior:** No regressions versus the pre-migration baseline in anything not intentionally changed.
- **Tests:** Full checklist in §A.7.
- **Rollback strategy:** Flag remains off by default until this phase passes cleanly.

### Stabilization Phase 8 — Flip default flag on; retain old path as rollback lever

- **Goal:** Make the new stabilizer the default behavior while keeping the old implementation available behind the (now-defaulted-on) flag for at least one release cycle.
- **Files affected:** `brush-engine.js` (default value of the flag only).
- **Expected behavior:** New stabilizer is what users experience by default; flipping the flag off remains a working rollback path.
- **Tests:** Monitor for regressions reported during the release cycle; re-run the Phase 7 checklist opportunistically.
- **Rollback strategy:** Flip the flag back off; old implementation is still present and untouched.

### Stabilization Phase 9 — Remove old stabilizer code

- **Goal:** Delete the old `_stabilizePoint`/`_stabilizerAdvance`/`_stabilizerStep`/`_stabilizerFinalize` implementation and the flag itself, only after the new stabilizer has been on-by-default for a full release cycle with no regressions reported.
- **Files affected:** `brush-engine.js`.
- **Expected behavior:** Identical to Phase 8's steady state, with dead code removed.
- **Tests:** Full regression pass one more time post-removal, to catch anything that was silently relying on old-path-specific behavior even while the flag was on.
- **Rollback strategy:** None beyond source control revert — this phase is only taken once confidence is high, per the explicit gating condition above.

---

## A.7. Manual Validation After Every Phase

The following checks apply, in whatever subset is relevant, after every phase above:

- **Slow line** — draw a slow, deliberate straight line; confirm smoothing feels appropriate to the Stabilization setting.
- **Fast line** — draw a fast straight line/flick; confirm no truncation, overshoot, or lag artifacts at the endpoint.
- **Circle** — draw a circle or tight curve at high Stabilization amounts; this is the real project's closest analog to the prototype's documented contraction-artifact worst case, and should be specifically compared against pre-migration behavior even though the underlying algorithm is different.
- **Pressure** — vary pressure through a stroke; confirm pressure response feels correct and is not being smoothed by the new position stabilizer (unless that contract has been explicitly changed per Risk 4).
- **Release** — lift the pen at various speeds and after various hold durations; confirm the finish/taper feels natural and consistent with pre-migration behavior.
- **Hold** — hold the pen stationary mid-stroke; confirm catch-up animates to the pen tip smoothly and pressure does not drift.
- **Zoom** — repeat the above at zoom 25%, 100%, and 400%+ (matching both documents' emphasis on zoom-dependent stabilization behavior).
- **Undo** — confirm undo captures exactly one entry per stroke, with correct pre-stroke pixel state, across all scenarios above.
- **Eraser** — repeat relevant checks with the eraser tool active, since it shares the same stroke pipeline.
- **Animation frame** — confirm stroke commit and layer/frame persistence (`saveActiveToKey`) is unaffected; switch frames mid-stroke and confirm correct origin-frame commit.
- **Layer** — switch layers mid-stroke; confirm correct origin-layer commit, including smart-raster ownership metadata.
- **Tool switching** — switch tools mid-stroke (including to tools with no explicit cancellation hook per Risk 10); confirm no stray marks or corrupted state.
- **Double-tap / rapid restart** — start a new stroke immediately after releasing the previous one; confirm no stray marks from a stale finalize callback.
- **Tab blur/background** — background the tab mid-stroke; confirm the stroke commits correctly and the canvas reappears correctly on return.
- **Airbrush continuous spray** — with airbrush + continuous spray enabled, confirm spray still tracks the real (new-stabilizer) position and does not double-stamp.
- **Mouse vs. pen** — confirm mouse (constant pressure) strokes still get a natural stroke-start taper, and pen (variable pressure) strokes behave correctly, on the same settings.
- **Smart-raster layers** — confirm palette/style ownership metadata still commits correctly on smart-raster layers specifically.

---

## A.8. Things That MUST NOT Change

- **Pressure feel** — the real project's existing pressure curves, jump-rate clamping, and driver-glitch handling must remain pixel/behavior-identical unless a pressure-contract change is separately, explicitly approved (see Risk 4).
- **Release/taper feel** — the existing `_getEndTaper`/`_beginEndTaperCapture` taper system is untouched by this migration; only the position-catch-up timing that feeds into it may change.
- **Undo behavior** — one undo entry per stroke, pre-stroke full-canvas (or style-bundle) snapshot model, must be preserved exactly.
- **Animation/frame integration** — `saveActiveToKey`, `ensureKey`, extended-frame handling, and frame-switch stroke-origin routing must all continue to work exactly as today.
- **Layer routing** — `curLayer`/`curFrame` semantics and smart-raster ownership commit must be unaffected.
- **Brush presets and settings sourcing** — the three-pattern settings model (cached DOM reads, `window._ts*` globals, bare globals) must continue to function; the new stabilizer must read whichever source currently holds "Stabilization amount" without introducing a competing settings surface.
- **Tool switching correctness** — session-staleness protection and origin-layer/frame stroke commit on tool/layer/frame switch must be preserved exactly.
- **Session ownership guarantees** — the `_activeStrokeSession` staleness check and its role in preventing cross-stroke corruption must remain fully intact and must govern the new stabilizer's async completion exactly as it governs the old one today.
- **Rendering pipeline** — Canvas2D-only rendering, dab rasterization, tip stamping, scatter/jitter, blend modes, and AA must be completely unaffected; this migration does not touch rendering.
- **Spacing and geometry** — the arc-length dab walk and midpoint-quadratic curve construction downstream of the seam must continue to receive well-formed input and behave identically to today, absent an explicitly tested and approved change in sample density/timing.

---

## A.9. Things Intentionally Deferred

The following are explicitly **not** part of this migration and should not be attempted alongside it, even where the prototype documents identify them as interesting future work:

- **New/alternative stabilization ideas beyond what's needed to reach feature parity** — e.g., arc-length-parameterized smoothing or predictive spline fitting, which the prototype's own audit identifies as *hypothetical* mitigations for its contraction limitation, not implemented features.
- **Curve-preserving research** — investigating whether a fundamentally different smoothing approach avoids geometric contraction on curved paths is a research question, not a migration task. It is out of scope here regardless of how the real project's new stabilizer ultimately performs on circles/tight curves.
- **Performance optimization** — e.g., vertex/array pooling, dynamic supersampling, or any other performance work identified in the prototype's audit as speculative or profiling-dependent. Migrate existing behavior first; optimize only if profiling on the real project identifies an actual bottleneck.
- **Cleanup/refactoring beyond what this plan requires** — e.g., extracting the duplicated `getCoalescedEvents()` pattern in the prototype, or unifying the real project's three settings-sourcing patterns into one object. These are legitimate future cleanup items noted in both audits, but are unrelated to shipping the stabilization migration correctly.
- **New rendering systems** — adopting WebGPU, adding supersampling, or any other rendering-layer change. The real project has no WebGPU path today and none is introduced by this migration.
- **Leash visualization / new debug UI** — whether to add a prototype-style leash overlay to the real project is a separate product/UX decision, independent of the stabilization algorithm itself.
- **Tilt-based dynamics** — both documents note that tilt/twist data is captured but not consumed downstream in either codebase. Wiring it up (in either project) is out of scope here.
- **Resolving the tool-switch coverage open question (Risk 10) beyond flagging it** — determining whether code outside `brush-engine.js` calls the cancellation hooks on generic tool switches requires inspection outside this audit's scope and is deferred to a follow-up investigation, not blocking the seam-level migration itself (though it should be resolved before Phase 8's default-flip, since it affects correctness under a real interaction pattern).

---

## A.10. Success Criteria (Deliverable A only)

"Migration complete" means all of the following are true:

- The new stabilizer (informed by, but not a literal port of, the prototype's algorithm) is the real project's default stroke-position stabilization implementation, and the old `_stabilizePoint`/RAF-catch-up implementation has been removed (Phase 9 complete).
- Every manual test in §A.7 passes with no regressions versus the pre-migration baseline, across all Stabilization amounts (including a deliberately-resolved 0% behavior), multiple zoom levels, and both mouse and pen input.
- The 0%-stabilization semantics question (Risk 5) has been explicitly resolved and signed off on, not left ambiguous.
- All existing tools continue working, including tool-switch-mid-stroke and rapid-restart scenarios.
- No undo regressions — one entry per stroke, correct pre-stroke state, across bitmap and smart-raster layers.
- Pressure behavior is unchanged (or, if intentionally changed per an explicit decision on Risk 4, the change is documented and approved, not incidental).
- Animation/frame and layer routing are unchanged, including the extended-frame mechanism and smart-raster ownership commit.
- Rendering (Canvas2D dab rasterization, blend modes, AA, tip stamping) is completely unchanged — this migration never touches it.
- The airbrush continuous-spray mechanism correctly tracks the new stabilizer's position output (Risk 6 resolved).
- The two-RAF-loop risk (Risk 7) has been verified absent — the new stabilizer's hold/finish catch-up is unified with, not parallel to, the real project's existing RAF/idle-delay model.
- Low-zoom behavior (interaction with the Savitzky–Golay reconstruction stage, Risk 9) has been specifically tested and found free of new artifacts.
- The old stabilizer implementation is fully removable with no remaining call sites, and has in fact been removed (Phase 9), after at least one full release cycle with the new implementation on by default and no reported regressions.

---

# Deliverable B — Animated Brush-Dab Preview

**Independent of Deliverable A.** Different prototype code, different real-project code, different files, different feature flag, different commits/phases. Nothing below alters painted stroke geometry, spacing, dab rasterization, pressure curves, brush presets, custom-tip behavior, Canvas2D stroke rendering, layer/frame routing, undo, smart-raster ownership, or stabilization hold/finish behavior.

## B.1. What "animated brush dab" means in the prototype (audit)

The prototype has **no dedicated animation loop** for this feature — it is not `requestAnimationFrame`-driven. It is an **event-driven live preview overlay**: a circle (or, while a stroke is in progress, a ring) that tracks the pointer and is redrawn from scratch on every relevant input event. "Animated" describes its continuous, per-event responsiveness to hover position, pressure, drawing state, and zoom — not a frame-loop animation.

**Exact mechanism, traced end-to-end:**

```
pointerenter / pointermove / pointerdown / pointerup / pointercancel
(hoverX, hoverY, hoverPointerType, hoverPressure, hovering, drawing — module-level state, lines ~1849–1998)
        ↓
updateBrushPreview()                              [line 2110]
   branches on: isSizeDragging → return (owned by its own preview)
                isPanning       → hideSizePreview()
                drawing         → showSizePreview(..., outlineOnly=true)   [live-drawing ring]
                isZoomDragging  → showSizePreview(pinned at drag-start anchor)
                previewSuppressedUntilMove → hideSizePreview()
                !hovering       → hideSizePreview()
                hoverPointerType==='pen' → showSizePreview(fixed lightest-pressure diameter)
                else            → showSizePreview(normal)
        ↓
showSizePreview(anchorX, anchorY, size, pressure, fixedDiameter, outlineOnly)   [line 2010]
   - computes rBacking via pressureRadius(size, pressure)                  [line 428]
   - computes cBackingX/Y: snaps the preview's origin to an integer
     backing-store pixel so the dab's antialiased edge sits at the same
     texel-grid phase as a real painted dab (lines 2027–2035, 2046–2050)
   - sets sizePreviewPixelated via updatePixelatedState(zoom, ...)         [line 287]
     (nearest-neighbor once zoom ≥ 1.2, bilinear once zoom ≤ 1.0, hysteresis
     holds the prior mode between the two thresholds)
   - clears and redraws into the #sizePreview 2D canvas context:
       outlineOnly        → 1px stroked ring (live-drawing indicator only,
                             never painted into the stroke)
       tool==='eraser'    → checkerboard fill clipped to the dab circle
       else               → solid fill, alpha = pressureAlpha(pressure)
                             (which, per line 444, is just `opacity` —
                             pressure does NOT affect alpha, only radius)
        ↓
#sizePreview <canvas> (a standalone DOM element, CSS position/size/left/top
set directly; NOT part of the WebGPU/WASM stroke pipeline, NOT part of the
strokeMaskTex/accTex/blit/composite chain — see §1.3 of ARCHITECTURE.md)
        ↓
visible result: a live circle (filled dab preview while hovering, thin ring
while actively drawing) that tracks the pointer, resizes/dims with pressure,
and re-renders at the correct blocky/smooth appearance for the current zoom.
```

**Findings against the audit checklist:**

| Question | Finding |
|---|---|
| Exact feature behavior | A live, non-painting cursor/dab preview: filled circle while hovering, thin outline ring while a stroke is in progress. Never writes into the stroke mask or accumulated canvas — confirmed independent of the painted-stroke pipeline. |
| Exact functions | `showSizePreview()` (2010), `hideSizePreview()` (2103), `updateBrushPreview()` (2110), plus the shared helpers `pressureRadius()` (428), `pressureAlpha()` (444), `updatePixelatedState()` (287), `wrapToCanvas()`/`canvasToWrap()` (2003–2008). |
| State variables | `hoverX`/`hoverY`, `hoverPointerType`, `hoverPressure`, `hovering`, `previewSuppressedUntilMove`, `sizePreviewPixelated`, plus read-only reuse of `drawing`, `isSizeDragging`, `isPanning`, `isZoomDragging`, `zoomDragStart`, `tool`, `color`, `zoom`/`panX`/`panY`, `SS`. |
| Overlay or destination canvas | Overlay only — `#sizePreview`, a dedicated `<canvas>` (`z-index:6`, `pointer-events:none`), entirely separate from the main paint `#stage`/WebGPU canvas and from `#stabViz` (the leash overlay, `z-index:7`). |
| Hover, drawing, or both | Both, via different render modes: filled dab while hovering (not drawing), thin outline ring while `drawing===true` (see `outlineOnly` branch, `updateBrushPreview` line 2113–2116). |
| Pressure behavior | Radius scales with pressure via `pressureRadius()`/`pressureInfluence(p^1.2)`; alpha does **not** respond to pressure (`pressureAlpha()` always returns the global `opacity`, by explicit design per the code comment at line 441–443). |
| Size behavior | Diameter is `size * SS`-based, matching the real dab's backing-store radius exactly (`pressureRadius(size, pressure)`, same function the actual brush geometry would use). |
| Zoom behavior | CSS size/position scale by `zoom` (`cssSize = backingSize / SS * zoom`); backing-store resolution itself is zoom-independent (fixed `SS=4`), matching the main canvas's supersampling model. |
| Pan behavior | Preview position is computed via `wrapToCanvas`/`canvasToWrap`, which include `panX`/`panY` — pans with the canvas. Hidden entirely while `isPanning` is true. |
| Pixelated/bilinear behavior | Independently tracked hysteresis state (`sizePreviewPixelated`, separate from the main canvas's `canvasPixelated`) via the same `updatePixelatedState()` function and the same 1.2/1.0 zoom thresholds — see ARCHITECTURE.md §6.4. |
| Subpixel / pixel-phase alignment | Yes — this is the feature's most distinctive property. The preview's origin is snapped to an integer backing-store pixel and drawn at the fractional center within that box, so its blocky/antialiased edge shifts in sync with the real canvas's texel grid as the pointer moves, rather than sliding independently (explicit design comment, lines 2026–2035). |
| Pen vs. mouse behavior | Pen hover uses a fixed "lightest pressure" diameter (`MIN_ABS_RADIUS_BACKING`-derived) independent of the selected brush size, so a hovering (not-yet-touching) pen shows the true lightest possible mark rather than the full brush size; mouse hover shows the full configured brush size directly. |
| Brush vs. eraser behavior | Brush fills with `color`; eraser fills with a checkerboard pattern (visually distinct "erase" indicator) instead of a solid color. |
| RAF vs. event-driven | **Event-driven, not RAF-driven.** `updateBrushPreview()` is called synchronously from `pointermove`/`pointerenter`/`pointerup`/`pointercancel`/drag-loop handlers and from UI control `oninput` handlers (color/size/opacity). No `requestAnimationFrame` loop exists for this feature anywhere in the file (confirmed by exhaustive `anim`/RAF search — the file's only RAF usages are all in the stabilization hold/finish machinery, an unrelated system). |
| Show/hide rules | Hidden during panning, during size-drag (which shows its own preview), when `previewSuppressedUntilMove` is set (e.g., immediately after certain drag-start/drag-stop transitions, to avoid a stale flash at the old position), and on `pointerleave`. Shown on `pointerenter`/hover and while drawing. |
| Independent of painted-stroke pipeline | Confirmed independent — `showSizePreview()` only ever writes to the `#sizePreview` 2D canvas context; it has no reference to `strokeMaskTex`, `accTex`, `beginStrokeCoverage`, `compositeStroke`, or any WebGPU pipeline object. |

**Not the same as WebGPU stroke rendering.** The prototype's WebGPU stroke pipeline (`strokePipeline`/`blitPipeline`/`compositePipeline`, ARCHITECTURE.md §1.3) is what paints the *actual* stroke. The animated-dab preview is a wholly separate, plain-Canvas2D overlay with no code path into or out of the WebGPU pipeline. No part of this deliverable requires or implies migrating WebGPU.

## B.2. Real-project equivalent (audit)

The real project already has a live brush cursor/preview system — found by direct source inspection, since `BRUSH_PIPELINE_AUDIT.md` does not cover it (that document is scoped entirely to `brush-engine.js`'s stroke pipeline).

- **File:** `src/brush/brush-size-cursor.js` (entire file is this feature; self-contained IIFE).
- **Supporting file:** `src/brush/brush-size-drag.js` — a separate, second preview (`#brush-size-preview` + `#brush-size-preview-label`) shown only while the brush-resize keybind ("S" by default) is held/dragged; it explicitly coordinates with `brush-size-cursor.js` via a `brush-resize-preview-toggle` event so the two previews never show at once.
- **Supporting file:** `src/ui/cursor-prefs.js` — `cursorStyle` preference (`'crosshair'`, `'point'`, `'brush'`, `'brush-shape'`), persisted to `localStorage`, sets the native canvas CSS `cursor` to `none` for brush/eraser so the overlay canvas is the only visible cursor.

**Current cursor/preview canvas:** `#brush-cursor-overlay`, a `document.createElement('canvas')` appended to `document.body` (not inside the canvas stack), `position:fixed`, `z-index:9998`, `pointer-events:none`, sized in device pixels via `prepare(cssSize)` (line 24) with explicit `devicePixelRatio` scaling.

**Input events that update it:** `pointerenter` (capture) and `pointermove` (window-level, capture) both call `track(event)` (line 97), which updates `hovering`/`lastX`/`lastY`/`lastPointerType` and calls `update(false)`; `pointerleave`, `pointerup`, `pointercancel`, `blur` also update visibility/position. Two custom events additionally trigger a forced redraw: `tool-changed` and `brush-resize-preview-toggle`.

**Current size and pressure response:** Size responds to `toolSizes[tool]` and `zoom` via `brushDiameter()` (line 45). **Pressure is not read anywhere in this file** — the cursor is a fixed-size ring at the *configured* brush size; it does not shrink/dim to preview live pressure the way the prototype's hovering pen preview does.

**Zoom and transform handling:** `brushDiameter()` multiplies by `zoom` directly. No pan handling is needed (the overlay tracks raw `clientX`/`clientY`, which is already screen-space). No explicit rotation/flip handling of the ring itself, though `drawShape()`'s custom-tip mode does apply `angle`/rotation and `flipX`/`flipY` to the *tip stamp* it draws (see below) — the outer ring in `drawCircle()` does not rotate (a circle has no rotational asymmetry to show).

**Brush/eraser behavior:** Eraser always forces the plain ring (`drawCircle()`), regardless of the user's chosen `cursorStyle`, per an explicit code comment (line 87–88, "Eraser always shows the circle regardless of cursor style pref").

**Custom-tip support:** Yes, and **this is functionality the prototype does not have at all.** `drawShape()` (line 54), used when `cursorStyle==='brush-shape'`, draws the actual `window.brushTipCanvas` image (if a custom tip is set) scaled to the current diameter, rotated by `angle` (live per-dab rotation while drawing, via `_activeDabRotation`, or the static `window._tsBrushAngle` while idle), flipped per `brushTipFlipX`/`brushTipFlipY`, and compressed by `roundness` on whichever axis is narrower — then rendered with a light silhouette fill plus a drop-shadow so it reads clearly against any canvas background color.

**Hardness, roundness, angle, antialiasing support:** Roundness and angle are supported (`drawShape()`, above) when `cursorStyle==='brush-shape'`. Hardness/antialiasing-mode are **not** visually represented in any cursor mode — the ring/tip-silhouette is a size-and-shape indicator only, not a soft-edge preview of the actual painted falloff.

**When hidden/shown:** `shouldShow()` (line 19) — hidden during: not a paint tool, not hovering and not actively drawing, active layer group, panning, zoom-drag, rotate-drag, space-held pan, `_brushResizePreviewActive` (the S-key resize preview is showing instead), or transform-tool active.

**Whether it already animates:** **Yes — already RAF-driven**, unlike the prototype. Line 111: `(function loop(){update(false);requestAnimationFrame(loop);})();` — a persistent per-frame loop that has been running since the file loaded, gated by `update()`'s own internal `shouldShow()`/signature-diffing so it's cheap when nothing changes (`signature()`, line 77, memoizes the last-drawn state and skips the redraw if nothing relevant changed, only updating position every frame).

**Missing compared with the prototype:**
- No pressure response at all (prototype: radius shrinks with pressure; pen-hover shows a fixed lightest-pressure dab).
- No subpixel/pixel-phase alignment to the canvas's own rendering grid (prototype: the dab's origin is snapped to the canvas's backing-pixel grid so its antialiased edge visually tracks the real dab).
- No zoom-dependent pixelated/bilinear hysteresis matching the main canvas's own crisp/soft switch (the real project *does* have an equivalent single-threshold switch for the main canvas itself — `core-state.js:431`, `useNN = zoom >= 1.5`, no hysteresis — but it is not applied to `#brush-cursor-overlay` or `previewCanvas`).
- No filled-dab hover mode — the real project's ring is always an outline (except the tip-silhouette mode, which is a translucent silhouette, not the actual brush color/opacity).
- No distinct "actively drawing" visual state (prototype switches from filled dab to thin outline ring while `drawing===true`; real project's ring looks the same whether hovering or drawing).

## B.3. Comparison table

| Behavior | Prototype | Real Project | Migration Decision | Reason |
|---|---|---|---|---|
| Hover preview | Filled dab, color/opacity-matched, pressure-responsive radius | Outline ring (or tip silhouette in `brush-shape` mode), no pressure response | **PORT FROM PROTOTYPE** | Filled, pressure-responsive hover preview is materially more informative and is the prototype's clearest win; nothing in the real project's ring conflicts with adding it as a new visual mode. |
| Drawing-time preview | Switches to thin outline ring while `drawing===true` | No distinct drawing-time visual — ring looks identical whether hovering or drawing | **PORT FROM PROTOTYPE** | Small, self-contained state check (`drawing`) already exists as `strokeActive()` in the real project; wiring the same ring-during-draw behavior is low-risk. |
| Pressure animation | Radius scales with pressure (`pressureInfluence(p^1.2)`); alpha does not | Not implemented | **PORT FROM PROTOTYPE** | Direct feature gap; real project already reads live pressure elsewhere (`_getPressure`), so the value is available, just not currently wired to the cursor. |
| Size animation | Diameter matches `pressureRadius()`, the same function real dab geometry would use | `brushDiameter()` = `toolSizes[tool] * zoom`, static per configured size | **ADAPT/BRIDGE** | Real project has no backing-store/SS concept to match against (Canvas2D, no supersampling) — the *contract* (preview radius should track true dab radius including pressure) is worth porting, but the literal `pressureRadius()` math is prototype-specific and must be bridged to the real project's own dab-radius calculation, not copied verbatim. |
| Zoom scaling | CSS size scales by `zoom`; backing store is zoom-independent (`SS=4` fixed) | Diameter scales by `zoom` directly (no supersampling concept) | **KEEP REAL PROJECT** | Real project has no supersampled backing store to preserve fidelity against; its direct `zoom` scaling is already the correct analog. |
| Pixel-phase alignment | Preview origin snapped to canvas's backing-pixel grid so it visually tracks real dab placement | Not implemented — overlay position follows raw client coordinates only | **NEEDS MANUAL TESTING** | Real project's main canvas has its own coordinate/DPR handling (`getPos`, flip/rotation/pivot) that the prototype's simpler `wrapToCanvas` does not account for; whether phase-snapping is visually meaningful (and how to compute it correctly) against the real project's transform stack needs to be verified against actual on-screen behavior, not assumed from either document alone. |
| Pixelated/bilinear switching | Independent hysteresis state for the preview canvas, matching the main canvas's crisp/soft switch | Main canvas has a single-threshold switch (`useNN=zoom>=1.5`); overlay canvas has none | **ADAPT/BRIDGE** | The real project's existing single-threshold rule (not hysteresis) is the reference to bridge to the overlay canvas — introducing the prototype's separate hysteresis constants (1.2 in / 1.0 out) would create two different zoom-switch behaviors for the main canvas vs. the cursor, which is itself a new inconsistency; align the overlay to whichever rule the main canvas ends up using. |
| Pointer ownership | Single global hover state (`hoverX/Y`, `hovering`), no pointer-capture interaction (no `setPointerCapture` conflict since no stroke is active while hovering) | Same general approach (`lastX/Y`, `hovering`), already coordinates with `_activeStrokePointerId`/pointer capture via `strokeActive()` | **KEEP REAL PROJECT** | Real project's version already exists, is already tested against pointer-capture during live strokes, and needs no change. |
| Mouse behavior | Full brush size shown directly on hover | Same (no pen-specific lightest-pressure gating) | **KEEP REAL PROJECT** | No prototype behavior to add here beyond what's already covered by the "Pressure animation" row. |
| Pen behavior | Fixed lightest-pressure diameter shown on pen hover (before contact), independent of configured brush size | Not implemented — pen hover shows full configured size like mouse | **PORT FROM PROTOTYPE** | Meaningful UX signal (shows the true lightest mark a light pen touch would produce) with a well-defined trigger (`hoverPointerType==='pen'`) already available in the real project's pointer event data. |
| Eraser behavior | Checkerboard fill instead of solid color | Forces plain ring (`drawCircle()`) regardless of `cursorStyle`, no checkerboard | **ADAPT/BRIDGE** | Real project's "eraser always shows a ring" rule is a deliberate existing decision (explicit code comment); the new filled-hover mode (from "Hover preview" row) should adopt a checkerboard fill for eraser specifically, consistent with the prototype, without disturbing the ring-only rule for `brush-shape`/other cursor styles if a product decision is made to keep those ring-only for eraser. |
| Custom brush tips | Not supported — prototype has no custom-tip system at all | Fully supported in `brush-shape` cursor mode (`drawShape()`): rotation, flip, roundness compression, silhouette + drop-shadow rendering | **KEEP REAL PROJECT** | Real project is strictly more capable here; nothing to port from the prototype, which has no equivalent concept. |
| Hide/show lifecycle | Suppressed during pan/zoom-drag/size-drag; `previewSuppressedUntilMove` flag prevents stale-position flashes | `shouldShow()` already covers pan/zoom-drag/rotate-drag/space-pan/transform-tool/layer-group/resize-drag-active | **KEEP REAL PROJECT** | Real project's show/hide logic is already broader (covers rotate-drag and transform-tool, which the prototype has no equivalent of) and already solves the "don't show two previews at once" problem via the `brush-resize-preview-toggle` event. No prototype behavior needed here beyond the drawing-time ring state (already listed above). |
| Rendering destination | `#sizePreview`, independent `<canvas>` inside the same DOM stack as the paint canvas (`z-index:6`), plain Canvas2D | `#brush-cursor-overlay`, independent `<canvas>` appended to `document.body` (outside the canvas stack), `position:fixed`, plain Canvas2D | **KEEP REAL PROJECT** | Both are already independent overlay canvases with no interaction with the paint pipeline; real project's existing element is the correct integration target — no new canvas element should be introduced. |
| Performance / update cadence | Event-driven only (pointermove/enter/leave/up/cancel + UI `oninput`); no RAF loop | **Already RAF-driven** (`requestAnimationFrame` loop since file load, line 111) with signature-based memoization to skip redundant redraws | **KEEP REAL PROJECT** | Real project's cadence model is a strict superset of the prototype's (continuous per-frame position tracking *and* change-detected redraw skipping) — porting the prototype's purely event-driven model would be a regression, not an improvement. New behaviors (pressure, drawing-ring, pen-lightest-dab) should be added as new inputs to the existing `signature()`/`update()` functions, not as a second update loop. |

## B.4. Feature boundary

Deliverable B changes only the visual appearance and input-responsiveness of `#brush-cursor-overlay` (and, where needed, coordinates with `#brush-size-preview`'s existing hide/show handshake). It must not alter:

- `_stampDab`/`_drawDabNow`/`_buildTipStamp`/`_buildSoftRoundMask` or any other actual dab rasterization
- `_effectiveSpacingFrac`/`_effectiveDabStep` spacing/placement
- `_applyPressureCurve`/`_resolveControl`/pressure curves used for the *painted* stroke
- brush presets (`brush-presets.js`) beyond reading existing values (tip image, roundness, angle, flip) the cursor already reads today
- any custom-tip rendering used for the actual stroke
- Canvas2D stroke rendering (`_drawDabNow` et al.)
- layer/frame routing, undo, or smart-raster ownership
- the stabilization hold/finish behavior that is the subject of Deliverable A

No evidence in either the prototype source or the real-project source suggests any of the above needs to change to implement Deliverable B; if a future implementation step discovers otherwise, that would require a separate, explicitly-justified decision, not an incidental change made "while in the area."

## B.5. Animated-dab migration seam

**The proposed shape in the task prompt is essentially correct, verified against the actual source:**

```
real-project pointer/hover state (lastX/Y, hovering, lastPointerType — brush-size-cursor.js)
  + resolved brush settings (toolSizes[tool], window.brushTipCanvas, roundness, angle, flip)
  + live pressure (NEW input — not currently read by brush-size-cursor.js; available via the
    same _getPressure()/pointer-event pressure value brush-engine.js already reads)
  + zoom/transform (zoom, already read; pan/rotation NOT currently needed since the overlay
    tracks raw client coordinates, not canvas-space coordinates)
        ↓
animated dab-preview state (extend the existing signature()/update() in brush-size-cursor.js —
  do not introduce a second state object or a second file)
        ↓
existing overlay renderer (drawCircle()/drawShape()/new drawFilledDab()-style function,
  still inside brush-size-cursor.js, still targeting #brush-cursor-overlay)
```

**Exact input contract:** `{clientX, clientY, pointerType, pressure}` from the same `pointermove`/`pointerenter` events `track()` already listens to (`brush-size-cursor.js:97-108`), plus a new pressure read added to `track(event)` (currently discards everything from the event except `clientX`/`clientY`/`pointerType`).

**Exact output contract:** no change to the *type* of output — still a 2D-canvas redraw of `#brush-cursor-overlay`, gated by `shouldShow()`, deduplicated by an extended `signature()`. The new visual modes (filled hover dab, drawing-time ring, pen-lightest-dab, pressure-scaled radius) are additional draw functions selected by the same `if/else` chain `update()` already uses to pick `drawPoint()`/`drawCross()`/`drawCircle()`/`drawShape()`.

**Files/functions involved:** `src/brush/brush-size-cursor.js` — `track()` (add pressure capture), `signature()` (add pressure + drawing-state to the memoization key), `update()` (add a drawing-time branch and a filled-dab branch), `shouldShow()`/`strokeActive()` (already suitable, no change needed), a new draw function (e.g., `drawFilledDab()`) alongside the existing `drawPoint`/`drawCross`/`drawCircle`/`drawShape`.

**State to initialize:** a `lastPressure` variable (parallel to existing `lastX/lastY/lastPointerType`), and — only if the pixel-phase-alignment question from §B.3 is resolved in favor of porting it — a per-frame computed snap-to-grid offset (no persistent state needed beyond what `update()` already recomputes each call).

**Update cadence:** unchanged — the existing RAF loop (`brush-size-cursor.js:111`) plus the existing `track()` event-driven position updates. No second loop.

**Cleanup/hide behavior:** unchanged — `shouldShow()` already governs visibility; the new filled-dab/drawing-ring modes are additional *draw* branches inside the existing show/hide gate, not a new lifecycle.

**Which existing real-project systems remain untouched:** `brush-engine.js` (the entire stroke pipeline — this seam only reads pressure, it does not call into brush-engine.js), `brush-presets.js`, `_stabilizerStep`/Deliverable A's entire seam, undo, layers/frames, `cursor-prefs.js`'s CSS-cursor-selection logic (still governs whether the overlay is shown at all vs. a native cursor), `brush-size-drag.js`'s existing S-key preview and its `brush-resize-preview-toggle` handshake.

**Compatibility risks:** see §B.8.

## B.6. Scope statement (see also the revised top-of-document scope block)

This migration has two independent deliverables:

1. Prototype-derived position stabilization behavior (Deliverable A, above).
2. Prototype-derived animated brush-dab preview behavior (Deliverable B, this section).

The following remain owned by the real project, untouched by either deliverable:

- The actual brush rasterizer (`_stampDab`, `_drawDabNow`, tip-stamp builders)
- Spacing and dab placement (`_effectiveSpacingFrac`, `_effectiveDabStep`, arc-length walk)
- Brush tips and dynamics (scatter, jitter, custom-tip images)
- Pressure curves used for the painted stroke (`_applyPressureCurve`, `PRESSURE_CURVES`)
- The rendering backend (Canvas2D — no WebGPU is introduced by either deliverable)
- Undo/history
- Layers, frames, and cels
- Presets and Tool Settings (beyond the cursor's existing read-only use of a few preset fields)

## B.7. Animated Dab migration phases

Small, reversible, feature-flagged. Never combined with a Stabilization Phase (§A.6) in the same commit.

### Animated Dab Phase 0 — Read-only/runtime baseline
- **Goal:** Confirm current behavior and establish the toggle point.
- **Files affected:** `src/brush/brush-size-cursor.js` (new flag declaration only, e.g. `window._animatedDabPreviewEnabled = false`).
- **Functions affected:** none functionally.
- **Expected behavior:** Identical to today in every respect.
- **Manual tests:** Confirm the flag exists, defaults off, and no visual change occurs anywhere.
- **Rollback strategy:** Delete the flag declaration; trivial, no risk.

### Animated Dab Phase 1 — Add feature flag / isolated preview state
- **Goal:** Add `lastPressure` tracking and drawing-state awareness to `track()`/`signature()` without changing any rendering yet.
- **Files affected:** `src/brush/brush-size-cursor.js`.
- **Functions affected:** `track()`, `signature()`.
- **Expected behavior:** No visual change (flag still off); pressure is captured and stored but unused.
- **Manual tests:** Console-inspect `lastPressure` updates as pen pressure varies; confirm still no visual change.
- **Rollback strategy:** Revert the two function bodies; flag remains off throughout, zero user-facing risk.

### Animated Dab Phase 2 — Port the basic visual behavior
- **Goal:** Implement `drawFilledDab()` (hover) and a drawing-time ring variant, gated behind the flag.
- **Files affected:** `src/brush/brush-size-cursor.js`.
- **Functions affected:** `update()` (new branches), new `drawFilledDab()`.
- **Expected behavior:** With flag on: filled dab on hover, thin ring while drawing. No pressure/size response yet (flat radius = configured brush size).
- **Manual tests:** Hover shows filled dab; start drawing, dab switches to ring; release, returns to filled dab.
- **Rollback strategy:** Flip flag off; falls back to existing `drawCircle()`/`drawShape()` ring exactly as today.

### Animated Dab Phase 3 — Integrate pressure and size response
- **Goal:** Wire `lastPressure` into the dab radius calculation, bridged against the real project's own dab-radius logic (not a literal port of `pressureRadius()`).
- **Files affected:** `src/brush/brush-size-cursor.js`.
- **Functions affected:** `drawFilledDab()`, `brushDiameter()` (or a new pressure-aware variant).
- **Expected behavior:** Dab radius shrinks with lighter pressure on pen input; mouse retains full configured size (no pressure signal to shrink from).
- **Manual tests:** Vary pen pressure while hovering (not drawing); confirm radius tracks pressure smoothly. Confirm mouse hover is unaffected.
- **Rollback strategy:** Flip flag off.

### Animated Dab Phase 4 — Integrate zoom and transform behavior
- **Goal:** Confirm existing `zoom`-based scaling (already present in `brushDiameter()`) behaves correctly for the new filled-dab/pressure-radius mode; add pen-lightest-dab hover behavior.
- **Files affected:** `src/brush/brush-size-cursor.js`.
- **Functions affected:** `drawFilledDab()`, `update()` (pen-hover branch).
- **Expected behavior:** Filled dab scales correctly with zoom; pen hover (not touching) shows a fixed, small "lightest touch" dab independent of configured brush size.
- **Manual tests:** Hover with pen at 25%, 100%, 400% zoom; confirm dab scales correctly and lightest-touch behavior is visible and consistent across zoom levels.
- **Rollback strategy:** Flip flag off.

### Animated Dab Phase 5 — Match pixelation/subpixel behavior
- **Goal:** Resolve the §B.3 "NEEDS MANUAL TESTING" item — decide and implement whether/how the overlay should align to the main canvas's pixelated/bilinear switch and/or backing-pixel phase.
- **Files affected:** `src/brush/brush-size-cursor.js`, possibly `src/core/core-state.js` (read-only reference to `useNN`/zoom threshold, no behavior change to the main canvas).
- **Functions affected:** `drawFilledDab()` or a new helper for pixel-phase snapping.
- **Expected behavior:** A deliberate, tested decision — either the overlay matches the main canvas's crisp/soft switch, or a documented reason why it does not.
- **Manual tests:** Zoom across the 150% threshold (real project's existing `useNN` cutoff) repeatedly; visually compare overlay crispness against the main canvas's own dab rendering at the same zoom.
- **Rollback strategy:** Flip flag off.

### Animated Dab Phase 6 — Validate brush, eraser, mouse, pen, and custom tips
- **Goal:** Confirm eraser checkerboard fill (new, ported from prototype) coexists correctly with the existing `brush-shape`/custom-tip cursor mode, and that all `cursorStyle` preferences still behave sensibly with the new filled-dab mode layered in.
- **Files affected:** `src/brush/brush-size-cursor.js`.
- **Functions affected:** `update()`'s cursor-style branch selection, `drawFilledDab()` (eraser checkerboard branch).
- **Expected behavior:** Eraser hover shows a checkerboard-filled dab (or a documented decision to keep eraser ring-only, per the open question in §B.3's "Eraser behavior" row); custom-tip (`brush-shape`) mode is unaffected or intentionally extended, not accidentally broken.
- **Manual tests:** Cycle through all four `cursorStyle` preferences with brush and eraser, with and without a custom tip loaded, on mouse and pen.
- **Rollback strategy:** Flip flag off.

### Animated Dab Phase 7 — Enable by default
- **Goal:** Flip the flag's default to `true`, keeping the old (ring-only) behavior available behind the flag as a rollback lever for one release cycle.
- **Files affected:** `src/brush/brush-size-cursor.js` (default value only).
- **Expected behavior:** New animated dab preview is what users see by default.
- **Manual tests:** Full §B.9 checklist.
- **Rollback strategy:** Flip flag back off; old ring-only rendering is untouched and fully functional.

### Animated Dab Phase 8 — Remove obsolete preview code only after validation
- **Goal:** Remove the flag and any now-dead old-path branches, only after a full release cycle on-by-default with no regressions reported.
- **Files affected:** `src/brush/brush-size-cursor.js`.
- **Expected behavior:** Identical to Phase 7's steady state, dead code removed.
- **Manual tests:** Full regression pass post-removal.
- **Rollback strategy:** None beyond source control revert — taken only once confidence is high.

## B.8. Compatibility risks

- **Duplicate cursor/preview renderers:** if the new filled-dab logic is added as a *second* overlay canvas instead of extending `#brush-cursor-overlay` in place, the app would end up with two competing cursor elements (in addition to the pre-existing `#brush-size-preview` S-key overlay, which already has a coordination handshake with `#brush-cursor-overlay` via `brush-resize-preview-toggle`). The seam in §B.5 is deliberately designed to extend the existing element/file to avoid this.
- **Two update loops:** the real project already has exactly one RAF loop for this feature (`brush-size-cursor.js:111`). Introducing a second, prototype-style event-only update path alongside it (rather than feeding new inputs into the existing loop's `update()`/`signature()`) would risk the same "two competing loops" failure mode flagged as Risk 7 in Deliverable A, applied to a different subsystem.
- **Wrong coordinate space:** `#brush-cursor-overlay` currently uses raw `clientX`/`clientY` (screen space), while the prototype's equivalent operates in backing-store/canvas space. If pixel-phase alignment (§B.5/Phase 5) is implemented, it must correctly convert through the real project's actual transform chain (pan/zoom/rotation/flip via `getPos`-equivalent logic) — not the prototype's simpler `wrapToCanvas`, which has no rotation/flip concept at all.
- **Pressure preview not matching painted pressure:** if the new pressure-responsive radius reads pressure via a different code path than `_getPressure()` (the real engine's driver-glitch-hardened reader), the preview could show a different pressure value than what the actual stroke would use, particularly on hardware known to report glitchy pressure — `_getPressure()` should be reused or its handling replicated exactly, not reimplemented.
- **Preview size not matching actual dab radius:** the bridged radius calculation (§B.3, "Size animation" row) must be checked against the real dab rasterizer's actual radius formula (hardness/roundness/jitter all affect true dab shape) or the preview will visibly mismatch the painted result, particularly for non-round or jittered tips.
- **Custom-tip mismatch:** the new filled-dab hover mode must not silently bypass or conflict with the existing `drawShape()` custom-tip rendering when `cursorStyle==='brush-shape'` — the classification in §B.3 keeps custom-tip rendering as **KEEP REAL PROJECT**; Phase 6 exists specifically to validate this interaction.
- **Brush/eraser state mismatch:** eraser must reliably resolve to the checkerboard fill (or the ring-only fallback, per the open decision) even when `tool` changes mid-hover; the existing `paintTool()`/eraser-forces-ring logic (line 87-88) is the reference to extend, not replace.
- **Zoom/pan/rotation/flip mismatch:** the overlay currently has no rotation/flip awareness for the ring itself (only `drawShape()`'s tip silhouette rotates/flips). If pixel-phase alignment is implemented, failing to account for canvas rotation/flip would misplace the preview relative to where the dab would actually land.
- **High-DPI or backing-scale mismatch:** `prepare(cssSize)` already handles `devicePixelRatio` scaling for the overlay canvas; any new draw function must continue to call `prepare()` (or equivalent) rather than assuming a 1:1 pixel ratio, or the preview will appear soft/misaligned on high-DPI displays.
- **Stale preview after pointer leave or tool switch:** `pointerleave`/`tool-changed` already trigger hide/force-redraw; new state (`lastPressure`, drawing-mode) must be reset or re-evaluated on these same events or a stale pressure/ring state could persist visually after the pointer leaves or the tool changes.
- **Preview interfering with pointer events:** `#brush-cursor-overlay` already has `pointer-events:none`; this must be preserved for any new draw modes (trivially true, since drawing calls don't touch this CSS property, but worth confirming no new DOM element is introduced without it — see "Duplicate cursor/preview renderers" above).
- **Excessive redraw cost:** the existing `signature()` memoization must be extended to include the new inputs (`pressure`, drawing-state) so the RAF loop continues to skip redraws when nothing relevant has changed, rather than redrawing every frame regardless of state (which the current implementation already deliberately avoids).
- **Accidental changes to actual painted dabs:** none of the above functions (`brush-size-cursor.js`) are called from anywhere in `brush-engine.js`'s stroke path (confirmed — `brush-size-cursor.js` only reads `toolSizes`/`tool`/`zoom`/`window.brushTipCanvas` etc., it does not call into `_stampDab`/`_drawDabNow`); this is a structural guarantee, not just a testing concern, but Phase 7/8 manual validation should still confirm no regression in actual painted output as a final check.

## B.9. Manual validation checklist (Animated Dab)

- Mouse hover
- Pen hover
- Light and heavy pressure (pen only — **requires tablet hardware**)
- Brush and eraser
- Very small and very large brush size
- Zoom: 5%, 25%, 100%, 500%, and a high zoom level (e.g. 2000%+)
- Pan, rotation, and flip (canvas-level transforms)
- Custom tips (`brush-shape` cursor mode, with a loaded custom tip image)
- Hard and soft brushes (visual reference only — the preview does not currently model hardness; confirm this remains an acknowledged gap, not an accidental regression)
- Antialiasing modes (AA on/off, i.e. round vs. pixelated/pencil mode)
- Size drag (S-key hold/drag) — confirm the pre-existing `brush-resize-preview-toggle` handshake with `#brush-size-preview` still correctly suppresses `#brush-cursor-overlay` during a size drag
- Tool switching (brush ↔ eraser ↔ other tools) mid-hover
- Pointer leave/enter
- Drawing start/end (confirm hover-dab ↔ drawing-ring transition is clean in both directions)
- Rapid consecutive strokes
- No cursor/preview left behind (e.g., after pointer leaves the window entirely, after `blur`, after a dropped `pointercancel`)
- Preview matches the actual painted dab closely (size, and — where pressure is enabled — general shape) — **requires visual comparison, cannot be verified from source alone**

**Items that specifically require tablet/pen hardware to validate** (cannot be exercised with mouse-only testing): pressure-responsive radius (light/heavy pressure), pen-hover lightest-touch dab behavior, and any pressure-vs-painted-output comparison.

## B.10. Success criteria (Deliverable B only)

Deliverable B is complete only when:

- The animated dab preview (hover: filled, pressure-responsive dab; drawing: thin ring) matches the approved prototype behavior, bridged against the real project's own dab-radius/coordinate conventions rather than literally ported.
- The feature remains independently switchable (its own flag) from Deliverable A's stabilization flag throughout testing.
- No duplicate preview overlay or duplicate RAF loop exists — the feature lives entirely inside the existing `#brush-cursor-overlay`/`brush-size-cursor.js` update loop.
- Custom-tip rendering (`brush-shape` mode), eraser behavior, and the S-key resize-preview handshake all continue to work exactly as before, or with an explicitly approved, documented change.
- Pressure shown in the preview matches pressure used by the actual painted stroke (same `_getPressure()`-equivalent source).
- Preview dab size closely matches actual painted dab size across brush sizes and zoom levels.
- The pixelation/subpixel-alignment question (§B.3) has been explicitly resolved and signed off on, not left ambiguous.
- Obsolete (pre-migration) preview code is removed only after one validated release cycle with the flag on by default (Phase 8).

---

## Combined Success Criteria (Both Deliverables)

Full migration — both deliverables — is complete only when:

- Deliverable A's success criteria (§A.10) are all met.
- Deliverable B's success criteria (§B.10) are all met.
- The two deliverables remain (and were, throughout development, until their respective Phase 7/8) independently switchable via separate flags, and were never combined into a single commit or phase.
- Actual painted brush output is unchanged, except for the intentional stabilization behavior change (Deliverable A) — Deliverable B introduces no change to painted output at all, by construction (§B.4).
- Pressure, presets, custom tips, eraser, layers, frames, undo, and smart-raster behavior all remain correct across both deliverables.
- No duplicate preview/cursor overlay and no duplicate RAF loop remains anywhere in the brush/cursor code, for either deliverable.
- Obsolete code from both deliverables is removed only after each has independently completed one validated release cycle on by default.

---

## Appendix: Explicit Statements of What Could Not Be Determined From the Documents Alone

Per the task's instruction not to speculate beyond what the two source documents support, the following are flagged as requiring additional code inspection or runtime testing rather than being guessed at in this plan:

- **Whether any code outside `brush-engine.js` calls tool-switch cancellation hooks for the general case** (Risk 10) — confirming this requires inspecting `setTool` and its callers, which are outside the scope of `BRUSH_PIPELINE_AUDIT.md`.
- **The prototype's exact input/output coordinate units and cadence, from the prototype's own perspective on the seam** — `BRUSH_PIPELINE_AUDIT.md` derives its seam contract entirely from the real project's side, by its own explicit statement; this plan's seam recommendation (§5) is consistent with both documents as written, but a matching read of the prototype's `feedPoint()`/`pushSmoothBuf()` inputs against the real project's conditioned-sample shape, side by side, was performed here only at the level of documented behavior, not literal source code — any subtle unit mismatch beyond what both documents describe should be re-verified against actual source before Phase 1 implementation begins.
- **Whether `tiltX`/`tiltY`/`twist` feed anything in real-project files other than `brush-engine.js`** (e.g., `brush-presets.js` tip-rotation-by-tilt features) — not traced in the supplied audit; flagged rather than assumed absent.
- **The undo VRAM budget question and Canvas2D-feature-parity product decision raised in the prototype's own audit (§18.3, §19.1)** — these are prototype-side pre-integration cleanup items that are moot for *this* migration specifically (since rendering/undo are out of scope here), but are noted in case a future, separate initiative revisits rendering-layer migration.

**Deliverable B additions:**

- **Whether eraser should show a filled checkerboard dab or remain ring-only** (§B.3, "Eraser behavior" row) — the real project's current ring-only-for-eraser rule is a deliberate existing decision (explicit code comment in `brush-size-cursor.js`); whether the new filled-hover mode should extend to eraser with a checkerboard fill (prototype behavior) or preserve ring-only-for-eraser is a product decision, not something the code alone resolves. Flagged for decision before Animated Dab Phase 6.
- **Whether/how pixel-phase alignment should be ported at all** (§B.3, "Pixel-phase alignment" row) — the real project's coordinate/transform stack (pan/zoom/rotation/flip via `getPos`-equivalent logic) is materially more complex than the prototype's `wrapToCanvas`, which has no rotation/flip concept. Whether the visual benefit of phase-snapping is worth the added complexity of computing it correctly against the real transform stack needs actual on-screen comparison, not a code-only decision. Flagged for Animated Dab Phase 5.
- **Whether the overlay's pixelated/bilinear switch should reuse the main canvas's exact threshold (`useNN = zoom >= 1.5`, single threshold) or adopt the prototype's separate hysteresis model (1.2 in / 1.0 out)** — using two different rules for the main canvas vs. the cursor overlay would itself be a new inconsistency; this needs a explicit decision, not a default port of the prototype's constants.
- **Whether hardness/antialiasing-mode should ever be visually represented in the cursor preview** — neither the prototype nor the real project's current cursor represents this; the task's audit checklist asks about it, and the honest answer from the code is that this is an unaddressed gap in both, not a migration target unless separately scoped.

This document is intended to be sufficiently detailed that implementation can begin later without revisiting the architectural decisions made above. Any point in this plan that depends on information not present in `ARCHITECTURE.md`, `BRUSH_PIPELINE_AUDIT.md`, or the direct source reads performed for Deliverable B has been explicitly marked as requiring further inspection rather than assumed.
