# PQ_FIBS Object Ledger

**Date:** 23 December 2025  
**Task:** 9 — Rendering & Objects (Polyline/Line/Label Budget + Manual GC)  
**File:** `PQ_FIBS.pine`  
**Branch:** `54-pq_fibs-32-audit---rendering-objects`

---

## 1. Indicator Object Limits (L8)

```pine
indicator(... max_lines_count=500, max_labels_count=500, max_boxes_count=500, max_polylines_count=100 ...)
```

| Object Type | Max Count | Notes |
|-------------|-----------|-------|
| `line`      | 500       | Shared across ZigZag, FibLevels, PositionVisual, UI buffers |
| `label`     | 500       | Shared across FibLevels, PositionVisual, UI buffers |
| `box`       | 500       | Used by PositionVisual (entry zone) |
| `polyline`  | 100       | ZigZagVisualBuffer only (POLYLINE mode) |
| `table`     | N/A       | No limit impact (5 debug tables, limit-safe) |

---

## 2. Object Creation Sites

### 2.1 ZigZag Engine — LINES Mode (L1980, L2017)

| Anchor | Function | Object | Gating Condition | Lifetime |
|--------|----------|--------|------------------|----------|
| L1980 | `ZigZagEngine.processPivot()` | `line.new` | `not usePolylineMode` (first segment) | Until deleted at L2012 |
| L2017 | `ZigZagEngine.processPivot()` | `line.new` | `not usePolylineMode` (new segment) | Until deleted at L2012 or mode switch |

**GC Pattern:** Explicit delete at L2012 when direction changes or mode switches.

---

### 2.2 ZigZagVisualBuffer — POLYLINE Mode (L2035-2140)

| Anchor | Method | Object | Gating Condition | Lifetime |
|--------|--------|--------|------------------|----------|
| L2085 | `renderActive()` | `polyline.new` | `array.size(points) >= 2` | Until re-rendered or GC'd |
| L2126 | `updateLive()` | `line.new` | `canShow and na(this.liveLine)` | Until deleted at L2069/L2122 |

**GC Pattern:** 
- Active polyline: Delete-before-create at L2084
- Sealed polylines: `gcPolys()` enforces `maxPolys` cap (default 50) via shift/delete at L2073-2077
- Live line: Explicit delete at L2069 (clearAll) and L2122 (hide)

**Buffer Configuration (L2035-2040):**
```pine
type ZigZagVisualBuffer
    int maxPolys = 50       // INPUT_ZZ_POLY_MAX_KEEP (1-100)
    int maxPoints = 9500    // INPUT_ZZ_POLY_MAX_POINTS (100-10000)
```

---

### 2.3 PositionVisual UDT (L2290-2360)

| Anchor | Method | Object | Gating Condition | Lifetime |
|--------|--------|--------|------------------|----------|
| L2320 | `create()` | `box.new` | Called from L5223/L5226 | Until `deleteAll()` |
| L2321 | `create()` | `line.new` (SL) | Always | Until `deleteAll()` |
| L2323 | `create()` | `line.new` (Smart SL) | `not na(smartSlPrice)` | Until `deleteAll()` |
| L2324 | `create()` | `line.new` (TP) | Always | Until `deleteAll()` |
| L2325 | `create()` | `label.new` (Risk) | Always | Until `deleteAll()` |
| L2346 | `update()` | `line.new` (Smart SL) | Late-bind if missing | Until `deleteAll()` |

**GC Pattern:** `deleteAll()` method (L2298-2312) deletes all 5 objects with na-guard checks.

**Trigger Sites:**
- L5222: `GLOBAL_posVisual.deleteAll()` — Called on `needsRecreate`
- L5223: `GLOBAL_posVisual.create(...)` — Called after deleteAll
- L5226: `GLOBAL_posVisual.create(...)` — First-time creation (no pivot change yet)
- L5229: `GLOBAL_posVisual.update(...)` — Dynamic learning adjustments

---

### 2.4 FibLevel Rendering (L3465-3495)

| Anchor | Method | Object | Gating Condition | Lifetime |
|--------|--------|--------|------------------|----------|
| L3479 | `FibLevel.draw()` | `line.new` | `forceRedraw and price > 0` | Until forceRedraw |
| L3489 | `FibLevel.draw()` | `label.new` | `forceRedraw and INPUT_LEVELS_LABEL != UI_OPT_none and price > 0` | Until forceRedraw |

**GC Pattern:** Delete-before-create at L3473 (line) and L3476 (label).

**Instance Count:** 22 FibLevel instances in `BUF_fibLevels` array (L2430-2451), each with optional line/label = up to 44 objects.

---

### 2.5 UI Buffers — Pivot Lines/Labels (L3211-3212, L3440-3462)

| Anchor | Function | Object | Gating Condition | Lifetime |
|--------|----------|--------|------------------|----------|
| L3450 | `f_drawLineTZ()` | `line.new` | `_x1 - bar_index < 500` | Until bar change GC |
| L3454 | `f_drawLabelTZ()` | `label.new` | `_x - bar_index < 500` | Until bar change GC |
| L3458 | `f_drawLinePVT()` | `line.new` | `_y1 > 0` | Until bar change GC |
| L3462 | `f_drawLabelPVT()` | `label.new` | `_y > 0` | Until bar change GC |

**Buffer Declarations (L3211-3212):**
```pine
var array<line> UI_linesBuffer = array.new<line>()
var array<label> UI_labelsBuffer = array.new<label>()
```

**GC Pattern (L3440-3446):**
```pine
if ta.change(time) != 0 and array.size(UI_linesBuffer) > 0
    for i = 1 to array.size(UI_linesBuffer) by 1
        line.delete(array.shift(UI_linesBuffer))

if ta.change(time) != 0 and array.size(UI_labelsBuffer) > 0
    for i = 1 to array.size(UI_labelsBuffer) by 1
        label.delete(array.shift(UI_labelsBuffer))
```

**Trigger:** On every bar change (`ta.change(time) != 0`), all buffered lines/labels are deleted.

---

### 2.6 Debug Tables (L5380-5548)

| Anchor | Table Name | Position | Gating Condition |
|--------|------------|----------|------------------|
| L5380 | `entropyDebugTable` | bottom_right | `perf_shouldShowDebugOverlay(INPUT_ENTROPY_DEBUG)` |
| L5431 | `hurstDebugTable` | middle_right | `perf_shouldShowDebugOverlay(INPUT_HURST_DEBUG)` |
| L5480 | `extDebugTable` | bottom_left | `perf_shouldShowDebugOverlay(INPUT_EXT_DEBUG)` |
| L5514 | `flowDebugTable` | top_left | `perf_shouldShowDebugOverlay(INPUT_DEBUG_FLOW)` |
| L5548 | `perfDashboard` | top_right | `perf_shouldShowDashboard()` |

**GC Pattern:** None needed — tables are `var` (persistent) and use `table.cell()` updates only.  
**Limit Impact:** Zero — tables don't count against line/label/box/polyline limits.

---

## 3. Object Budget Summary

### 3.1 Worst-Case Static Allocation

| Module | Lines | Labels | Boxes | Polylines |
|--------|-------|--------|-------|-----------|
| ZigZag (LINES mode) | 1 | 0 | 0 | 0 |
| ZigZag (POLYLINE mode) | 1 (live) | 0 | 0 | 51 (50 sealed + 1 active) |
| PositionVisual | 4 (SL, SmartSL, TP, live) | 1 | 1 | 0 |
| FibLevels (22 levels) | 22 | 22 | 0 | 0 |
| UI Buffers | Variable | Variable | 0 | 0 |
| **Subtotal (POLYLINE mode)** | **27** | **23** | **1** | **51** |

### 3.2 Dynamic Headroom

| Object Type | Max | Reserved | Headroom | Headroom % |
|-------------|-----|----------|----------|------------|
| Lines       | 500 | 27       | 473      | 94.6%      |
| Labels      | 500 | 23       | 477      | 95.4%      |
| Boxes       | 500 | 1        | 499      | 99.8%      |
| Polylines   | 100 | 51       | 49       | 49.0%      |

**Risk Assessment:**
- Lines/Labels/Boxes: ✅ SAFE — >90% headroom
- Polylines: ⚠️ MODERATE — 49% headroom, but capped by `maxPolys` (configurable 1-100)

---

## 4. Object Lifecycle Patterns

### 4.1 Pattern: Delete-Before-Create
Used for objects that must be replaced atomically.

| Module | Delete Site | Create Site | Pattern |
|--------|-------------|-------------|---------|
| ZigZagVisualBuffer.renderActive | L2084 | L2085 | `polyline.delete(this.active)` then `polyline.new(...)` |
| FibLevel.draw | L3473, L3476 | L3479, L3489 | `line.delete(this.ln)` then `line.new(...)` |
| PositionVisual | L5222 | L5223 | `deleteAll()` then `create(...)` |

### 4.2 Pattern: Shift-Delete (FIFO Queue)
Used for capped buffers with oldest-first eviction.

| Module | Buffer | Cap Enforcement | Delete Site |
|--------|--------|-----------------|-------------|
| ZigZagVisualBuffer.gcPolys | `this.sealed` | `maxPolys` (50) | L2077 |
| UI_linesBuffer | `UI_linesBuffer` | Per-bar clear | L3442 |
| UI_labelsBuffer | `UI_labelsBuffer` | Per-bar clear | L3446 |

### 4.3 Pattern: Conditional Guard
Used for optional objects that may not exist.

| Module | Guard Pattern | Site |
|--------|---------------|------|
| PositionVisual.deleteAll | `if not na(this.boxEntry)` | L2299 |
| ZigZagVisualBuffer.clearAll | `if not na(this.active)` | L2059 |
| ZigZagEngine.processPivot | `if not na(this.ln)` | L2011 |

---

## 5. Rendering Isolation Verification

### 5.1 Rendering → Signal Boundary

**Requirement:** Rendering code must NOT mutate signal state (GLOBAL_zigzag.changed, GLOBAL_activeTrade, etc.).

| Render Module | Writes To | Signal State? | Verdict |
|---------------|-----------|---------------|---------|
| ZigZagVisualBuffer.* | polyline/line objects, internal arrays | ❌ No | ✅ PASS |
| PositionVisual.* | box/line/label objects, internal refs | ❌ No | ✅ PASS |
| FibLevel.draw | line/label objects, `GLOBAL_pivotLevelsLogRetracements` | ⚠️ Logs only | ✅ PASS |
| UI buffer functions | line/label arrays | ❌ No | ✅ PASS |
| Debug tables | table cells only | ❌ No | ✅ PASS |

**FibLevel Note:** `GLOBAL_pivotLevelsLogRetracements.put()` at L3492 is a read-write to a logging map, but this is:
1. After all signal decisions are made
2. Only affects alert JSON construction (downstream, not upstream)
3. Does not influence GLOBAL_zigzag.changed, spawn decisions, or trade state

### 5.2 Signal → Rendering Trigger Chain

```
GLOBAL_zigzag.changed (L2027, L3323, L3358)
    │
    ├─► GLOBAL_zigzagBuffer.addSegment() [L3315, L3350]
    │
    ├─► needsFullRedraw = true [L3800]
    │   └─► FibLevel.draw(forceRedraw=true) [L3802-loop]
    │
    └─► needsRecreate = true [L5220]
        └─► GLOBAL_posVisual.deleteAll() + create() [L5222-5223]
```

**Key Invariant:** `GLOBAL_zigzag.changed` is a one-shot flag set during pivot processing (L2027, L3323, L3358) and reset at bar start via `resetChanged()` (L3289). Rendering reads this flag but never writes it.

---

## 6. Open Questions / Edge Cases

### 6.1 UI Buffer Unbounded Growth Risk
The `UI_linesBuffer` and `UI_labelsBuffer` arrays have no explicit cap. They are cleared per-bar, but within a single bar:

```pine
f_drawLineTZ(...)  // L3450 - pushes to UI_linesBuffer
f_drawLinePVT(...) // L3458 - pushes to UI_linesBuffer
```

**Worst Case:** If called many times per bar (e.g., multiple HTF pivots), buffer could exceed 500 lines before bar-end GC.

**Mitigation:** The `_x1 - bar_index < 500` guard (L3449) limits creation to recent bars, preventing runaway growth.

**Verdict:** ✅ LOW RISK — Guard prevents historical spam; single-bar burst unlikely to hit 500.

### 6.2 Polyline Cap vs User Setting
`INPUT_ZZ_POLY_MAX_KEEP` allows user to set `maxPolys` up to 100, matching the indicator limit. If set to 100:

- `maxPolys = 100` (cap)
- `sealed` array could hold 100 polylines
- Plus 1 active polyline = 101 total

**Potential Issue:** Off-by-one exceeds `max_polylines_count=100`.

**Mitigation Check (L2073-2077):**
```pine
method gcPolys(ZigZagVisualBuffer this) =>
    if not na(this.sealed)
        while array.size(this.sealed) > this.maxPolys
            polyline doomed = array.shift(this.sealed)
            polyline.delete(doomed)
```

The GC runs in `flushIfFull()` which is called after pushing to sealed but before creating new active. This means:
- Push to sealed (now 100)
- gcPolys runs (deletes to 100)
- But active is then re-created → 101 total

**Verdict:** ⚠️ EDGE CASE — At `maxPolys=100`, could hit 101 polylines transiently. Recommend default of 50 (currently set) or cap input at 99.

---

## 7. Recommendations

1. **Cap INPUT_ZZ_POLY_MAX_KEEP at 99** to prevent off-by-one overflow at max setting.
2. **Add UI_linesBuffer cap check** (optional) if ever extending to multi-HTF rendering.
3. **Document table immunity** in performance guide — tables are limit-safe.

---

## Appendix A: Object Type Reference

| Pine Object | Constructor | Destructor | Limit Parameter |
|-------------|-------------|------------|-----------------|
| `line` | `line.new()` | `line.delete()` | `max_lines_count` |
| `label` | `label.new()` | `label.delete()` | `max_labels_count` |
| `box` | `box.new()` | `box.delete()` | `max_boxes_count` |
| `polyline` | `polyline.new()` | `polyline.delete()` | `max_polylines_count` |
| `table` | `table.new()` | `table.delete()` | N/A (no limit) |
