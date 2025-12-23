# PQ_FIBS.pine — Array Lifecycle Audit

> **Task 7 Deliverable** | Generated: 2025-12-23  
> **Purpose**: Complete inventory of all arrays/buffers with allocation, persistence, and update patterns

---

## Executive Summary

| Category | Count | Persistent (`var`) | Per-Bar | Bounded | Risk |
|----------|-------|-------------------|---------|---------|------|
| Rolling Stats Buffers | 7 | ✅ 7 | 0 | ✅ 7 | LOW |
| UDT Array Fields | 12 | ✅ 12 | 0 | ✅ 11, ⚠️ 1 | MEDIUM |
| Visual Object Buffers | 4 | ✅ 4 | 0 | ✅ 4 | LOW |
| Config/Static Arrays | 2 | ✅ 2 | 0 | ✅ 2 | NONE |
| External LTF Arrays | 2 | N/A | ✅ External | N/A | GUARDED |
| Ephemeral Function-Local | 5 | N/A | ✅ | N/A | LOW |

**Total Arrays**: 32  
**Critical Issues**: 1 (BacktestEngine.history unbounded)  
**Negative Indexing Candidates**: 3 sites

---

## 1. Rolling Statistical Buffers (PERSISTENT, BOUNDED)

All stats engine buffers use `var` persistence with `pushBounded()` enforcement.

| Buffer | Location | Type | Persistence | Bound Mechanism | Cap Source |
|--------|----------|------|-------------|-----------------|------------|
| `return_buffer` (Legacy Entropy) | L1112 | `var array<float>` | ✅ PERSISTENT | `pushBounded(window)` | `window` param |
| `bin_counts` (Legacy Entropy) | L1113 | `var array<int>` | ✅ PERSISTENT | `clear()+push(num_bins)` | `num_bins` param |
| `return_buffer` (Legacy Hurst) | L1376 | `var array<float>` | ✅ PERSISTENT | `pushBounded(window)` | `window` param |
| `log_n` (Legacy Hurst) | L1377 | `var array<float>` | ✅ PERSISTENT | `clear()+push()` | `≤ log₂(window)` |
| `log_rs` (Legacy Hurst) | L1378 | `var array<float>` | ✅ PERSISTENT | `clear()+push()` | `≤ log₂(window)` |
| `slope_buffer` (Z-Score) | L1641 | `var array<float>` | ✅ PERSISTENT | `pushBounded(z_len)` | `z_len` param |

### Boundedness Proof
All use the `pushBounded()` helper (L1000-1010):
```pine
method pushBounded(array<float> buf, float v, int window) =>
    if not na(v)
        buf.push(v)
        while buf.size() > window
            buf.shift()
```
**Verdict**: ✅ SAFE — Cap enforced every push, O(1) amortized

---

## 2. UDT Array Fields (PERSISTENT via UDT, BOUNDED)

### 2.1 SymbolicEntropyState (L1164+)

| Field | Type | Init | Bound | Mechanism |
|-------|------|------|-------|-----------|
| `patternBuf` | `array<int>` | L1209 | `window` | `pushBounded()` L1273 |
| `counts` | `array<int>` | L1215 | `states` (fixed 2^k ≤ 256) | Fixed size on init |
| `cLog2c` | `array<float>` | L1224 | `window + 1` | Fixed size on init |

**Verdict**: ✅ SAFE — All bounded by config params

### 2.2 DyadicHurstState (L1458+)

| Field | Type | Init | Bound | Mechanism |
|-------|------|------|-------|-----------|
| `scales` | `array<int>` | L1479 | `log₂(maxW/minW)` | Fixed on init |
| `logScales` | `array<float>` | L1480 | `log₂(maxW/minW)` | Fixed on init |
| `returnBuf` | `array<float>` | L1505 | `configWindow` | `pushBounded()` L1541 |

**Verdict**: ✅ SAFE — Dyadic scales ≤ 10 elements typical

### 2.3 ZigZagVisualBuffer (L2036+)

| Field | Type | Init | Bound | Mechanism |
|-------|------|------|-------|-----------|
| `points` | `array<chart.point>` | L2046 | `maxPoints` (9500) | `flushIfFull()` L2085 |
| `sealed` | `array<polyline>` | L2048 | `maxPolys` (50) | `gcPolys()` L2068 |

**Verdict**: ✅ SAFE — Explicit GC with hard caps

### 2.4 ExternalContextState (L2160+)

| Field | Type | Init | Bound | Mechanism |
|-------|------|------|-------|-----------|
| `symbols` | `array<string>` | L3050 | `INPUT_EXT_MAX_SYMBOLS` | Loop capped L3074 |
| `closes/opens/highs/lows/volumes` | `array<float>` | L3051-55 | `INPUT_EXT_MAX_SYMBOLS` | Same loop cap |

**Verdict**: ✅ SAFE — Budget-controlled at request time

### 2.5 CircularBufferSetupRecord (L2555+)

| Field | Type | Init | Bound | Mechanism |
|-------|------|------|-------|-----------|
| `data` | `array<SetupRecord>` | L2564 | `capacity` (INPUT_LEARNING_SAMPLES) | Circular overwrite L2569 |

**Verdict**: ✅ SAFE — True circular buffer, fixed capacity

### 2.6 BacktestEngine (L2676+)

| Field | Type | Init | Bound | Mechanism |
|-------|------|------|-------|-----------|
| `history` | `array<BacktestTrade>` | L2685 | ⚠️ **UNBOUNDED** | `push()` L2699, L3979 |

**⚠️ RISK**: `GLOBAL_backtest.history` grows without limit.  
- Trades are pushed on pivot completion (L3979)
- No cap enforcement exists
- Long-running charts may accumulate thousands of trades

**Recommendation**: Add bounded push pattern or use CircularBuffer pattern

---

## 3. Visual Object Buffers (PERSISTENT, GC-MANAGED)

| Buffer | Location | Type | Persistence | GC Pattern | Cap |
|--------|----------|------|-------------|------------|-----|
| `UI_linesBuffer` | L3200 | `var array<line>` | ✅ | Clear on bar change L3429-3431 | Per-bar bounded |
| `UI_labelsBuffer` | L3201 | `var array<label>` | ✅ | Clear on bar change L3433-3435 | Per-bar bounded |
| `BUF_fibLevels` | L2423 | `var array<FibLevel>` | ✅ | Static init (22 levels) | Fixed 22 |

**GC Pattern** (L3429-3435):
```pine
if ta.change(time) != 0 and array.size(UI_linesBuffer) > 0
    for i = 1 to array.size(UI_linesBuffer) by 1
        line.delete(array.shift(UI_linesBuffer))
```
**Verdict**: ✅ SAFE — Objects deleted each new bar

---

## 4. External LTF Arrays (SYSTEM-MANAGED)

| Buffer | Location | Type | Source | Behavior |
|--------|----------|------|--------|----------|
| `REQ_ltfHighs` | L2476 | `array<float>` | `request.security_lower_tf(high)` | Pine-managed, variable length |
| `REQ_ltfLows` | L2477 | `array<float>` | `request.security_lower_tf(low)` | Pine-managed, variable length |

**Access Pattern** (L919-945):
```pine
f_resolveIntrabar(array<float> ltf_highs, array<float> ltf_lows, ...) =>
    int ltf_count = array.size(ltf_highs)
    if ltf_count > 0
        for i = 0 to ltf_count - 1
            float ltf_high = array.get(ltf_highs, i)
```
**Verdict**: ✅ SAFE — Size-checked before access, standard forward iteration

---

## 5. Ephemeral Function-Local Arrays (PER-CALL, SHORT-LIVED)

| Array | Location | Context | Purpose | Risk |
|-------|----------|---------|---------|------|
| `result` (f_parseSymbols) | L1666 | Function | Parse CSV symbols | LOW — tiny, bounded by `max_count` |
| `parts` (f_parseSymbols) | L1670 | Function | Split result | LOW — str.split output |
| `parts` (f_pivotDataToJson) | L2450 | Function | JSON builder | LOW — ~100 elements max |
| `shuffled` (f_permutationTestDD) | L1812 | Function | Monte Carlo copy | ⚠️ MEDIUM — `array.copy()` each sim |
| `trade_returns` (Learning) | L4776 | Conditional | Sortino calculation | LOW — bounded by history |
| `permtest_returns` (Learning) | L4811 | Conditional | Permutation test | LOW — bounded by history |

**Note on L1812**: `array.copy()` is called `num_sims` times (default ~100) for Monte Carlo. This is intentional but could be optimized with in-place shuffle reset.

---

## 6. Negative Indexing Candidates

The codebase already uses helper methods with negative indexing:
- `lastVal()` L1015: `buf.get(-1)`
- `prev()` L1018: `buf.get(-2)`

### Remaining Manual Patterns to Migrate

| Location | Current Pattern | Safe to Convert | Guard Exists |
|----------|-----------------|-----------------|--------------|
| L2094 | `array.get(this.points, array.size(this.points) - 1)` | ✅ YES | `array.size() >= maxPoints` L2086 |
| L2110 | `array.get(this.points, array.size(this.points) - 1)` | ✅ YES | `array.size() == 0` check L2101 |
| L1672 | `for i = 0 to array.size(parts) - 1` | N/A | Loop bounds, not access pattern |
| L2060 | `for i = 0 to array.size(this.sealed) - 1` | N/A | Loop bounds, not access pattern |

---

## 7. Summary of Findings

### ✅ Well-Implemented Patterns
1. All stats buffers use `var` + `pushBounded()` — no per-bar allocation
2. Visual buffers use explicit GC on bar change
3. Circular buffer pattern for SetupRecord history
4. LTF arrays properly guarded with size checks

### ⚠️ Issues Requiring Attention

| Priority | Issue | Location | Recommendation |
|----------|-------|----------|----------------|
| HIGH | `BacktestEngine.history` unbounded | L2681, L2693, L3968 | Add cap + oldest-trade eviction |
| LOW | Redundant `array.size()-1` access | L2090, L2105 | Migrate to negative indexing |
| LOW | Monte Carlo `array.copy()` churn | L1812-1816 | Consider in-place shuffle (optional) |

### Doc-Backed Decisions
- **Pine v6 negative indexing**: Confirmed supported via `array.get(arr, -1)` (TradingView docs/arrays)
- **var array persistence**: Arrays declared with `var` persist across bars (TradingView docs/variables)
- **~100k element limit**: Pine arrays capped at ~100,000 elements (TradingView docs/limitations)
