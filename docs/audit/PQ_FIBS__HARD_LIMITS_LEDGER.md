# PQ_FIBS.pine — Hard Limits Ledger

> **Version**: 1.0  
> **Generated**: 2025-12-22  
> **Source**: `PQ_FIBS.pine` (5,591 lines)  
> **Pine Version**: v6

---

## 1. Request Context Budget

### 1.1 Pine v6 Limit
- **Standard**: 40 unique request contexts per script
- **Higher plans**: May allow more, but script is designed for ≤40

### 1.2 Current Request Inventory

| ID | Location | Call | Unique Context | Gating Condition |
|----|----------|------|----------------|------------------|
| R1 | L2476 | `request.security_lower_tf(syminfo.tickerid, INPUT_INTRABAR_TF, high)` | 1 | **Always** (global scope) |
| R2 | L2477 | `request.security_lower_tf(syminfo.tickerid, INPUT_INTRABAR_TF, low)` | 1 | **Always** (global scope) |
| R3 | L1694 (fn def) | `f_reqExtClose()` → `request.security(sym, tf, close, ...)` | 1 per symbol | `INPUT_EXT_ENABLED` + per-symbol loop |
| R4 | L1697 (fn def) | `f_reqExtOHLCV()` → `request.security(sym, tf, [o,h,l,c,v], ...)` | 1 per symbol | `INPUT_EXT_ENABLED` + per-symbol loop |

### 1.3 Context Budget Summary

| Scenario | Contexts Used | Remaining (of 40) |
|----------|---------------|-------------------|
| External OFF (default) | 2 | 38 |
| External ON, 1 symbol | 4 | 36 |
| External ON, 5 symbols (max recommended) | 12 | 28 |
| External ON, 10 symbols (user override) | 22 | 18 |

### 1.4 Enforcement Points

| Control | Location | Mechanism |
|---------|----------|-----------|
| `INPUT_EXT_MAX_SYMBOLS` | L764 | User input, capped at `STATIC_EXT_maxSymbolsCap` |
| `STATIC_EXT_maxSymbolsCap` | ~L456 | Compile-time constant (5) |
| Budget exceeded flag | L3083 | `GLOBAL_extContext.budgetExceeded := true` |

### 1.5 Failure Mode & Detection

| Failure Mode | Detection Signal | Mitigation |
|--------------|------------------|------------|
| Exceeds 40 contexts | Compilation error or runtime limit | Cap `INPUT_EXT_MAX_SYMBOLS` to 5 |
| Invalid symbol | `na` values propagate | `ignore_invalid_symbol = true` |

### 1.6 Immovable Constraints

> ⚠ **CRITICAL**: R1 and R2 (`request.security_lower_tf`) MUST remain at global scope (L2476-2477). Pine v6 prohibits these inside functions or conditionals.

---

## 2. Loop Time Budget

### 2.1 Pine v6 Limit
- **Per-bar loop time**: ~500ms maximum
- **Total execution**: 40s maximum

### 2.2 Loop Hotspot Inventory

| ID | Location | Loop Type | Bound | Typical Iterations | Gating |
|----|----------|-----------|-------|-------------------|--------|
| L1 | L929 | `for i = 0 to ltf_count - 1` | LTF array size | ~60-240 (intrabar) | `barstate.isconfirmed` |
| L2 | L1029 | `for i = 0 to n - 1` | Function param | ~50-100 | Entropy calc |
| L3 | L1102-1135 | Nested `for` | bins × window | 5×20 = 100 | Legacy entropy mode |
| L4 | L1218-1233 | Nested `for` | states × window | 16×20 = 320 | K-gram entropy |
| L5 | L1334 | `for i = 0 to counts.size() - 1` | K-gram states | 16-64 | Entropy calc |
| L6 | L1348-1358 | `for` × 2 | Hurst length | 100 each | Hurst R/S |
| L7 | L1399-1420 | Nested `for` | chunks × points | 8×12 = 96 | Hurst dyadic |
| L8 | L1498-1571 | Multiple `for` | scales | 3-8 | Hurst averaging |
| L9 | L1628 | `for i = 0 to length - 1` | Z-score length | 14 | Z-score linreg |
| L10 | L1672 | `for i = 0 to array.size(parts) - 1` | Symbol count | 1-5 | Symbol parsing |
| L11 | L1776 | `for i = n - 1 to 1` | Fisher-Yates | n | Permutation test |
| L12 | L1815 | `for sim = 0 to num_sims - 1` | Monte Carlo sims | 100 | Permutation test |
| L13 | L2060-2072 | `for`/`while` | Polyline GC | ~20-100 | ZigZag render |
| L14 | L2453 | `for i = 0 to levelCount - 1` | Fib levels | 15 | Level init |
| L15 | L2525-2640 | Multiple flag loops | Bit flags | 8 each | Flag encode/decode |
| L16 | L2707 | `for i = 0 to total - 1` | Trade history | ~50-200 | Backtest stats |
| L17 | L3074-3088 | `for` × 2 | Symbol count | 5 each | External context |
| L18 | L3430-3434 | `for` × 2 | Buffer GC | ~100-500 | Line/label cleanup |
| L19 | L3985 | `for i = 0 to total - 1` | Trade count | ~50-200 | Backtest aggregation |
| L20 | L4101-4120 | Nested `for` | Zone buffer | ~20×20 | Near-miss detection |
| L21 | L4459-4809 | Multiple `for` | Learning samples | ~50-100 each | Learning metrics |
| L22 | L5406 | `for i = 0 to counts.size() - 1` | Debug display | 16-64 | Debug mode only |

### 2.3 Worst-Case Iteration Estimate

| Subsystem | Worst-Case Iterations | Gating |
|-----------|----------------------|--------|
| Intrabar resolver | 240 | `barstate.isconfirmed` |
| Entropy (legacy) | 500 | `INPUT_ENTROPY_MODE == LEGACY` |
| Entropy (k-gram) | 640 | K-gram mode, init only |
| Hurst (legacy) | 200 | `INPUT_HURST_MODE == LEGACY` |
| Hurst (dyadic) | 100 | Dyadic modes |
| Permutation test | 100 | `INPUT_permtestEnabled` |
| Learning engine | 400 | `barstate.islast` only |
| Object GC | 1000 | Per pivot change |
| **Total worst-case** | **~3000** | Most gated to confirmed bars |

### 2.4 Enforcement Points

| Control | Location | Mechanism |
|---------|----------|-----------|
| `barstate.isconfirmed` | Multiple | Skips loops on intrabar updates |
| `barstate.islast` | Learning, debug | Heavy work only on final bar |
| `perf_shouldComputeHeavy()` | L1048 | Gate for entropy/Hurst/Z-score |
| `INPUT_PERF_PRESET == Fast` | L497 | Reduces window sizes |

### 2.5 Failure Mode & Detection

| Failure Mode | Detection Signal | Mitigation |
|--------------|------------------|------------|
| Loop timeout | Script stops, error in console | Reduce windows, use Fast preset |
| Excessive iterations | Slow chart updates | Use dyadic Hurst, k-gram entropy |

---

## 3. Array Memory Budget

### 3.1 Pine v6 Limit
- **Array elements**: ~100,000 total per script
- **Individual array**: ~100,000 elements

### 3.2 Persistent Array Inventory (`var` arrays)

| ID | Location | Variable | Type | Max Size | Bounded By |
|----|----------|----------|------|----------|------------|
| A1 | L1112 | `return_buffer` | `array<float>` | 100 | Window size |
| A2 | L1113 | `bin_counts` | `array<int>` | 10 | Bin count |
| A3 | L1209-1224 | `SymbolicEntropyState.*` | Multiple | 64+100 | K-gram states + window |
| A4 | L1376-1378 | Hurst buffers | `array<float>` × 3 | 500 each | Window size |
| A5 | L1479-1505 | `DyadicHurstState.*` | Multiple | ~500 | Window + scales |
| A6 | L1641 | `slope_buffer` | `array<float>` | 50 | Z-score length |
| A7 | L2046-2048 | `ZigZagVisualBuffer.*` | Mixed | 1000 | `maxPoints` config |
| A8 | L2423 | `BUF_fibLevels` | `array<FibLevel>` | 15 | Fixed fib levels |
| A9 | L2564 | `CircularBuffer.data` | `array<SetupRecord>` | 50-100 | `INPUT_LEARNING_SAMPLES` |
| A10 | L2681 | `BacktestEngine.history` | `array<BacktestTrade>` | ~200 | Trade count |
| A11 | L3200-3201 | `UI_linesBuffer`, `UI_labelsBuffer` | Drawing arrays | 500 each | Object caps |

### 3.3 Per-Bar Array Allocations

| ID | Location | Pattern | Risk | Mitigation |
|----|----------|---------|------|------------|
| P1 | L1666 | `array.new<string>(0)` in `f_parseSymbols` | Low | Only on first bar |
| P2 | L2450 | `array.new<string>(0)` in fib parsing | Low | Called once per pivot |
| P3 | L3050-3055 | External context arrays | Medium | Reinit only on `barstate.isfirst` |

### 3.4 Memory Budget Estimate

| Category | Elements | Notes |
|----------|----------|-------|
| Rolling buffers (entropy/Hurst/z-score) | ~2,000 | Windows × 3 engines |
| Learning history | ~5,000 | 100 setups × 50 fields |
| ZigZag visuals | ~2,000 | 1000 points + sealed polys |
| Fib levels | ~100 | 15 levels × fields |
| Drawing GC buffers | ~1,000 | Line/label refs |
| **Total estimate** | **~10,000** | Well under 100k limit |

### 3.5 Enforcement Points

| Control | Location | Mechanism |
|---------|----------|-----------|
| `CircularBufferSetupRecord.capacity` | L2559 | Fixed size, overwrites oldest |
| `this.maxPolys` | L2072 | ZigZag polyline GC |
| `perf_getVisualMaxKeep()` | L1078 | Preset-driven retention |
| `while buf.size() > window` | L1003, L1009 | Entropy/Hurst buffer trim |

### 3.6 Failure Mode & Detection

| Failure Mode | Detection Signal | Mitigation |
|--------------|------------------|------------|
| Array limit exceeded | Compilation/runtime error | Circular buffers prevent growth |
| Memory pressure | Slow execution | Reduce learning samples, use Fast preset |

---

## 4. Drawing Object Budget

### 4.1 Pine v6 Limits

| Object Type | Hard Limit |
|-------------|------------|
| Lines | 500 |
| Labels | 500 |
| Boxes | 500 |
| Polylines | 100 |
| Tables | Unlimited (but heavy) |

### 4.2 Object Creation Sites

| ID | Location | Object | Frequency | Gating |
|----|----------|--------|-----------|--------|
| D1 | L1977 | `line.new` (ZigZag segment) | Per pivot extend | Lines mode |
| D2 | L2014 | `line.new` (ZigZag segment) | Per new pivot | Lines mode |
| D3 | L2082 | `polyline.new` (ZigZag) | Per segment batch | Polyline mode |
| D4 | L2121 | `line.new` (live segment) | Per bar | `INPUT_ZZ_LIVE_SEGMENT` |
| D5 | L2315 | `box.new` (entry zone) | Per setup | `INPUT_SHOW_STAT_POS` |
| D6 | L2316-2319 | `line.new` × 3 (SL/TP) | Per setup | Position visual |
| D7 | L2320 | `label.new` (risk label) | Per setup | Position visual |
| D8 | L2341 | `line.new` (smart SL) | Per update | Smart SL enabled |
| D9 | L3439 | `line.new` (fib time zone) | Per zone | `INPUT_SHOW_FIB_TIME` |
| D10 | L3443 | `label.new` (TZ label) | Per zone | `INPUT_FIB_TZ_LABEL` |
| D11 | L3447 | `line.new` (pivot) | Per level | Level enabled |
| D12 | L3451 | `label.new` (pivot) | Per level | Level enabled |
| D13 | L3468 | `line.new` (fib level) | Per level | Level enabled |
| D14 | L3478 | `label.new` (fib level) | Per level | Level enabled |
| D15 | L5366 | `table.new` (entropy debug) | Once | `var`, debug mode |
| D16 | L5417 | `table.new` (Hurst debug) | Once | `var`, debug mode |
| D17 | L5466 | `table.new` (ext debug) | Once | `var`, debug mode |
| D18 | L5500 | `table.new` (flow debug) | Once | `var`, debug mode |

### 4.3 Garbage Collection Sites

| Location | Mechanism | Trigger |
|----------|-----------|---------|
| L3430 | `for` loop deleting oldest lines | Buffer size > max |
| L3434 | `for` loop deleting oldest labels | Buffer size > max |
| L2072 | `while` deleting oldest polylines | Sealed count > `maxPolys` |
| PositionVisual methods | Explicit `line.delete`/`box.delete` | Setup transition |

### 4.4 Object Budget Estimate

| Category | Lines | Labels | Boxes | Polylines |
|----------|-------|--------|-------|-----------|
| ZigZag segments | 50-200 | 0 | 0 | 0-20 |
| Fib levels | 30 | 30 | 0 | 0 |
| Position visuals | 10 | 5 | 5 | 0 |
| Time zones | 20 | 10 | 0 | 0 |
| **Peak estimate** | **~260** | **~45** | **~5** | **~20** |
| **Headroom** | 48% | 91% | 99% | 80% |

### 4.5 Failure Mode & Detection

| Failure Mode | Detection Signal | Mitigation |
|--------------|------------------|------------|
| Object limit hit | Oldest objects auto-deleted | GC buffers explicit deletion |
| Visual clutter | Too many on-chart elements | Use Fast preset, reduce levels |

---

## 5. Compiled Token Budget

### 5.1 Pine v6 Limit
- **Compiled tokens**: 100,000 per script
- **With libraries**: 1,000,000 combined

### 5.2 Token Hotspots

| Category | Location | Estimated Impact | Notes |
|----------|----------|------------------|-------|
| Color constants | L14-35 | Low (~100 tokens) | 20 color definitions |
| String constants | L37-430 | Medium (~2000 tokens) | UI text, tooltips |
| Tooltip blocks | L193+ | High (~3000 tokens) | Long tooltip strings |
| Type definitions | L1924-2730 | Medium (~2000 tokens) | 20 UDTs |
| Function bodies | L867-1800 | High (~15000 tokens) | Stats engines |
| Rendering logic | L3400-4200 | High (~10000 tokens) | Fib drawing |
| Learning engine | L4390-4850 | High (~5000 tokens) | Metrics calc |
| Debug overlays | L5359-5590 | Medium (~2000 tokens) | Table population |

### 5.3 Token Risk Assessment

| Risk | Impact | Mitigation |
|------|--------|------------|
| Large tooltips | ~3000 tokens | Keep concise, avoid duplication |
| Verbose UDT fields | ~500 tokens | Already using descriptive names |
| Inline string building | ~1000 tokens | JSON builders are function-isolated |
| Dead code | Unknown | No evidence of significant dead code |

### 5.4 Failure Mode & Detection

| Failure Mode | Detection Signal | Mitigation |
|--------------|------------------|------------|
| Token limit exceeded | Compilation error: "Script is too long" | Extract to library, trim tooltips |

---

## 6. Architecture Risk Register

### 6.1 Risk: Request Context Exhaustion

| Attribute | Value |
|-----------|-------|
| **Description** | External context with many symbols could approach 40-context limit |
| **Impacted Limit** | Request contexts (40 max) |
| **Location** | L1694-1697 (request calls), L764 (max symbols input) |
| **Current Mitigation** | `STATIC_EXT_maxSymbolsCap = 5` (10 contexts max from external) |
| **Safe Threshold** | ≤10 symbols = 22 contexts = 18 remaining |

### 6.2 Risk: Loop Timeout on Legacy Modes

| Attribute | Value |
|-----------|-------|
| **Description** | Legacy entropy/Hurst modes use O(N²) algorithms |
| **Impacted Limit** | Per-bar loop time (500ms) |
| **Location** | L1102-1135 (legacy entropy), L1348-1420 (legacy Hurst) |
| **Current Mitigation** | `perf_shouldComputeHeavy()` gating, Fast preset degradation |
| **Safe Threshold** | Use dyadic/k-gram modes on windows >100 |

### 6.3 Risk: Object Exhaustion on Volatile Markets

| Attribute | Value |
|-----------|-------|
| **Description** | Rapid pivot changes could create many ZigZag lines |
| **Impacted Limit** | Drawing objects (500 lines) |
| **Location** | L1977-2014 (ZigZag line creation) |
| **Current Mitigation** | GC buffers (L3430-3434), `perf_getVisualMaxKeep()` |
| **Safe Threshold** | `INPUT_ZZ_POLY_MAX_KEEP` capped at 250 |

### 6.4 Risk: Debug Mode Overhead

| Attribute | Value |
|-----------|-------|
| **Description** | Debug overlays create table cells and compute display strings |
| **Impacted Limit** | Execution time, visual clutter |
| **Location** | L5359-5590 (debug tables) |
| **Current Mitigation** | `perf_shouldShowDebugOverlay()` gates to `barstate.islast` only |
| **Safe Threshold** | Use Debug preset only during development |

### 6.5 Risk: Learning Buffer Growth

| Attribute | Value |
|-----------|-------|
| **Description** | Learning history could grow without bound |
| **Impacted Limit** | Array memory (100k elements) |
| **Location** | L2589 (GLOBAL_setupHistory) |
| **Current Mitigation** | `CircularBufferSetupRecord` with fixed capacity (L2559) |
| **Safe Threshold** | `INPUT_LEARNING_SAMPLES` ≤ 200 |

### 6.6 Risk: Intrabar Request Scope Lock

| Attribute | Value |
|-----------|-------|
| **Description** | Intrabar requests cannot be moved/conditionally executed |
| **Impacted Limit** | Code flexibility, request budget |
| **Location** | L2476-2477 |
| **Current Mitigation** | Documented as immovable; 2 contexts always consumed |
| **Safe Threshold** | Design around these 2 fixed contexts |

---

## 7. Detection Signals Summary

| Limit | How to Detect Approaching | How to Detect Exceeded |
|-------|--------------------------|------------------------|
| Request contexts | Count `request.*` calls | Compilation error |
| Loop time | Slow chart updates | Script timeout error |
| Array memory | Memory warnings in console | Runtime error |
| Drawing objects | Visual flickering/deletion | Oldest auto-deleted |
| Compiled tokens | — | Compilation error |

---

## 8. Pine v6 Documentation References

| Topic | Doc Reference | Key Rule |
|-------|---------------|----------|
| Request limits | [Dynamic requests](https://www.tradingview.com/pine-script-docs/en/v6/concepts/Other-timeframes-and-data.html#dynamic-requests) | 40 max per script |
| `request.security_lower_tf` scope | [Lower-timeframe data](https://www.tradingview.com/pine-script-docs/en/v6/concepts/Other-timeframes-and-data.html#lower-timeframe-data) | Must be at global scope |
| Loop limits | [Loops](https://www.tradingview.com/pine-script-docs/en/v6/language/Loops.html) | 500ms per bar |
| Array limits | [Arrays](https://www.tradingview.com/pine-script-docs/en/v6/language/Arrays.html) | ~100k elements |
| Drawing limits | [Drawings](https://www.tradingview.com/pine-script-docs/en/v6/concepts/Drawings.html) | 500/500/500/100 |
| Token limits | [Limitations](https://www.tradingview.com/pine-script-docs/en/v6/writing/Limitations.html) | 100k tokens |

---

*Document generated as part of Task 2: Architecture & Runtime Constraints Audit*
