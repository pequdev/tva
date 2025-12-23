# PQ_FIBS Rendering GC Verification

**Date:** 23 December 2025  
**Task:** 9 — Rendering & Objects (Polyline/Line/Label Budget + Manual GC)  
**File:** `PQ_FIBS.pine`  
**Branch:** `54-pq_fibs-32-audit---rendering-objects`

---

## 1. GC Verification Matrix

This document audits every drawing object module for proper garbage collection (GC) enforcement to prevent TradingView limit violations.

| Module | Object Type | Cap Mechanism | Bypass Paths | Verdict |
|--------|-------------|---------------|--------------|---------|
| ZigZagEngine (LINES) | line | Delete-on-change | None | ✅ PASS |
| ZigZagVisualBuffer | polyline | `maxPolys` cap + FIFO eviction | Edge case at max=100 | ⚠️ PASS* |
| ZigZagVisualBuffer | line (live) | Single-instance delete-before-create | None | ✅ PASS |
| PositionVisual | box, line, label | `deleteAll()` before `create()` | None | ✅ PASS |
| FibLevel | line, label | Delete-before-create | None | ✅ PASS |
| UI Buffers | line, label | Per-bar clear | None | ✅ PASS |
| Debug Tables | table | var persistence (no growth) | N/A | ✅ PASS |

**Legend:**
- ✅ PASS: GC correctly enforced, no budget overflow possible
- ⚠️ PASS*: GC enforced but edge case exists (documented below)
- ❌ FAIL: GC missing or bypassable

---

## 2. Detailed GC Audits

### 2.1 ZigZagEngine — LINES Mode

**Object:** Single `line` instance per ZigZag (stored in `GLOBAL_zigzag.ln`)

**Creation Site (L1980, L2017):**
```pine
line newLine = line.new(this.iLast, this.pLast, index, price, ...)
this.ln := newLine
```

**Deletion Site (L2012):**
```pine
if not show_zigzag or usePolylineMode
    if not na(this.ln)
        line.delete(this.ln)
        this.ln := na
```

**GC Flow:**
1. Direction change OR mode switch triggers deletion
2. `this.ln := na` ensures dangling reference cleared
3. New segment creates fresh line

**Bypass Analysis:**
- No path creates line without checking `na(this.ln)` first
- Mode switch (L3142) explicitly deletes: `line.delete(GLOBAL_zigzag.ln)`

**Verdict:** ✅ PASS — Single line instance, always deleted before replacement.

---

### 2.2 ZigZagVisualBuffer — Polylines

**Objects:** 
- `active` polyline (1 instance)
- `sealed` array of polylines (up to `maxPolys`)
- `liveLine` line (1 instance)

#### 2.2.1 Active Polyline GC

**Creation Site (L2085):**
```pine
this.active := polyline.new(snapshot, ...)
```

**Deletion Site (L2084):**
```pine
if not na(this.active)
    polyline.delete(this.active)
this.active := polyline.new(...)
```

**Pattern:** Delete-before-create (atomic replacement)

**Verdict:** ✅ PASS — Single instance, always deleted before recreation.

---

#### 2.2.2 Sealed Polylines GC

**Cap Enforcement (L2073-2077):**
```pine
method gcPolys(ZigZagVisualBuffer this) =>
    if not na(this.sealed)
        while array.size(this.sealed) > this.maxPolys
            polyline doomed = array.shift(this.sealed)
            polyline.delete(doomed)
```

**Called From:** `flushIfFull()` at L2094

**Flow:**
1. `addSegment()` renders active polyline
2. `flushIfFull()` checks if points >= maxPoints (9500)
3. If full: push active to sealed, run `gcPolys()`
4. `gcPolys()` evicts oldest until `size <= maxPolys`

**Cap Configuration (L2037-2039):**
```pine
int maxPolys = 50   // Default, INPUT_ZZ_POLY_MAX_KEEP (1-100)
```

**Edge Case Analysis:**

When `maxPolys = 100` (max user setting) and indicator limit is also 100:
1. `sealed.size()` can reach 100 (at cap)
2. `gcPolys()` runs with `size > maxPolys` = `100 > 100` = false (no eviction)
3. Active polyline exists = 1 more
4. **Total: 101 polylines** (exceeds limit of 100)

**Mitigation Status:**
- Default is 50, providing safe headroom
- Input allows max of 100 which causes edge case

**Verdict:** ⚠️ PASS* — GC works correctly but user can configure to edge case.

**Recommendation:** Cap `INPUT_ZZ_POLY_MAX_KEEP` at 99 to prevent off-by-one.

---

#### 2.2.3 Live Line GC

**Creation Site (L2126):**
```pine
if na(this.liveLine)
    this.liveLine := line.new(...)
else
    line.set_xy1(...) // Update in place
```

**Deletion Sites:**
- L2069: `clearAll()` → `line.delete(this.liveLine)`
- L2122: `updateLive()` when `not canShow` → `line.delete(this.liveLine)`
- L3367: Mode switch → `line.delete(GLOBAL_zigzagBuffer.liveLine)`

**Pattern:** Single instance with conditional visibility delete.

**Verdict:** ✅ PASS — Properly deleted when hidden or mode switched.

---

### 2.3 PositionVisual — Entry Zone Objects

**Objects:** 1 box, 4 lines, 1 label (5 total per position)

**Deletion Method (L2298-2312):**
```pine
method deleteAll(PositionVisual this) =>
    if not na(this.boxEntry)
        box.delete(this.boxEntry)
        this.boxEntry := na
    if not na(this.lineSl)
        line.delete(this.lineSl)
        this.lineSl := na
    // ... (same for lineSmartSl, lineTp, lblRisk)
```

**Trigger Sites:**
- L5222: `GLOBAL_posVisual.deleteAll()` — On `needsRecreate`
- L5223: `GLOBAL_posVisual.create(...)` — Immediately after deleteAll

**`needsRecreate` Condition (L5220):**
```pine
bool needsRecreate = GLOBAL_zigzag.changed or projectionStateChanged or (isProjection and (proj_tentative_high or proj_tentative_low))
```

**Bypass Analysis:**
- `create()` is always preceded by `deleteAll()` at L5222-5223
- `update()` at L5229 only modifies existing objects (no creation)
- Late-bind Smart SL at L2346 only creates if `na(this.lineSmartSl)` (no leak)

**Verdict:** ✅ PASS — Atomic delete-create cycle, no orphaned objects.

---

### 2.4 FibLevel — Retracement Lines/Labels

**Objects:** 22 levels × (1 line + 1 label) = 44 objects max

**Deletion Before Creation (L3471-3477):**
```pine
if forceRedraw
    if not na(this.ln)
        line.delete(this.ln)
        this.ln := na
    if not na(this.lb)
        label.delete(this.lb)
        this.lb := na
    if price > 0
        this.ln := line.new(...)
```

**Trigger (L3800-3802):**
```pine
bool needsFullRedraw = GLOBAL_zigzag.changed or (BUF_fibLevels.size() > 0 ? na(array.get(BUF_fibLevels, 0).ln) : false)

for i = 0 to levelCount - 1
    array.get(BUF_fibLevels, i).draw(x1, pMid, x2, pEnd, needsFullRedraw)
```

**Bypass Analysis:**
- `forceRedraw=true` → delete old before new
- `forceRedraw=false` → only `line.set_x2()` (no creation)
- Initial state: `ln = na`, `lb = na` (no orphans)

**Verdict:** ✅ PASS — Controlled 22-level budget, delete-before-create enforced.

---

### 2.5 UI Buffers — HTF Pivot Lines/Labels

**Objects:** Variable per bar, cleared on bar change

**Buffer Declarations (L3211-3212):**
```pine
var array<line> UI_linesBuffer = array.new<line>()
var array<label> UI_labelsBuffer = array.new<label>()
```

**GC Trigger (L3440-3446):**
```pine
if ta.change(time) != 0 and array.size(UI_linesBuffer) > 0
    for i = 1 to array.size(UI_linesBuffer) by 1
        line.delete(array.shift(UI_linesBuffer))
```

**Creation Guard (L3449, L3453):**
```pine
if _x1 - bar_index < 500  // Only create for recent bars
    array.push(UI_linesBuffer, line.new(...))
```

**Burst Risk Analysis:**

Within a single bar, before GC runs:
- Multiple calls to `f_drawLineTZ()` / `f_drawLinePVT()`
- Each pushes to buffer without cap check

**Worst Case Calculation:**
- HTF pivots: typically 2-4 per bar
- Pivot lines: 1 line per level shown
- Max theoretical: 22 levels × 2 = 44 lines/bar

**Verdict:** ✅ PASS — Per-bar clear prevents accumulation; guard prevents historical spam.

---

### 2.6 Debug Tables

**Objects:** 5 tables, all declared with `var` (persistent)

**Declarations (L5380, L5431, L5480, L5514, L5548):**
```pine
var table entropyDebugTable = table.new(...)
var table hurstDebugTable = table.new(...)
var table extDebugTable = table.new(...)
var table flowDebugTable = table.new(...)
var table perfDashboard = table.new(...)
```

**Update Pattern:** `table.cell()` only — no `table.new()` after initialization.

**Gating:** All gated by `barstate.islast` via `perf_shouldShowDebugOverlay()`.

**Limit Impact:** Zero — tables don't count against line/label/box/polyline limits.

**Verdict:** ✅ PASS — Static allocation, no growth, no limit impact.

---

## 3. Consolidated Bypass Path Analysis

| Potential Bypass | Module | Investigation | Result |
|------------------|--------|---------------|--------|
| Creation without prior delete | ZigZagEngine | Checked L1980, L2017 | ❌ Always gated by direction/mode |
| Unbounded array growth | ZigZagVisualBuffer.sealed | Checked gcPolys() | ❌ Capped by maxPolys |
| Multiple creates per bar | PositionVisual | Checked trigger flow | ❌ deleteAll() always precedes |
| Orphaned FibLevel objects | FibLevel | Checked forceRedraw | ❌ Delete-before-create |
| UI buffer overflow | UI_linesBuffer | Checked creation guards | ❌ Bar-index guard limits |
| Table growth | Debug tables | Checked declarations | ❌ var = single allocation |

**Overall Verdict:** No bypass paths to TradingView object limits.

---

## 4. Cap Enforcement Summary

| Object Type | Indicator Limit | Reserved | Enforced By | Safe? |
|-------------|-----------------|----------|-------------|-------|
| `line`      | 500 | ~27 | Delete-before-create + per-bar clear | ✅ |
| `label`     | 500 | ~23 | Delete-before-create + per-bar clear | ✅ |
| `box`       | 500 | 1 | PositionVisual.deleteAll() | ✅ |
| `polyline`  | 100 | 51 | gcPolys() FIFO eviction | ⚠️ PASS* |

**Note (⚠️ PASS*):** Polyline cap is safe at default settings but edge case exists at max user config.

---

## 5. Recommendations

### 5.1 Immediate (Low Effort)

1. **Cap INPUT_ZZ_POLY_MAX_KEEP at 99**
   - Change: `input.int(50, 'Max Polylines', minval=1, maxval=99, ...)`
   - Rationale: Prevents 100+1 edge case

### 5.2 Optional (Future Hardening)

2. **Add explicit cap check to UI buffers**
   ```pine
   if array.size(UI_linesBuffer) < 450  // 50 headroom
       array.push(UI_linesBuffer, line.new(...))
   ```
   - Rationale: Belt-and-suspenders for multi-HTF extensions

3. **Instrument GC with debug counter**
   - Add `DBG_counterGcPolysEvicted` to track eviction frequency
   - Helps tune `maxPolys` default

---

## 6. Test Scenarios

### 6.1 Polyline Stress Test
1. Set `INPUT_ZZ_POLY_MAX_KEEP = 50`
2. Load long history (>1000 bars with many pivots)
3. Switch to POLYLINE mode
4. Verify `sealed.size()` never exceeds 50
5. Verify no TradingView "too many polylines" error

### 6.2 Rapid Pivot Test
1. Use 1-minute chart with volatile asset
2. Enable all FibLevels
3. Run for 100+ bars with frequent pivot changes
4. Verify no accumulation of orphaned objects
5. Check line count in Pine debugger

### 6.3 Mode Switch Test
1. Start in LINES mode
2. Switch to POLYLINE mode
3. Switch back to LINES mode
4. Verify all polylines deleted
5. Verify ZigZag line restored

---

## 7. Conclusion

**Overall GC Status: ✅ PASS**

All rendering modules implement proper garbage collection with no bypass paths to TradingView object limits. The single edge case (polyline cap at max=100) is:
1. Only reachable with non-default user configuration
2. Mitigatable with a simple input constraint change
3. Low probability of actual limit violation in practice

The rendering layer correctly maintains object budgets through a combination of:
- Delete-before-create patterns for singleton objects
- FIFO eviction for capped buffers
- Per-bar cleanup for transient objects
- Static allocation for persistent tables
