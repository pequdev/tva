# PQ_FIBS Loop & Hotspot Report — Final Version

**Version:** vFINAL  
**Date:** 23 December 2025  
**Source:** `PQ_FIBS.pine` (5,605 lines)  
**Branch:** `55-pq_fibs-33-audit---logging-docs-consolidation`  
**Tasks Consolidated:** 6

---

## 1. Executive Summary

| Metric | Value |
|--------|-------|
| **Total `for` Loops** | 41 |
| **Stats Engine Loops** | 10 |
| **Per-Bar Loops** | ~6 (lightweight) |
| **Last-Bar Only Loops** | 8 (heavy) |
| **Worst-Case Per-Bar Iterations** | ~155 |
| **Worst-Case Last-Bar Iterations** | ~252,500 |

**Key Findings:**
- All heavy loops gated by `barstate.islast` or optional toggles
- Symbolic entropy: O(window) → O(1) via incremental updates
- Dyadic Hurst: O(N×scales) → O(log N) via power-of-two scales
- Permutation test: O(sims × samples) but optional + last-bar only

---

## 2. Per-Bar Loop Inventory

### 2.1 Always-Run Loops

| Anchor | Function | Bounds | Complexity | Typical | Worst | Risk |
|--------|----------|--------|------------|---------|-------|------|
| L2453 | Fib level render | `0 to 21` | O(1) | 22 | 22 | 🟢 LOW |
| L2525+ | Flag bit ops | `0 to 15` | O(1) | 16 | 16 | 🟢 LOW |
| L2060 | ZigZag flush | `0 to sealed.size()` | O(polys) | 5 | 50 | 🟢 LOW |

### 2.2 Gated Per-Bar Loops (Regime Engines)

| Anchor | Function | Bounds | Complexity | Typical | Worst | Gating |
|--------|----------|--------|------------|---------|-------|--------|
| L1102 | Entropy bins | `0 to bins-1` | O(bins) | 5 | 10 | Legacy mode |
| L1135 | Entropy buffer | `0 to window-1` | O(window) | 50 | 100 | Legacy mode |
| L1348 | Hurst RS | `0 to length-1` | O(length) | 25 | 100 | Legacy mode |
| L1390 | Hurst scales | `while n < bufSize` | O(log N) | 4 | 6 | Legacy mode |
| L1628 | Z-Score linreg | `0 to length-1` | O(length) | 14 | 50 | Always |

**Note:** Symbolic/Dyadic modes replace O(window) with O(1) incremental updates.

---

## 3. Last-Bar Only Loop Inventory

### 3.1 Learning Engine (barstate.islast)

| Anchor | Function | Bounds | Complexity | Worst | Risk |
|--------|----------|--------|------------|-------|------|
| L4461 | Main learning | `0 to samples-1` | O(samples) | 500 | 🟡 MED |
| L4743 | Log-returns | `0 to samples-1` | O(samples) | 500 | 🟡 MED |
| L4780 | Sortino calc | `0 to samples-1` | O(samples) | 500 | 🟡 MED |
| L4811 | Permtest prep | `0 to samples-1` | O(samples) | 500 | 🟡 MED |

### 3.2 Permutation Test (Optional + islast)

| Anchor | Function | Bounds | Nesting | Complexity | Worst | Risk |
|--------|----------|--------|---------|------------|-------|------|
| L1815 | Monte Carlo | `0 to sims-1` | 2 | O(sims×samples) | 250,000 | 🔴 HIGH |

**Gating:** `INPUT_permtestEnabled and outcomes >= STATIC_PERMTEST_minTrades`

### 3.3 Debug Overlays (Toggle + islast)

| Anchor | Function | Bounds | Complexity | Worst | Risk |
|--------|----------|--------|------------|-------|------|
| L5420 | Entropy counts sum | `0 to counts.size()-1` | O(states) | 256 | 🟢 LOW |

---

## 4. Complexity Class Summary

| Class | Examples | Per-Bar | Last-Bar |
|-------|----------|---------|----------|
| **O(1)** | Symbolic entropy update, flag check | ✅ | ✅ |
| **O(log N)** | Dyadic Hurst scales | ✅ | ✅ |
| **O(levels)** | Fib render (22 fixed) | ✅ | ✅ |
| **O(window)** | Legacy entropy, legacy Hurst | ⚠️ Gated | N/A |
| **O(samples)** | Learning loops | N/A | ✅ |
| **O(sims×samples)** | Permutation test | N/A | ⚠️ Optional |

---

## 5. Heavy Math Operations

### 5.1 Transcendental Functions

| Function | Anchor | Frequency | Notes |
|----------|--------|-----------|-------|
| `math.log()` | L1103, L1380 | Per-bin, per-scale | Shannon entropy, R/S |
| `math.pow()` | L1393, L5393 | Per-scale, debug | Dyadic scales |
| `math.sqrt()` | L1355, L1623 | Per-segment, per-bar | StdDev |
| `math.exp()` | L4685 | Last-bar | Beta distribution |

### 5.2 Floating Point Hotspots

| Operation | Anchor | Risk | Mitigation |
|-----------|--------|------|------------|
| Division | L1103 (prob/log) | Div by 0 | Prob > 0 check |
| Division | L1623 (stdev) | StdDev = 0 | Default 0 |
| Division | L4180 (ATR) | ATR = 0 | ATR floor |
| Log | L1380 | log(0) | Segment check |

---

## 6. Mode Comparison

### 6.1 Entropy Engine

| Mode | Algorithm | Per-Bar | Iterations |
|------|-----------|---------|------------|
| Legacy | Full buffer scan | O(window) | 100 max |
| Binary | Incremental freq | O(1) | 1 |
| K-gram | Incremental freq | O(1) | 1 |

**Recommendation:** Use K-gram (default) for production.

### 6.2 Hurst Engine

| Mode | Algorithm | Per-Bar | Iterations |
|------|-----------|---------|------------|
| Legacy | Full R/S calc | O(N×scales) | 2,000 max |
| Dyadic Fast | Single chunk | O(scale) | ~60 |
| Dyadic Stable | Multi-chunk avg | O(chunks×scale) | ~180 |

**Recommendation:** Use Dyadic Fast (default) for production.

---

## 7. Runtime Budget by Scenario

### 7.1 Default Configuration (Balanced Preset)

| Component | Per-Bar Loops | Iterations |
|-----------|---------------|------------|
| Fib levels | 1 | 22 |
| Z-Score | 1 | 14 |
| Entropy (K-gram) | 0 | 0 (O(1) update) |
| Hurst (Dyadic Fast) | 2 | ~6 |
| **Total** | **4** | **~42** |

### 7.2 Fast Preset

| Component | Per-Bar Loops | Iterations |
|-----------|---------------|------------|
| Fib levels | 1 | 22 |
| Z-Score | 1 | 14 |
| Entropy | 0 | 0 (degraded window) |
| Hurst | 1 | ~4 |
| **Total** | **3** | **~40** |

### 7.3 Debug Preset (Worst Case)

| Component | Per-Bar Loops | Iterations |
|-----------|---------------|------------|
| All above | 4 | ~42 |
| External symbols | 1 | 20 |
| Debug count sums | 1 | 256 |
| **Total** | **6** | **~318** |

### 7.4 Last-Bar Only (Learning Enabled)

| Component | Loops | Iterations |
|-----------|-------|------------|
| Learning main | 4 | 2,000 |
| Permtest (if on) | 1 | 250,000 |
| **Total (no permtest)** | **4** | **2,000** |
| **Total (with permtest)** | **5** | **252,000** |

---

## 8. Loop Risk Matrix

| Risk Level | Count | Examples | Mitigation |
|------------|-------|----------|------------|
| 🟢 LOW | 35 | Fib render, flags, debug | Fixed bounds |
| 🟡 MODERATE | 5 | Legacy entropy, Z-Score | Mode switch |
| 🔴 HIGH | 1 | Permutation test | Optional + islast |

---

## 9. Gating Summary

| Guard | Loops Protected | Location |
|-------|-----------------|----------|
| `barstate.isconfirmed` | Regime engines | L1052 |
| `barstate.islast` | Learning, debug | L1090, L5379+ |
| `INPUT_*_ENABLED` | Feature toggles | Various |
| Mode switches | O(N) → O(1) | L1163, L1449 |
| `_regimeGatesPassed` | Spawn logic | L3854 |

---

## 10. Optimization Opportunities

### 10.1 Already Implemented

| Optimization | Benefit | Location |
|--------------|---------|----------|
| Symbolic entropy | O(window) → O(1) | L1163-1340 |
| Dyadic Hurst | O(N×scales) → O(log N) | L1449-1620 |
| barstate.islast for learning | No per-bar learning | L4406 |
| array.join for strings | O(1) vs O(n²) | L2468, L3527+ |
| Debug toggle gating | Zero overhead when OFF | L1057-1061 |

### 10.2 Potential Future

| Optimization | Benefit | Complexity |
|--------------|---------|------------|
| Incremental linreg | O(n) → O(1) | Medium |
| Lazy Fib level creation | Skip offscreen | Low |
| Batch permtest | Chunk iterations | High |

---

## 11. Invariants

| Invariant | Location | Enforcement |
|-----------|----------|-------------|
| Permtest only on islast | L1815 | Guard in caller |
| Learning only on islast | L4406 | barstate check |
| Legacy modes gated | L1052-1055 | isconfirmed + bar_index |
| Debug loops in guard | L5379+ | perf_shouldShowDebugOverlay |
| History cap = 100 | L2674 | Circular buffer |
| Samples cap = 500 | L4461 | min() with INPUT |

---

*Consolidated from Task 6. Supersedes prior hotspot map versions.*
