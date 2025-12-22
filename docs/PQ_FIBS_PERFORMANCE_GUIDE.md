# PQ_FIBS Performance Guide

> Limit-Safe Discipline Layer for Runtime Budget Management (Ticket #31)

## Table of Contents

1. [Runtime Limits](#runtime-limits)
2. [Profiler Workflow](#profiler-workflow)
3. [Performance Presets](#performance-presets)
4. [Optimization Patterns](#optimization-patterns)
5. [Plot Budget](#plot-budget)
6. [Token & Compile Hygiene](#token--compile-hygiene)
7. [Troubleshooting](#troubleshooting)

---

## Runtime Limits

Pine Script v6 enforces strict runtime constraints. Exceeding these causes script failures.

| Limit | Basic Account | Premium Account |
|-------|---------------|-----------------|
| Total execution time | 20 seconds | 40 seconds |
| Loop time per bar | 500ms | 500ms |
| Plot count | 64 | 64 |
| Line/Box/Label IDs | 500 | 500 |
| Polyline IDs | 100 | 100 |
| Compiled tokens | 100,000 | 100,000 |
| Imported library tokens | 1,000,000 | 1,000,000 |
| Compilation time | 2 minutes | 2 minutes |

### calc_bars_count Parameter

The `calc_bars_count` parameter in `indicator()` limits the number of historical bars processed.

```pine
indicator(..., calc_bars_count = 0)  // 0 = all bars (default)
indicator(..., calc_bars_count = 10000)  // limits to 10,000 bars
```

**⚠️ Pine Script Limitation:** `calc_bars_count` must be a **compile-time literal integer**. It cannot be wired to an `input()` variable. Changing this value requires script recompilation.

**Recommended values:**
- `0` — All bars (maximum compatibility, default)
- `10000` — For profiling heavy scripts
- `5000` — For extreme performance optimization

**References:**
- [Pine Limitations](https://www.tradingview.com/pine-script-docs/writing/limitations/)
- [Profiling and Optimization](https://www.tradingview.com/pine-script-docs/writing/profiling-and-optimization/)

---

## Profiler Workflow

**Every performance-related change MUST follow this workflow:**

### Step 1: Enable Profiler
1. Open Pine Editor
2. Click **More** (⋮) menu
3. Select **Profiler mode**

### Step 2: Identify Hotspots
- Look for **red/orange "flame" bars** indicating high percentage time
- Document the **top 3 lines/blocks** consuming >10% each
- Note the **code line numbers** and approximate percentages

### Step 3: Optimize Hotspots
Apply patterns from the [Optimization Patterns](#optimization-patterns) section.

### Step 4: Re-measure
- Run profiler again after changes
- **Note:** Profiler adds ~10-20% overhead; results are estimates
- Compare relative improvements, not absolute times

### Step 5: Document
Include in your PR:
- [ ] Profiler screenshot (before/after if applicable)
- [ ] Hotspot summary (top 3 lines, % time)
- [ ] Optimization applied
- [ ] Verified no regressions in default behavior

---

## Performance Presets

The `⚡ Performance Preset` input controls runtime/feature trade-offs:

### 🟢 Balanced (Default)
- All features enabled at user-configured settings
- Confirmed bar gating for heavy computations
- Default behavior — no degradation

### ⚡ Fast
- Degrades optional heavy features for speed
- Reduces entropy window to 10 bars
- Reduces Hurst window to 50 bars
- Forces Dyadic Fast mode for Hurst (O(log N))
- Forces K-gram mode for Entropy (O(1) incremental)
- Optionally skips alert JSON serialization

### 🔍 Debug
- Enables all debug overlays automatically
- Uses table-based rendering (no plot count impact)
- Shows performance dashboard
- Useful for troubleshooting without manually enabling each debug toggle

---

## Optimization Patterns

### Loops

```pine
// ❌ BAD: Linear scan O(N)
for i = 0 to array.size(arr) - 1
    if array.get(arr, i) == target
        result := i
        // continues scanning...

// ✅ GOOD: Early break
for i = 0 to array.size(arr) - 1
    if array.get(arr, i) == target
        result := i
        break  // stop immediately

// ✅ BETTER: Use map for O(1) lookup
var map<string, int> lookup = map.new<string, int>()
result := lookup.get(key)
```

### Memory

```pine
// ❌ BAD: Per-bar allocation
myArray = array.new<float>(100)  // allocates every bar!

// ✅ GOOD: Persistent array with bounded push
var array<float> buffer = array.new<float>(0)
buffer.push(newValue)
while buffer.size() > maxWindow
    buffer.shift()
```

### Gating

```pine
// ❌ BAD: Heavy work on every tick
entropy := f_computeEntropy(...)  // runs on every update

// ✅ GOOD: Gate with confirmed bar + warm-up
if perf_shouldComputeHeavy(INPUT_ENTROPY_ENABLED, entropyWindow)
    entropy := f_computeEntropy(...)
```

### Strings

```pine
// ❌ BAD: Concatenation in loop
result = ""
for i = 0 to 99
    result += str.tostring(array.get(arr, i)) + ","

// ✅ GOOD: Array join pattern
parts = array.new<string>(0)
for i = 0 to 99
    array.push(parts, str.tostring(array.get(arr, i)))
result = array.join(parts, ",")

// ✅ GOOD: Cache expensive operations
var string cachedKey = str.replace(str.replace(levelStr, ".", "_"), "-", "neg")
```

---

## Plot Budget

Pine v6 limits scripts to **64 plot counts**. Various functions consume plot counts:

| Function | Plot Count Cost |
|----------|-----------------|
| `plot()` | 1 (or more with series color) |
| `plotshape()` | 1 |
| `plotchar()` | 1 |
| `alertcondition()` | 1 |
| `bgcolor()` with series color | 1 |
| `barcolor()` with series color | 1 |
| `fill()` with series color | 1 |
| `table.new()` | **0** (free!) |
| `line.new()` | **0** (uses object limit instead) |
| `box.new()` | **0** (uses object limit instead) |
| `label.new()` | **0** (uses object limit instead) |
| `polyline.new()` | **0** (uses object limit instead) |

### Current Plot Budget Audit

The script uses the following plot-count calls:
- `plotchar()` × 2 (exhaustion bar, high volatility)
- `plot()` × 4 (pivot start, pivot end, fib levels for data window)

**Total: ~6 plot counts** (well under 64 limit)

### Rules for New Features

1. **Prefer tables** for debug dashboards
2. **Prefer drawing objects** (line/box/label) for debug traces
3. **Use `display.data_window`** for auxiliary numeric data
4. If adding `plot()`, document the plot count impact in PR

---

## Token & Compile Hygiene

### Token Limits
- **Compiled tokens:** 100,000 max
- **Imported libraries:** 1,000,000 combined max

### Guidelines

1. **Remove unused code**
   - Compiler excludes unused functions ONLY if truly disconnected
   - Delete obsolete functions, don't just comment them

2. **Avoid duplication**
   - Centralize repeated serialization logic
   - Use helper functions instead of copy-paste

3. **Extract to libraries when**
   - A module exceeds ~10,000 tokens
   - Multiple scripts share the same utilities
   - The module is stable and rarely changes

4. **Compilation time**
   - Must complete within 2 minutes
   - Repeated near-limit compilations → temporary ban escalation
   - Keep builds fast by avoiding complex constant expressions

### Library Extraction Candidates

When token pressure increases, consider extracting:
- JSON serialization helpers (~500 tokens)
- Statistical functions (entropy, Hurst) (~2,000 tokens)
- Backtest engine (~3,000 tokens)

---

## Troubleshooting

### "Calculation takes too long to execute"

1. Switch to **⚡ Fast** preset
2. Reduce window sizes (entropy, Hurst)
3. Check for infinite loops or O(N²) patterns
4. Use profiler to find hotspots

### "Script creates too many plots"

1. Audit all `plot*()`, `alertcondition()`, `bgcolor/barcolor/fill`
2. Convert debug plots to tables
3. Use `display.none` for auxiliary plots

### "Max drawing objects exceeded"

1. Implement explicit deletion: `line.delete(oldest)`
2. Use `perf_getVisualMaxKeep()` to limit kept objects
3. Consider polylines (100 limit) vs lines (500 limit)

### Slow on 1m charts

1. Switch to **⚡ Fast** preset
2. Disable unused features (Monte Carlo, Intrabar Precision)
3. Use larger timeframes when possible

---

## PR Checklist

- [ ] Profiler screenshot attached
- [ ] Hotspots documented (top 3 lines, % time)
- [ ] No new `plot*()` calls without justification
- [ ] New drawing objects have deletion strategy
- [ ] Default behavior unchanged (regression test passed)
- [ ] Token count verified (no significant increase)
