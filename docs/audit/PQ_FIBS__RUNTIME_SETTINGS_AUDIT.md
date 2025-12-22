# PQ_FIBS.pine — Runtime Settings Audit

> **Version**: 1.0  
> **Generated**: 2024-12-22  
> **Source**: `PQ_FIBS.pine` (5,591 lines)  
> **Pine Version**: v6

---

## 1. Script Declaration Analysis

### 1.1 Indicator Declaration (L8)

```pine
indicator(
    title = 'Shox Fibonacci Pivots [PQ_MOD]',
    shorttitle = 'FIBS/2.1',
    overlay = true,
    max_lines_count = 500,
    max_labels_count = 500,
    max_boxes_count = 500,
    max_polylines_count = 100,
    max_bars_back = 5000,
    dynamic_requests = true,
    calc_bars_count = 0
)
```

### 1.2 Parameter Analysis

| Parameter | Value | Pine v6 Limit | Utilization | Rationale |
|-----------|-------|---------------|-------------|-----------|
| `overlay` | `true` | N/A | Required | Draws on main chart |
| `max_lines_count` | 500 | 500 | **100%** | Maximum allowed; used for fib levels, ZigZag, position visuals |
| `max_labels_count` | 500 | 500 | **100%** | Maximum allowed; used for fib labels, debug tables |
| `max_boxes_count` | 500 | 500 | **100%** | Maximum allowed; used for position entry zones |
| `max_polylines_count` | 100 | 100 | **100%** | Maximum allowed; used for ZigZag polyline mode |
| `max_bars_back` | 5000 | 5000 | **100%** | Maximum allowed; needed for learning history depth |
| `dynamic_requests` | `true` | N/A | Enabled | Required for external context (up to 40 contexts) |
| `calc_bars_count` | 0 | Literal | All bars | Full history needed for learning engine |

### 1.3 Runtime Constraints Header (L2-7)

The script documents its own constraints inline:

```pine
// ── Pine v6 Runtime Constraints
//   - Execution: 40s max, 500ms per bar loop
//   - Plots: 64 total, Drawing objects: 500 line/box/label, 100 polyline
//   - Compiled tokens: 100k (libraries: 1M combined)
//   - Dynamic requests: 40 max per script
//   - calc_bars_count: compile-time literal (0 = all bars)
```

---

## 2. Drawing Object Budget Analysis

### 2.1 Current Caps vs Usage Estimate

| Object Type | Cap | Estimated Peak Usage | Safety Margin | Notes |
|-------------|-----|---------------------|---------------|-------|
| Lines | 500 | ~200 | 60% | Fib levels + ZigZag segments + position SL/TP |
| Labels | 500 | ~150 | 70% | Fib labels + debug overlays |
| Boxes | 500 | ~50 | 90% | Position entry zones only |
| Polylines | 100 | ~20 | 80% | ZigZag polyline mode only |

### 2.2 Object Lifecycle Management

| Mechanism | Location | Purpose |
|-----------|----------|---------|
| `UI_linesBuffer` | L3200 | `var` array for line GC |
| `UI_labelsBuffer` | L3201 | `var` array for label GC |
| `ZigZagVisualBuffer.sealed` | L2048 | Polyline GC with `maxPolys` cap |
| `perf_getVisualMaxKeep()` | L1078 | Preset-driven object retention |

---

## 3. Request Context Budget Analysis

### 3.1 Request Call Sites

| Location | Type | Context Impact | Gating |
|----------|------|----------------|--------|
| L2476 | `request.security_lower_tf(high)` | 1 context | Always (global scope) |
| L2477 | `request.security_lower_tf(low)` | 1 context | Always (global scope) |
| L1694 (fn) | `f_reqExtClose()` → `request.security(close)` | 1 per symbol | `INPUT_EXT_ENABLED` |
| L1697 (fn) | `f_reqExtOHLCV()` → `request.security(OHLCV)` | 1 per symbol | `INPUT_EXT_ENABLED` |

### 3.2 Context Budget Calculation

| Scenario | Request Contexts | Notes |
|----------|------------------|-------|
| Base (intrabar) | 2 | Always consumed (global scope) |
| External context OFF | +0 | Default state |
| External context ON (1 symbol) | +2 | Close + OHLCV per symbol |
| External context ON (5 symbols) | +10 | Max recommended |
| **Worst Case** | **12** | Well under 40 limit |

### 3.3 Immovable Constraints

> ⚠ **CRITICAL**: `request.security_lower_tf` calls at L2476-2477 MUST remain at global scope per Pine v6 rules. Moving them inside functions or conditionals will cause compilation errors.

---

## 4. Bars Back / History Depth

### 4.1 Settings

| Parameter | Value | Implication |
|-----------|-------|-------------|
| `max_bars_back` | 5000 | Script can reference `[5000]` offsets |
| `calc_bars_count` | 0 | Calculates on ALL available bars |

### 4.2 History-Dependent Components

| Component | Required Depth | Actual Depth | Status |
|-----------|----------------|--------------|--------|
| Learning engine history | ~50-100 setups | Bounded by `INPUT_LEARNING_SAMPLES` | ✅ Safe |
| Entropy rolling window | Up to 100 bars | `INPUT_ENTROPY_WINDOW` (default 20) | ✅ Safe |
| Hurst rolling window | Up to 500 bars | `INPUT_HURST_WINDOW` (default 100) | ✅ Safe |
| ZigZag pivot history | ~50 pivots | Bounded by `STATIC_HIST_MAX_PIVOT` | ✅ Safe |

---

## 5. Performance Presets

### 5.1 Preset Definitions (L497-502)

| Preset | Mode | Purpose |
|--------|------|---------|
| `🟢 Balanced` | Default | All features, confirmed bar gating |
| `⚡ Fast` | Performance | Degrades optional heavy features |
| `🔍 Debug` | Development | Enables all debug overlays |

### 5.2 Preset-Driven Degradation

| Feature | Balanced | Fast | Debug |
|---------|----------|------|-------|
| Entropy window | User setting | Max 10 | User setting |
| Hurst window | User setting | Max 50 | User setting |
| Hurst mode | User setting | Dyadic Fast | User setting |
| Visual objects kept | 250 | 100 | 250 |
| Alert JSON | Full | Skip if enabled | Full |
| Debug overlays | Off | Off | All enabled |

---

## 6. Verification Checklist

| Item | Status | Evidence |
|------|--------|----------|
| `overlay=true` matches chart rendering | ✅ | Script draws fibs/positions on main chart |
| Object caps at maximum allowed | ✅ | 500/500/500/100 declared |
| `dynamic_requests=true` matches external context usage | ✅ | External context uses `request.security` |
| `calc_bars_count=0` is intentional | ✅ | Learning engine needs full history |
| `max_bars_back=5000` sufficient | ✅ | All windows ≤500 bars |
| Request budget documented | ✅ | Max 12 contexts, well under 40 |

---

## 7. Risk Flags

| Risk | Impact | Location | Mitigation |
|------|--------|----------|------------|
| Object exhaustion on high-frequency signals | Medium | ZigZag/Fib drawing | GC buffers prune oldest |
| Full history calc on 100k+ bar charts | Low | `calc_bars_count=0` | Perf gates skip non-confirmed bars |
| Request scope immovability | High | L2476-2477 | Documented as immovable constraint |

---

## 8. Pine v6 Documentation References

| Topic | Doc Reference | Key Rule |
|-------|---------------|----------|
| `indicator()` parameters | [indicator() - Pine v6](https://www.tradingview.com/pine-script-reference/v6/#fun_indicator) | `max_*_count` are hard caps |
| `request.security_lower_tf` | [request.security_lower_tf() - Pine v6](https://www.tradingview.com/pine-script-reference/v6/#fun_request.security_lower_tf) | MUST be at global scope |
| `dynamic_requests` | [Dynamic requests - Pine v6](https://www.tradingview.com/pine-script-docs/en/v6/concepts/Other-timeframes-and-data.html#dynamic-requests) | 40 max per script |
| Drawing object limits | [Drawings - Pine v6](https://www.tradingview.com/pine-script-docs/en/v6/concepts/Drawings.html) | 500 lines/labels/boxes, 100 polylines |

---

*Document generated as part of Task 2: Architecture & Runtime Constraints Audit*
