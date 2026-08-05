# MIGRATION_PLAN.md
## Integrating the Stabilization Prototype (`stabilizationtest.html`) into the Real Project (`behind-me-is-winner`)

**Status:** Planning document only. No source code was modified or written to produce this plan. No implementation code is included below.

**Sources:**
- `ARCHITECTURE.md` — audit of the prototype (`stabilizationtest.html`, commit `0d0327d`, 2026-08-04)
- `BRUSH_PIPELINE_AUDIT.md` — audit of the real project's brush engine (`src/brush/brush-engine.js` + supporting files)

**Scope constraint carried over from both source audits:** the *only* thing this migration moves from the prototype into the real project is the **stabilization/pressure-timing behavior**. The prototype's renderer (WebGPU), undo model, and "app" (no layers/frames/presets) are not being migrated — they have no real-project equivalent to replace, and the real project's audit explicitly identifies its own rendering, undo, and layer/frame systems as things that must not be touched by this work.

---

## 1–2. Subsystem Comparison Table

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
| Debug systems | Substantial: always-on per-stroke console logging (`[stabilizer cadence]`, `[stabilizer finish]`), a confirmed-unused embedded WASM module, an unreachable `#fatal` overlay, a global `window.setStabDebugConstantPressure` debug hook, a production-facing leash visualization overlay | Not comprehensively audited in `BRUSH_PIPELINE_AUDIT.md` beyond passing references to "telemetry/profiler teardown calls" in `_finalizePointerEndStroke` | **Prototype debug surface should not be migrated; real project's is unknown/out of scope** | The prototype audit itself recommends removing or gating nearly all of its debug systems before any integration (§13, §19.3, §20.3 of `ARCHITECTURE.md`). None of this should cross into the real project. The leash *visualization* is a UI feature question, not part of the stabilization algorithm seam, and is deferred (see §9 below). |

---

## 3. Subsystem Classification (KEEP / REPLACE / ADAPT / REMOVE / LEAVE FOR LATER)

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

## 4. Compatibility Risks

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

## 5. First Migration Seam

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

## 6. Migration Order

Each phase is small, independently testable, and reversible via a feature flag. No phase combines large subsystems.

### Phase 0 — Instrumentation-only

- **Goal:** Establish a toggle point without changing any behavior.
- **Files affected:** `brush-engine.js` (new flag declaration only).
- **Functions affected:** none functionally; a new flag (e.g., `window._newStabilizerEnabled`, default `false`) is introduced but not yet read anywhere that changes behavior.
- **Expected behavior:** Identical to pre-migration in every respect.
- **Tests:** Confirm the flag exists and defaults to off; confirm no behavior change anywhere in the app.
- **Rollback strategy:** Delete the flag declaration; trivial, no risk.

### Phase 1 — New stabilizer as an inert, unit-testable pure function

- **Goal:** Implement the new stabilization algorithm as a standalone function (e.g., `_newStabilizePoint(x, y, t)`), callable but not wired into the live pipeline.
- **Files affected:** `brush-engine.js` (new function(s), no call-site changes).
- **Functions affected:** none of the existing call sites; purely additive.
- **Expected behavior:** No change to the running application. The new function can be exercised in isolation (feed known input sequences, inspect output) via a test harness or console.
- **Tests:** Unit-level only — feed synthetic sample sequences (straight line, held-still, sharp corner, fast flick) and verify the output shape and general smoothing behavior against expectations derived from the prototype's documented behavior, adapted for the real project's EMA/leash-based framing rather than a literal port of the moving-average math.
- **Rollback strategy:** Function is unused; delete it, no risk to the running app.

### Phase 2 — Wire the flag into `_stabilizePoint`'s call site only (position, no hold/finish yet)

- **Goal:** Branch to the new implementation for live position filtering when the flag is on, while explicitly leaving hold/finish catch-up as a stub.
- **Files affected:** `brush-engine.js`.
- **Functions affected:** `_stabilizePoint`'s call sites (the two locations noted in the audit as feeding it) branch on the flag.
- **Expected behavior:** With the flag off (default), zero change. With the flag on, live-drawing position filtering uses the new algorithm, but **hold-while-stationary catch-up and release catch-up are not yet implemented** — this phase alone will visibly break "hold" and "finish" feel and must only be tested with those interactions disabled, mocked, or explicitly acknowledged as broken, or shipped together with Phase 3.
- **Tests:** Draw continuous strokes only (no holding still, no careful release timing) at multiple zoom levels and Stabilization slider values, flag on vs. off, comparing general smoothing feel qualitatively.
- **Rollback strategy:** Flip the flag off; old code path is untouched and fully functional.

### Phase 3 — Implement the finalize/accelerate-to-completion contract

- **Goal:** Implement the new stabilizer's equivalent of `_stabilizerFinalize`/`_stabilizerAccelerateToCompletion`/`_stabilizerCancel`, and wire it into the exact same call sites as today: `_pointerEndStroke`, `_endStroke` (conditionally), `finishActiveDrawingBeforeArtworkChange`.
- **Files affected:** `brush-engine.js`.
- **Functions affected:** `_pointerEndStroke`, `_endStroke`, `finishActiveDrawingBeforeArtworkChange`, all behind the same flag.
- **Expected behavior:** With the flag on, releasing the pen (with the new stabilizer active) now correctly converges and commits, including the fast-forward path when a new stroke interrupts an old one still converging. The `_activeStrokeSession` staleness check must be preserved and must correctly reject a stale finalize callback.
- **Tests:** Lift pen immediately after a fast flick (no truncation/overshoot); start a new stroke immediately after releasing the previous one (no stray marks from a stale finalize callback); switch tool/layer/frame mid-stroke (stroke commits to its **origin** layer/frame, not the destination); background the tab mid-stroke (stroke commits correctly on return).
- **Rollback strategy:** Flip the flag off; both position and finish logic revert together to the existing implementation.

### Phase 4 — Hold-while-stationary catch-up, unified with the existing RAF/idle-delay model

- **Goal:** Implement hold-while-stationary catch-up for the new stabilizer, explicitly unified with (not parallel to) the existing `_stabilizerSchedule`/`_stabilizerStep` idle-delay and convergence-epsilon pattern, to avoid the two-competing-RAF-loops risk (Risk 7).
- **Files affected:** `brush-engine.js`.
- **Functions affected:** the new stabilizer's RAF scheduling logic; no changes to `_scheduleRecomposite`'s independent RAF coalescing.
- **Expected behavior:** Holding the pen stationary mid-stroke visibly catches the brush tip up to the pen position, matching (in spirit, not necessarily in exact timing constants) both the prototype's `strokeHoldTick` behavior and the real project's existing `_stabilizerStep` behavior.
- **Tests:** Hold pen stationary mid-stroke at multiple Stabilization amounts and zoom levels; confirm catch-up animates smoothly and pressure does not drift during the hold (pressure must remain pinned per the existing "position only" contract unless Risk 4 has been separately, explicitly resolved).
- **Rollback strategy:** Flip the flag off; falls back fully to the existing `_stabilizerStep` implementation.

### Phase 5 — Zoom compensation and low-zoom interaction validation

- **Goal:** Confirm or adapt the new stabilizer's zoom behavior against the real project's leash-bound zoom scaling and the downstream low-zoom Savitzky–Golay reconstruction (Risk 9), which was not designed against a new upstream sample density/timing profile.
- **Files affected:** `brush-engine.js` (new stabilizer's internal zoom handling only; `_curveAddPoint`'s SG stage is not modified).
- **Functions affected:** the new stabilizer's zoom-dependent internals.
- **Expected behavior:** Stroke quality at low zoom (`zoom < 1`) with the new stabilizer active is comparable to today's behavior — no new jitter, no new over-smoothing artifacts introduced by a sample-density mismatch with the SG stage.
- **Tests:** Draw slow curves, circles, and fast strokes at zoom 25%, 50%, 100%, 400%, comparing flag-on vs. flag-off for smoothness and responsiveness.
- **Rollback strategy:** Flip the flag off.

### Phase 6 — Resolve the 0%-stabilization semantics decision

- **Goal:** Explicitly resolve, as a reviewed decision (not a guess), whether Stabilization slider at 0% should produce true bypass or the current coerced minimum behavior, and implement the new stabilizer's 0% behavior accordingly (Risk 5).
- **Files affected:** `brush-engine.js` (the new stabilizer's amount-mapping function only).
- **Functions affected:** the new stabilizer's equivalent of `_stabilizationAmount()`.
- **Expected behavior:** Documented, deliberate 0% behavior, independent of whatever the historical coercion bug/decision was.
- **Tests:** Draw at Stabilization 0% specifically, across zoom levels, with a decision-owner (whoever owns brush feel) explicitly signing off on the observed behavior as correct.
- **Rollback strategy:** This phase is a decision + small implementation change; can be reverted independently of Phases 1–5 since it only affects the amount-mapping function.

### Phase 7 — Full manual validation pass with flag on

- **Goal:** Comprehensive manual test pass (see §7 below) across real device/pen hardware, all Stabilization amounts (including the resolved 0% behavior), multiple zoom levels, and all interaction patterns.
- **Files affected:** none (testing phase).
- **Expected behavior:** No regressions versus the pre-migration baseline in anything not intentionally changed.
- **Tests:** Full checklist in §7.
- **Rollback strategy:** Flag remains off by default until this phase passes cleanly.

### Phase 8 — Flip default flag on; retain old path as rollback lever

- **Goal:** Make the new stabilizer the default behavior while keeping the old implementation available behind the (now-defaulted-on) flag for at least one release cycle.
- **Files affected:** `brush-engine.js` (default value of the flag only).
- **Expected behavior:** New stabilizer is what users experience by default; flipping the flag off remains a working rollback path.
- **Tests:** Monitor for regressions reported during the release cycle; re-run the Phase 7 checklist opportunistically.
- **Rollback strategy:** Flip the flag back off; old implementation is still present and untouched.

### Phase 9 — Remove old stabilizer code

- **Goal:** Delete the old `_stabilizePoint`/`_stabilizerAdvance`/`_stabilizerStep`/`_stabilizerFinalize` implementation and the flag itself, only after the new stabilizer has been on-by-default for a full release cycle with no regressions reported.
- **Files affected:** `brush-engine.js`.
- **Expected behavior:** Identical to Phase 8's steady state, with dead code removed.
- **Tests:** Full regression pass one more time post-removal, to catch anything that was silently relying on old-path-specific behavior even while the flag was on.
- **Rollback strategy:** None beyond source control revert — this phase is only taken once confidence is high, per the explicit gating condition above.

---

## 7. Manual Validation After Every Phase

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

## 8. Things That MUST NOT Change

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

## 9. Things Intentionally Deferred

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

## 10. Success Criteria

"Migration complete" means all of the following are true:

- The new stabilizer (informed by, but not a literal port of, the prototype's algorithm) is the real project's default stroke-position stabilization implementation, and the old `_stabilizePoint`/RAF-catch-up implementation has been removed (Phase 9 complete).
- Every manual test in §7 passes with no regressions versus the pre-migration baseline, across all Stabilization amounts (including a deliberately-resolved 0% behavior), multiple zoom levels, and both mouse and pen input.
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

## Appendix: Explicit Statements of What Could Not Be Determined From the Documents Alone

Per the task's instruction not to speculate beyond what the two source documents support, the following are flagged as requiring additional code inspection or runtime testing rather than being guessed at in this plan:

- **Whether any code outside `brush-engine.js` calls tool-switch cancellation hooks for the general case** (Risk 10) — confirming this requires inspecting `setTool` and its callers, which are outside the scope of `BRUSH_PIPELINE_AUDIT.md`.
- **The prototype's exact input/output coordinate units and cadence, from the prototype's own perspective on the seam** — `BRUSH_PIPELINE_AUDIT.md` derives its seam contract entirely from the real project's side, by its own explicit statement; this plan's seam recommendation (§5) is consistent with both documents as written, but a matching read of the prototype's `feedPoint()`/`pushSmoothBuf()` inputs against the real project's conditioned-sample shape, side by side, was performed here only at the level of documented behavior, not literal source code — any subtle unit mismatch beyond what both documents describe should be re-verified against actual source before Phase 1 implementation begins.
- **Whether `tiltX`/`tiltY`/`twist` feed anything in real-project files other than `brush-engine.js`** (e.g., `brush-presets.js` tip-rotation-by-tilt features) — not traced in the supplied audit; flagged rather than assumed absent.
- **The undo VRAM budget question and Canvas2D-feature-parity product decision raised in the prototype's own audit (§18.3, §19.1)** — these are prototype-side pre-integration cleanup items that are moot for *this* migration specifically (since rendering/undo are out of scope here), but are noted in case a future, separate initiative revisits rendering-layer migration.

This document is intended to be sufficiently detailed that implementation can begin later without revisiting the architectural decisions made above. Any point in this plan that depends on information not present in `ARCHITECTURE.md` or `BRUSH_PIPELINE_AUDIT.md` has been explicitly marked as requiring further inspection rather than assumed.
