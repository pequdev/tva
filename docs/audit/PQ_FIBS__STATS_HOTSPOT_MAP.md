# PQ_FIBS Stats Hotspot Map — Loop & Heavy Math Inventory

> **Task 6 Deliverable**: Comprehensive inventory of all loops and heavy math operations in stats engines
> **Branch**: `51-pq_fibs-29-stats-engines-complexity-consolidation`
> **Generated**: 2024-12-23

---

## Executive Summary

This document catalogs all loop constructs and heavy math operations within `PQ_FIBS.pine` stats engines. Each hotspot is classified by:
- **Complexity class**: O(1), O(window), O(N²), O(window×scales), etc.
- **Frequency**: Per-bar, gated, last-bar only, event-driven
- **Runtime risk**: LOW (< 1ms), MODERATE (1-10ms), HIGH (> 10ms potential)

**Key Findings**:
- **41 total `for` loops** identified across the codebase
- **10 loops** are in stats engine hot paths (entropy, hurst, z-score)
- **3 loops** are O(window) per bar — potential runtime risk with large windows
- **Symbolic entropy engine** reduces O(window) → O(1) via incremental updates
- **Dyadic Hurst engine** reduces O(N×scales) → O(log N) via power-of-two scales

---

## 1. Loop Inventory — Stats Engines

### 1.1 Shannon Entropy Engine (Legacy Mode)

| Anchor | Function | Loop Bounds | Nesting | Complexity | Frequency | Risk |
|--------|----------|-------------|---------|------------|-----------|------|
| L1102 | `f_shannonEntropy` | `0 to num_bins - 1` | 1 | O(bins) ≈ O(1) | Per call | 🟢 LOW |
| L1131 | `f_rollingEntropy` | `0 to num_bins - 1` | 1 | O(bins) ≈ O(1) | Per call | 🟢 LOW |
| L1135 | `f_rollingEntropy` | `0 to buf_size - 1` | 2 (nested) | O(window) | Per bar (legacy mode) | 🟡 MODERATE |

**Notes**:
- Legacy mode (`STATIC_ENTROPY_modeLegacy`) iterates full buffer every bar
- Default bins = 5, max bins = 10 — outer loop negligible
- Inner loop bounds: `window` (default 20, max 100)
- Worst case: 100 iterations × 100 bars/chart_window = 10,000 ops/chart

### 1.2 Symbolic Entropy Engine (O(1) Mode)

| Anchor | Function | Loop Bounds | Nesting | Complexity | Frequency | Risk |
|--------|----------|-------------|---------|------------|-----------|------|
| L1218 | `SymbolicEntropyState.init` | `0 to numStates - 1` | 1 | O(states) | Once per init | 🟢 LOW |
| L1227 | `SymbolicEntropyState.init` | `0 to window` | 1 | O(window) | Once per init | 🟢 LOW |
| L1233 | `SymbolicEntropyState.init` | `1 to window` | 1 | O(window) | Once per init | 🟢 LOW |
| L1334 | `SymbolicEntropyState.getDebugInfo` | `0 to counts.size() - 1` | 1 | O(states) | Debug only | 🟢 LOW |

**Notes**:
- Init loops run once per config change (not per bar)
- `update()` method is O(1) — no loops, incremental freq update
- States = 2^k where k ∈ [1, 8], so max 256 iterations (once)

### 1.3 Hurst Exponent Engine (Legacy Mode)

| Anchor | Function | Loop Bounds | Nesting | Complexity | Frequency | Risk |
|--------|----------|-------------|---------|------------|-----------|------|
| L1348 | `f_computeRS` | `0 to length - 1` | 1 | O(length) | Per scale | 🟡 MODERATE |
| L1358 | `f_computeRS` | `0 to length - 1` | 1 | O(length) | Per scale | 🟡 MODERATE |
| L1399 | `f_rollingHurst` | `0 to num_chunks - 1` | 2 (nested in while) | O(chunks) | Per scale | 🟡 MODERATE |
| L1420 | `f_rollingHurst` | `0 to num_points - 1` | 1 | O(scales) | Per bar | 🟢 LOW |

**While Loop**:
| Anchor | Function | Bounds | Complexity | Frequency | Risk |
|--------|----------|--------|------------|-----------|------|
| L1390 | `f_rollingHurst` | `n: minW → bufSize (×1.5)` | O(log(window/minW)) scales | Per bar (legacy mode) | 🟡 MODERATE |

**Notes**:
- Legacy Hurst is O(window × scales × chunks) per bar
- With window=100, minW=20: ~4 scales, each processing full buffer
- Worst case: 100 × 4 × 5 = 2,000 iterations per bar

### 1.4 Dyadic Hurst Engine (O(log N) Mode)

| Anchor | Function | Loop Bounds | Nesting | Complexity | Frequency | Risk |
|--------|----------|-------------|---------|------------|-----------|------|
| L1498 | `DyadicHurstState.init` | `0 to numScales - 1` | 1 | O(log N) | Once per init | 🟢 LOW |
| L1529 | `DyadicHurstState.computeAvgRS` | `0 to numChunks - 1` | 1 | O(chunks) | Per scale (stable mode) | 🟡 MODERATE |
| L1553 | `DyadicHurstState.update` | `0 to numScales - 1` | 1 | O(log N) | Per bar | 🟢 LOW |
| L1571 | `DyadicHurstState.update` | `0 to numScales - 1` | 1 | O(log N) | Per bar | 🟢 LOW |

**Notes**:
- Dyadic scales: powers of 2 from minW to maxW
- With window=100, minW=20: scales = [32, 64] = 2 scales
- Fast mode skips multi-chunk averaging — O(scale) per scale
- Stable mode: O(chunks × scale) per scale, but chunks ≤ 3 typically

### 1.5 Z-Score Momentum Engine

| Anchor | Function | Loop Bounds | Nesting | Complexity | Frequency | Risk |
|--------|----------|-------------|---------|------------|-----------|------|
| L1628 | `f_linregSlope` | `0 to length - 1` | 1 | O(length) | Per bar | 🟡 MODERATE |

**Notes**:
- Linear regression loop runs every bar
- Default length = 14, max = 50
- Uses `meanStdev` helper which has its own O(n) loop (L1029)

### 1.6 Volatility & Helper Functions

| Anchor | Function | Loop Bounds | Nesting | Complexity | Frequency | Risk |
|--------|----------|-------------|---------|------------|-----------|------|
| L1029 | `array.meanStdev` | `0 to n - 1` | 1 | O(n) | Per call | 🟢 LOW |
| L1738 | `f_sortino` | `0 to n - 1` | 1 | O(n) | Per call | 🟢 LOW |
| L1745 | `f_sortino` | `0 to n - 1` | 1 | O(n) | Per call | 🟢 LOW |
| L1792 | `f_maxDrawdown` | `0 to n - 1` | 1 | O(n) | Per call | 🟢 LOW |

---

## 2. Loop Inventory — Learning Engine

### 2.1 Near-Miss Detection

| Anchor | Function | Loop Bounds | Nesting | Complexity | Frequency | Risk |
|--------|----------|-------------|---------|------------|-----------|------|
| L4101 | Near-miss loop 1 | `0 to max_check` | 1 | O(samples) | Per bar (gated) | 🟢 LOW |
| L4120 | Near-miss loop 2 | `0 to max_check` | 1 | O(samples) | Per bar (gated) | 🟢 LOW |

**Notes**:
- `max_check = min(INPUT_LEARNING_SAMPLES - 1, history_size - 1)`
- Default samples = 50, max = 500
- Gated by `INPUT_LEARNING_ENABLED and history_size >= min`

### 2.2 Main Learning Loop

| Anchor | Function | Loop Bounds | Nesting | Complexity | Frequency | Risk |
|--------|----------|-------------|---------|------------|-----------|------|
| L4461 | Learning main loop | `0 to samples_to_check - 1` | 1 | O(samples) | Last bar only | 🟢 LOW |
| L4743 | Log-return loop | `0 to samples_to_check - 1` | 1 | O(samples) | Last bar only | 🟢 LOW |
| L4780 | Sortino loop | `0 to samples_to_check - 1` | 1 | O(samples) | Last bar only | 🟢 LOW |
| L4811 | Permtest loop | `0 to samples_to_check - 1` | 1 | O(samples) | Last bar only | 🟢 LOW |

**Notes**:
- All learning loops are gated by `barstate.islast`
- Only execute on final bar — no per-bar overhead
- Max iterations: 500 (INPUT_LEARNING_SAMPLES max)

### 2.3 Monte Carlo Permutation Test

| Anchor | Function | Loop Bounds | Nesting | Complexity | Frequency | Risk |
|--------|----------|-------------|---------|------------|-----------|------|
| L1815 | `f_permutationTestDD` | `0 to num_sims - 1` | 2 | O(sims × samples) | Last bar only | 🟡 MODERATE |

**Notes**:
- Nested: outer sim loop × inner shuffle + drawdown loop
- Default sims = 100, max = 500
- Worst case: 500 sims × 500 samples = 250,000 iterations
- **Gated by**: `INPUT_permtestEnabled and total_outcomes >= STATIC_PERMTEST_minTrades`
- **Runtime risk**: High if enabled with max settings, but last-bar only

---

## 3. Loop Inventory — Other Components

### 3.1 Intrabar Resolution

| Anchor | Function | Loop Bounds | Nesting | Complexity | Frequency | Risk |
|--------|----------|-------------|---------|------------|-----------|------|
| L929 | `f_resolveIntrabar` | `0 to ltf_count - 1` | 1 | O(ltf_bars) | Per bar (gated) | 🟢 LOW |

**Notes**:
- `ltf_count` = LTF bars within HTF bar (typically 1-60)
- Gated by `INPUT_INTRABAR_ENABLED` and only when both SL/TP hit

### 3.2 Symbol Parsing

| Anchor | Function | Loop Bounds | Nesting | Complexity | Frequency | Risk |
|--------|----------|-------------|---------|------------|-----------|------|
| L1672 | `f_parseSymbols` | `0 to parts.size() - 1` | 1 | O(symbols) | Per bar (gated) | 🟢 LOW |
| L3074 | External symbol copy | `0 to _symCount - 1` | 1 | O(symbols) | Per bar (gated) | 🟢 LOW |
| L3088 | External request loop | `0 to _symCount - 1` | 1 | O(symbols) | Per bar (gated) | 🟢 LOW |

**Notes**:
- Max symbols capped at 20 (`STATIC_EXT_maxSymbolsCap`)
- Gated by `INPUT_EXT_ENABLED and _extGated`

### 3.3 Flag Bit Operations

| Anchor | Function | Loop Bounds | Nesting | Complexity | Frequency | Risk |
|--------|----------|-------------|---------|------------|-----------|------|
| L2525, L2536, L2629, L2640, L2768, L2779 | Flag operations | `0 to STATIC_FLAG_maxBit` | 1 | O(1) ≈ 16 | Per call | 🟢 LOW |

**Notes**:
- Fixed 16-bit loop for bitmask operations
- Constant time regardless of data size

### 3.4 ZigZag Rendering

| Anchor | Function | Loop Bounds | Nesting | Complexity | Frequency | Risk |
|--------|----------|-------------|---------|------------|-----------|------|
| L2060 | `ZigZagVisualBuffer.flushIfFull` | `0 to sealed.size() - 1` | 1 | O(segments) | Event-driven | 🟢 LOW |
| L2453 | Fib level rendering | `0 to levelCount - 1` | 1 | O(levels) | Per bar | 🟢 LOW |
| L2707 | Cleanup loop | `0 to total - 1` | 1 | O(drawings) | Event-driven | 🟢 LOW |

---

## 4. Heavy Math Operations

### 4.1 Transcendental Functions

| Anchor | Function | Operation | Frequency | Notes |
|--------|----------|-----------|-----------|-------|
| L964 | `f_logReturn` | `math.log(price_now / price_prev)` | Per bar | Core volatility metric |
| L1106 | `f_shannonEntropy` | `p * math.log(p) / math.log(2)` | O(bins) per bar (legacy) | Entropy calculation |
| L1107 | `f_shannonEntropy` | `math.log(float(num_bins)) / math.log(2)` | Per bar (legacy) | Max entropy |
| L1204-1205 | `SymbolicEntropyState.init` | `math.log()` × 2 | Once per init | Precomputed constants |
| L1231-1234 | `SymbolicEntropyState.init` | `math.log(c)` in loop | Once per init | Lookup table build |
| L1266 | `SymbolicEntropyState.update` | `math.log()` fallback | Rare (overflow) | Only if count > window |

### 4.2 Square Root Operations

| Anchor | Function | Operation | Frequency | Notes |
|--------|----------|-----------|-----------|-------|
| L994 | `f_annualizedVol` | `math.sqrt(periods_per_year)` | Per bar | Volatility scaling |
| L1035 | `array.meanStdev` | `math.sqrt(variance)` | Per call | Stdev computation |
| L1365 | `f_computeRS` | `math.sqrt(sq_sum / length)` | Per scale | R/S stdev |

### 4.3 Built-in Heavy Functions

| Anchor | Usage | Function | Complexity | Frequency |
|--------|-------|----------|------------|-----------|
| L2911 | `ta.atr(INPUT_ATR_LENGTH)` | ATR | O(window) internally | Per bar |
| L2923 | `ta.stdev(logReturn, window)` | Rolling stdev | O(window) internally | Per bar |
| L3015 | `ta.atr(INPUT_ZSCORE_ATR_LEN)` | ATR for z-score | O(window) internally | Per bar |
| L3025 | `ta.percentrank()` | Percentile rank | O(window) internally | Per bar |
| L3026 | `ta.rsi()` | RSI | O(window) internally | Per bar |

**Notes**:
- Pine built-ins are optimized C++ implementations
- Much faster than equivalent Pine loops
- Cannot be further optimized in Pine code

---

## 5. Complexity Summary by Engine

| Engine | Mode | Per-Bar Complexity | Max Iterations | Runtime Risk |
|--------|------|-------------------|----------------|--------------|
| Shannon Entropy | Legacy | O(window × bins) | 100 × 10 = 1,000 | 🟡 MODERATE |
| Shannon Entropy | Symbolic | O(1) | 1 | 🟢 LOW |
| Hurst Exponent | Legacy | O(window × scales × chunks) | 100 × 4 × 5 = 2,000 | 🟡 MODERATE |
| Hurst Exponent | Dyadic Stable | O(log(N) × chunks × scale) | 2 × 3 × 64 = 384 | 🟢 LOW |
| Hurst Exponent | Dyadic Fast | O(log(N) × scale) | 2 × 64 = 128 | 🟢 LOW |
| Z-Score Momentum | — | O(lr_len + z_len) | 14 + 50 = 64 | 🟢 LOW |
| Learning Engine | — | O(samples) | 500 (last bar only) | 🟢 LOW |
| Permutation Test | — | O(sims × samples) | 500 × 500 = 250,000 | 🟡 MODERATE (gated) |

---

## 6. Hotspot Risk Matrix

### High Priority (Address if performance issues occur)

| Rank | Hotspot | Anchor | Trigger | Mitigation |
|------|---------|--------|---------|------------|
| 1 | Legacy Entropy + Legacy Hurst | L1135, L1390+ | Both legacy modes enabled | Use Symbolic/Dyadic modes |
| 2 | Permutation Test | L1815 | `INPUT_permtestEnabled` with high sims | Reduce sims or disable |
| 3 | Large window Hurst | L1390+ | `INPUT_HURST_WINDOW > 100` | Use Dyadic mode |

### Medium Priority (Monitor)

| Rank | Hotspot | Anchor | Trigger | Mitigation |
|------|---------|--------|---------|------------|
| 4 | Linear regression | L1628 | `INPUT_ZSCORE_LR_LEN = 50` | Reduce length if needed |
| 5 | Learning loops × 4 | L4461, L4743, L4780, L4811 | `INPUT_LEARNING_SAMPLES = 500` | Reduce samples |

### Low Priority (No action needed)

All other loops are O(1), gated, or have fixed small bounds.

---

## 7. Gating Verification

All expensive operations have explicit gates:

| Engine | Gate Expression | Anchor |
|--------|-----------------|--------|
| Entropy | `perf_shouldComputeHeavy(INPUT_ENTROPY_ENABLED, window)` | L2938 |
| Hurst | `perf_shouldComputeHeavy(INPUT_HURST_ENABLED, window)` | L2980 |
| Z-Score | Always runs (cheap, needed for history) | L3015 |
| Learning | `f_canRunLearning(enabled, min, size) and barstate.islast` | L4392 |
| Permtest | `INPUT_permtestEnabled and outcomes >= min` | L4799 |

---

## 8. Recommendations

1. **Default to Fast modes**: PerfController presets should default to Symbolic entropy + Dyadic Fast hurst
2. **Document max safe windows**: Add tooltips warning about performance with window > 100
3. **Gate permtest more aggressively**: Consider last-bar + confirmed only
4. **No code changes required**: Existing gating and mode routing is sufficient

---

## Appendix: Pine v6 Loop Performance Notes

**Doc Reference**: TradingView Pine Script v6 Language Reference — Loops

- Per-bar execution budget: ~500ms
- Nested loops multiply: O(N²) with N=1000 → 1M iterations may timeout
- `array.from()` and `array.copy()` create per-bar allocations — avoid in hot paths
- Prefer `ta.*` built-ins over manual loops when equivalent (C++ optimized)
- `var` arrays persist across bars — no per-bar allocation
