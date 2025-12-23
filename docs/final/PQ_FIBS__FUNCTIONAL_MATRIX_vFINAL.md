# PQ_FIBS Functional Matrix — Final Version

**Version:** vFINAL  
**Date:** 23 December 2025  
**Source:** `PQ_FIBS.pine` (5,605 lines)  
**Branch:** `55-pq_fibs-33-audit---logging-docs-consolidation`  
**Tasks Consolidated:** 1-10

---

## 1. Subsystem Inventory (10 Core Groups)

| # | Subsystem | Lines | Inputs | Outputs | Frequency | Complexity | Failure Modes |
|---|-----------|-------|--------|---------|-----------|------------|---------------|
| 1 | **Constants & Theme** | L10-425 | — | UI_*, STATIC_* | Compile-time | O(1) | None |
| 2 | **Input Declarations** | L511-865 | User | INPUT_* (211) | Compile-time | O(1) | Invalid ranges |
| 3 | **Performance Gating** | L1043-1092 | Inputs, bar_index | bool gates | Every bar | O(1) | False negatives |
| 4 | **Entropy Engine** | L1095-1340 | close series | entropyNorm [0-1] | Gated | O(window) / O(1)* | Div by zero |
| 5 | **Hurst Engine** | L1340-1620 | close series | hurst [0-1] | Gated | O(log N)* | log(0) edge |
| 6 | **Z-Score Engine** | L1617-1700 | close series | zscore [-3,3] | Gated | O(length) | StdDev=0 |
| 7 | **External Context** | L1661-1730 | symbol CSV | closes[] | Gated | O(symbols) | Invalid symbol |
| 8 | **ZigZag Engine** | L1924-2130, L3113-3365 | H/L, depth | pivot coords | Every bar | O(depth) | Low history |
| 9 | **Backtest Engine** | L2600-2730, L3882-4010 | Price action | win/loss stats | Trade close | O(history) | No trades |
| 10 | **Learning Engine** | L4390-4750 | Setup history | adaptive params | Last bar | O(samples) | Empty buffer |

*Symbolic modes reduce O(window) → O(1); Dyadic mode reduces O(N×scales) → O(log N)

---

## 2. Runtime Budget Summary

### 2.1 Request Budget (40 max)

| Request Type | Count | Scope | Gating |
|--------------|-------|-------|--------|
| `request.security_lower_tf` (highs) | 1 | Global | Always |
| `request.security_lower_tf` (lows) | 1 | Global | Always |
| `request.security` (external) | 0-20 | Function | 4-layer gate |
| **Default Total** | 2 | | Ext disabled by default |
| **Maximum Total** | 22 | | Well under 40 limit |

### 2.2 Drawing Object Budget

| Object | Limit | Reserved | Headroom | Risk |
|--------|-------|----------|----------|------|
| `line` | 500 | ~27 | 94.6% | ✅ Safe |
| `label` | 500 | ~23 | 95.4% | ✅ Safe |
| `box` | 500 | 1 | 99.8% | ✅ Safe |
| `polyline` | 100 | 51 | 49.0% | ⚠️ Moderate |
| `table` | N/A | 5 | N/A | ✅ Safe |

### 2.3 Loop Budget Per Bar

| Context | Loops | Typical Iters | Worst Case | Gated |
|---------|-------|---------------|------------|-------|
| Entropy (symbolic) | 1 | 1 | 1 | ✅ Yes |
| Hurst (dyadic) | 2 | 6 | 10 | ✅ Yes |
| Fib levels | 1 | 22 | 22 | ❌ No |
| ZigZag visual | 1 | ~5 | 100 | ❌ No |
| External context | 1 | 0-5 | 20 | ✅ Yes |
| **Per-bar Total** | ~6 | ~35 | ~155 | |

### 2.4 Last-Bar Only Loops

| Context | Loops | Worst Case | Notes |
|---------|-------|------------|-------|
| Learning main | 4 | 500 each | barstate.islast |
| Permutation test | 1 nested | 250,000 | Optional, islast |
| Debug overlays | 1 | 256 | islast, debug toggle |

---

## 3. Complexity Hotspots (Ranked)

| Rank | Subsystem | Score | Key Factor | Mitigation |
|------|-----------|-------|------------|------------|
| 1 | Learning Engine | 🔴 HIGH | History scan × metrics | islast gate |
| 2 | Permutation Test | 🔴 HIGH | O(sims × samples) | Optional + islast |
| 3 | Hurst Legacy | 🟠 MED-HIGH | O(window × scales) | Dyadic mode |
| 4 | Entropy Legacy | 🟠 MEDIUM | O(window) | Symbolic mode |
| 5 | Statistical Position | 🟡 MEDIUM | MAE/MFE tracking | Pivot-gated |
| 6 | ZigZag Engine | 🟡 MEDIUM | State machine | Always runs |
| 7 | Backtest Engine | 🟡 MEDIUM | Trade lifecycle | Event-driven |
| 8 | External Context | 🟢 LOW | Symbol iteration | Multi-layer gate |
| 9 | Fib Level Calc | 🟢 LOW | Simple arithmetic | 22 fixed levels |
| 10 | Debug Overlays | 🟢 LOW | Table population | islast + toggle |

---

## 4. Input/Output Flow Matrix

### 4.1 Regime Subsystem

```
close → [Entropy Engine] → entropyNorm → [RegimeState] → entropyOk
close → [Hurst Engine] → hurst → [RegimeState] → hurstOk  
close → [Z-Score Engine] → zscore → [RegimeState] → zscoreOk
                                          ↓
                                    _regimeGatesPassed
```

### 4.2 Signal Flow

```
H/L → [ZigZag Engine] → GLOBAL_zigzag.changed
                              ↓
                        [CachedPivots] → pivot coords
                              ↓
                        [FibLevel.draw] → level prices
                              ↓
                        [StatisticalPosition] → entry/SL/TP
                              ↓
                        [PositionVisual] → box/lines/labels
                              ↓
                        [AlertDispatch] → alert() calls
```

### 4.3 Learning Flow

```
Trade closes → [SetupHistory buffer] → GLOBAL_backtest.history
                                              ↓
                                    [LearningEngine] @ islast
                                              ↓
                                    GLOBAL_learning.{winRate, slMult, tpMult, ...}
                                              ↓
                                    [StatisticalPosition] → adaptive SL/TP
```

---

## 5. Failure Mode Analysis

| Subsystem | Mode | Trigger | Impact | Mitigation |
|-----------|------|---------|--------|------------|
| Entropy | Div by zero | window=0 | NaN prop | Min window=10 |
| Hurst | log(0) | Empty segment | NaN prop | Segment check |
| Z-Score | Div by zero | StdDev=0 | Inf z-score | Default 0 |
| External | Bad symbol | Typo in CSV | na values | ignore_invalid |
| Learning | Empty buf | No trades | NaN metrics | min samples |
| Position | ATR=0 | Flat market | Div error | ATR floor |
| ZigZag | No bars | bar_index < depth | No pivots | Bar check |
| Backtest | No trades | Early chart | Empty stats | Trade check |
| Drawing | Limit hit | Many objects | Not drawn | FIFO prune |
| Alert | Msg > 4KB | Large JSON | Truncation | Field priority |

---

## 6. GC & Object Lifecycle Patterns

| Module | Object | Pattern | Delete Site | Cap |
|--------|--------|---------|-------------|-----|
| ZigZag LINES | line | Delete-on-change | L2012 | 1 |
| ZigZag POLY | polyline | FIFO eviction | L2077 | maxPolys |
| ZigZag POLY | line (live) | Toggle hide | L2122 | 1 |
| PositionVisual | box,line,label | Atomic replace | L2298-2312 | 5 |
| FibLevel | line,label | Redraw delete | L3473,L3476 | 44 |
| UI Buffers | line,label | Per-bar clear | L3442,L3446 | ~50/bar |
| Debug Tables | table | var persist | N/A | 5 |

---

## 7. Rendering Cadence

| Module | Trigger | Frequency | Heavy Op |
|--------|---------|-----------|----------|
| ZigZag line | Pivot confirm | Per-pivot | line.new |
| ZigZag poly | Segment add | Per-pivot | polyline.new |
| ZigZag live | Every bar | Per-bar | line.set_* |
| PositionVisual | needsRecreate | Per-pivot | All objects |
| FibLevel | needsFullRedraw | Per-pivot | 44 objects |
| UI Buffers | Bar change | Per-bar | Clear + redraw |
| Debug Tables | barstate.islast | Once | table.cell |

---

## 8. Debug Path Audit

### 8.1 Debug Toggles

| Toggle | Location | Gate Function |
|--------|----------|---------------|
| `INPUT_ENTROPY_DEBUG` | L736 | `perf_shouldShowDebugOverlay()` |
| `INPUT_HURST_DEBUG` | L744 | `perf_shouldShowDebugOverlay()` |
| `INPUT_EXT_DEBUG` | L770 | `perf_shouldShowDebugOverlay()` |
| `INPUT_DEBUG_FLOW` | L772 | `perf_shouldShowDebugOverlay()` |
| `INPUT_PERF_SHOW_DASHBOARD` | L519 | `perf_shouldShowDashboard()` |
| `STATIC_PERF_presetDebug` | L511 | Enables all overlays |

### 8.2 Gate Function Implementation

```pine
perf_shouldShowDebugOverlay(bool explicit_toggle) =>
    (explicit_toggle or INPUT_PERF_PRESET == STATIC_PERF_presetDebug) and barstate.islast
```

### 8.3 Debug OFF Overhead: **PASS**

| Check | Result |
|-------|--------|
| Strings built before guard? | ❌ No |
| Tables created before guard? | ❌ No — `var` inside guard |
| Loops run before guard? | ❌ No |
| Objects created before guard? | ❌ No |

**Verdict:** Debug OFF = constant-time boolean checks only.

---

## 9. String Builder Pattern Compliance

### 9.1 `array.join()` Usage (20 occurrences)

| Location | Context | Pattern |
|----------|---------|---------|
| L2468 | LevelLog.toJson | `array.join(parts, '')` |
| L3527 | Alert barstate JSON | `array.join(array.from(...), '')` |
| L3535 | Alert OHLCV JSON | `array.join(array.from(...), '')` |
| L3637 | Alert full JSON | `array.join(array.from(...), '')` |
| L3706-3712 | Event log JSON | `array.join(array.from(...), '')` |

### 9.2 Remaining Concatenation (Debug Only)

| Location | Context | Verdict |
|----------|---------|---------|
| L5424 | Entropy buffer display | ✅ Inside guard, islast |
| L5449-5451 | Hurst scales display | ✅ Inside guard, islast |
| L5500 | Ext valid count | ✅ Inside guard, islast |
| L5538 | Block rate % | ✅ Inside guard, islast |

**Verdict:** All concatenation in debug paths properly gated by `barstate.islast`.

---

## 10. Integration Coupling Map

| Source → Target | Coupling | Strength |
|-----------------|----------|----------|
| ZigZag → CachedPivots | Data | Strong |
| CachedPivots → FibLevel | Data | Strong |
| FibLevel → PositionVisual | Coords | Strong |
| Regime → Learning | Modulation | Medium |
| Learning → Position | Adaptation | Medium |
| Position → Alerts | Payload | Strong |
| All → Debug | Read-only | Weak |

---

## 11. Refactoring Safety Zones

### 11.1 Safe (Low Risk)

| Zone | Lines | Reason |
|------|-------|--------|
| JSON helpers | L3490-3705 | Pure functions |
| Debug tables | L5359-5605 | Isolated, read-only |
| Constants | L10-425 | No logic |
| Math helpers | L867-1000 | Pure functions |

### 11.2 Caution (Medium Risk)

| Zone | Lines | Reason |
|------|-------|--------|
| Fib rendering | L3465-3510 | Object lifecycle |
| Position visual | L2290-2360 | Stateful objects |
| Learning update | L4390-4750 | Complex state |

### 11.3 Critical (High Risk)

| Zone | Lines | Reason |
|------|-------|--------|
| ZigZag engine | L1924-2130, L3280-3365 | Core state machine |
| Global requests | L2476-2477 | Scope-critical |
| Pivot caching | L3145-3210 | Signal integrity |
| Intrabar resolver | L725-727, L919-955 | Backtest accuracy |

---

## 12. Invariants

| Invariant | Location | Enforcement |
|-----------|----------|-------------|
| Polyline cap ≤ maxPolys + 1 | L2073-2077 | gcPolys() FIFO |
| Debug only on islast | L1057-1061 | Gate functions |
| Requests at global scope | L2476-2477 | Pine constraint |
| Learning on islast only | L4406 | barstate check |
| Setup history circular | L2674 | shift on cap |
| FibLevel redraw on change | L3800 | needsFullRedraw |

---

*Consolidated from Tasks 1-10 audit artifacts. Supersedes all prior matrix versions.*
