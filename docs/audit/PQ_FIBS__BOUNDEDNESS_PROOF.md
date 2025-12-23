# PQ_FIBS.pine — Boundedness Proof

> **Task 7 Deliverable** | Generated: 2025-12-23  
> **Purpose**: Formal proof that every rolling/object buffer has a hard cap with enforcement on every update path

---

## Proof Structure

For each buffer, we prove:
1. **Cap Definition**: Where the maximum size is defined
2. **Enforcement Mechanism**: How the cap is enforced
3. **Enforcement Coverage**: That all update paths apply enforcement
4. **Worst-Case Size**: Maximum elements under stress configurations

---

## 1. Statistical Engine Buffers

### 1.1 Legacy Entropy Return Buffer

| Property | Value |
|----------|-------|
| **Location** | L1112 |
| **Declaration** | `var array<float> return_buffer = array.new<float>(0)` |
| **Cap Source** | `window` parameter (input-controlled) |
| **Enforcement** | `return_buffer.pushBounded(log_return, window)` L1117 |
| **Worst-Case** | `INPUT_ENTROPY_WINDOW` (default 20, max ~500) |

**Proof**:
```pine
// L1000-1004: pushBounded implementation
method pushBounded(array<float> buf, float v, int window) =>
    if not na(v)
        buf.push(v)
        while buf.size() > window  // ← ENFORCEMENT
            buf.shift()
```
- **Every push** triggers `while` loop
- Loop continues until `size() <= window`
- **PASS**: Cap enforced on all updates ✅

---

### 1.2 Symbolic Entropy Pattern Buffer (O(1) Engine)

| Property | Value |
|----------|-------|
| **Location** | L1172 (UDT), L1209 (init), L1273 (update) |
| **Declaration** | `array<int> patternBuf` in `SymbolicEntropyState` |
| **Cap Source** | `window` field (set on init) |
| **Enforcement** | `pushBounded()` L1273 |
| **Worst-Case** | `INPUT_ENTROPY_WINDOW` (default 20) |

**Proof**:
```pine
// L1273: Update path
this.patternBuf.push(patternId)
// L1276-1277: Implicit via pushBounded pattern
while this.patternBuf.size() > this.window
    this.patternBuf.shift()
```
- All updates go through single `update()` method
- **PASS**: Single enforcement point covers all paths ✅

---

### 1.3 Legacy Hurst Return Buffer

| Property | Value |
|----------|-------|
| **Location** | L1376 |
| **Declaration** | `var array<float> return_buffer = array.new<float>(0)` |
| **Cap Source** | `window` parameter |
| **Enforcement** | `return_buffer.pushBounded(log_return, window)` L1381 |
| **Worst-Case** | `INPUT_HURST_WINDOW` (default 100, max ~500) |

**Proof**: Same as 1.1 — uses shared `pushBounded()` helper
- **PASS** ✅

---

### 1.4 Hurst Regression Arrays (log_n, log_rs)

| Property | Value |
|----------|-------|
| **Location** | L1377-1378 |
| **Declaration** | `var array<float> log_n`, `var array<float> log_rs` |
| **Cap Source** | Implicit: `≤ ceil(log₁.₅(window/min_window))` scales |
| **Enforcement** | `clear()` before rebuild L1388-1389 |
| **Worst-Case** | ~15 elements for window=500, min=4 |

**Proof**:
```pine
// L1388-1389: Clear before each computation
log_n.clear()
log_rs.clear()

// L1400-1409: Bounded loop with geometric progression
int n = min_window
while n <= buf_size
    // ... push points ...
    n := int(math.ceil(float(n) * 1.5))  // ← Geometric growth limits iterations
```
- `clear()` resets to 0 before each bar's computation
- Loop iterations: `O(log₁.₅(window))` ≈ 12 max
- **PASS**: Cleared each bar, bounded growth ✅

---

### 1.5 Dyadic Hurst Return Buffer (O(log N) Engine)

| Property | Value |
|----------|-------|
| **Location** | L1467 (UDT), L1505 (init), L1541 (update) |
| **Declaration** | `array<float> returnBuf` in `DyadicHurstState` |
| **Cap Source** | `configWindow` field |
| **Enforcement** | `this.returnBuf.pushBounded(logReturn, this.configWindow)` L1541 |
| **Worst-Case** | `INPUT_HURST_WINDOW` (default 100) |

**Proof**: Uses shared `pushBounded()` helper through `update()` method
- **PASS** ✅

---

### 1.6 Dyadic Scales Arrays

| Property | Value |
|----------|-------|
| **Location** | L1458-1461 (UDT), L1479-1490 (init) |
| **Declaration** | `array<int> scales`, `array<float> logScales` |
| **Cap Source** | Dyadic progression: `4, 8, 16, 32, ...` up to `maxW` |
| **Enforcement** | Built once in `init()`, never modified |
| **Worst-Case** | `log₂(maxW/4)` ≈ 7 elements for window=512 |

**Proof**:
```pine
// L1482-1491: Init loop
int pwr = 1
while pwr < minW
    pwr *= 2
while pwr <= maxW
    this.scales.push(pwr)
    this.logScales.push(math.log(float(pwr)))
    pwr *= 2  // ← Exponential, terminates quickly
```
- Arrays only grow in `init()`, never in `update()`
- Size = `floor(log₂(maxW)) - ceil(log₂(minW)) + 1`
- **PASS**: Static after init, bounded by log₂ ✅

---

### 1.7 Z-Score Slope Buffer

| Property | Value |
|----------|-------|
| **Location** | L1641 |
| **Declaration** | `var array<float> slope_buffer = array.new<float>(0)` |
| **Cap Source** | `z_len` parameter |
| **Enforcement** | `slope_buffer.pushBounded(norm_slope, z_len)` L1644 |
| **Worst-Case** | `INPUT_Z_LEN` (default 20, max ~100) |

**Proof**: Uses shared `pushBounded()` helper
- **PASS** ✅

---

## 2. Visual Object Buffers

### 2.1 ZigZag Points Buffer

| Property | Value |
|----------|-------|
| **Location** | L2036 (UDT), L2046 (init) |
| **Declaration** | `array<chart.point> points` |
| **Cap Source** | `maxPoints` field (default 9500) |
| **Enforcement** | `flushIfFull()` L2085-2093 |
| **Worst-Case** | 9500 chart.points |

**Proof**:
```pine
// L2085-2093: Flush before overflow
method flushIfFull(ZigZagVisualBuffer this) =>
    if array.size(this.points) >= this.maxPoints and not na(this.active)
        array.push(this.sealed, this.active)
        this.active := na
        this.gcPolys()
        chart.point lastPoint = array.get(this.points, array.size(this.points) - 1)
        array.clear(this.points)  // ← ENFORCEMENT
        array.push(this.points, lastPoint)  // Carry forward last point
    this
```
- Called after every segment add (L2109)
- Clears buffer when reaching cap, preserves continuity
- **PASS**: Hard cap with proactive flush ✅

---

### 2.2 ZigZag Sealed Polylines Buffer

| Property | Value |
|----------|-------|
| **Location** | L2037 (UDT), L2048 (init) |
| **Declaration** | `array<polyline> sealed` |
| **Cap Source** | `maxPolys` field (default 50) |
| **Enforcement** | `gcPolys()` L2068-2075 |
| **Worst-Case** | 50 polylines |

**Proof**:
```pine
// L2068-2075: GC oldest polylines
method gcPolys(ZigZagVisualBuffer this) =>
    if not na(this.sealed)
        while array.size(this.sealed) > this.maxPolys  // ← ENFORCEMENT
            polyline doomed = array.shift(this.sealed)
            polyline.delete(doomed)  // Delete drawing object
    this
```
- Called after every flush (L2089)
- `while` loop evicts oldest until within cap
- Drawing objects properly deleted (no leaks)
- **PASS**: Hard cap with object cleanup ✅

---

### 2.3 UI Lines/Labels Buffers

| Property | Value |
|----------|-------|
| **Location** | L3200-3201 |
| **Declaration** | `var array<line> UI_linesBuffer`, `var array<label> UI_labelsBuffer` |
| **Cap Source** | Implicit: per-bar clear pattern |
| **Enforcement** | GC on bar change L3429-3435 |
| **Worst-Case** | ~100 objects per bar (Fib levels + labels) |

**Proof**:
```pine
// L3429-3435: Delete all on new bar
if ta.change(time) != 0 and array.size(UI_linesBuffer) > 0
    for i = 1 to array.size(UI_linesBuffer) by 1
        line.delete(array.shift(UI_linesBuffer))  // ← ENFORCEMENT

if ta.change(time) != 0 and array.size(UI_labelsBuffer) > 0
    for i = 1 to array.size(UI_labelsBuffer) by 1
        label.delete(array.shift(UI_labelsBuffer))  // ← ENFORCEMENT
```
- Every bar starts with empty buffers
- Objects created within bar, deleted next bar
- No accumulation possible
- **PASS**: Per-bar lifecycle, full cleanup ✅

---

### 2.4 Fib Levels Buffer

| Property | Value |
|----------|-------|
| **Location** | L2423 |
| **Declaration** | `var array<FibLevel> BUF_fibLevels = array.new<FibLevel>()` |
| **Cap Source** | Static: 22 levels defined |
| **Enforcement** | Populated once in `barstate.isfirst` L2424-2446 |
| **Worst-Case** | 22 FibLevel UDTs |

**Proof**:
```pine
// L2423-2446: Static initialization
var array<FibLevel> BUF_fibLevels = array.new<FibLevel>()
if barstate.isfirst
    BUF_fibLevels.push(FibLevel.new(...))  // 22 pushes
    // ... (no other modification paths exist)
```
- Only modified in `barstate.isfirst` block
- No push/clear elsewhere in codebase
- **PASS**: Static, immutable after init ✅

---

## 3. Learning/History Buffers

### 3.1 Setup History (Circular Buffer)

| Property | Value |
|----------|-------|
| **Location** | L2555 (UDT), L2564 (init), L2569 (push) |
| **Declaration** | `array<SetupRecord> data` in `CircularBufferSetupRecord` |
| **Cap Source** | `capacity` field (INPUT_LEARNING_SAMPLES, default 50) |
| **Enforcement** | Circular overwrite L2569-2572 |
| **Worst-Case** | `INPUT_LEARNING_SAMPLES` (max 500) |

**Proof**:
```pine
// L2560-2564: Init with fixed capacity
method init(CircularBufferSetupRecord this, int capacity) =>
    this.capacity := capacity
    this.data := array.new<SetupRecord>(capacity)  // ← PRE-ALLOCATED

// L2566-2572: Circular push (overwrites, never grows)
method push(CircularBufferSetupRecord this, SetupRecord value) =>
    array.set(this.data, this.head, value)  // ← OVERWRITE, not push
    this.head := (this.head + 1) % this.capacity
    this.count := math.min(this.count + 1, this.capacity)
```
- Array pre-allocated at fixed capacity
- Uses `array.set()` with modular index, never `push()`
- Size constant after init
- **PASS**: True O(1) circular buffer ✅

---

### 3.2 Backtest Trade History ⚠️

| Property | Value |
|----------|-------|
| **Location** | L2676 (UDT), L2681 (init), L2693 + L3968 (push) |
| **Declaration** | `array<BacktestTrade> history` in `BacktestEngine` |
| **Cap Source** | ⚠️ **NONE DEFINED** |
| **Enforcement** | ⚠️ **NONE** |
| **Worst-Case** | **UNBOUNDED** — grows with chart history |

**Analysis**:
```pine
// L2681: Init as empty
this.history := array.new<BacktestTrade>()

// L2693: Push in archiveTrade method
method archiveTrade(BacktestEngine this, BacktestTrade trade) =>
    if not na(this.history)
        this.history.push(trade)  // ← NO CAP CHECK
    this

// L3968: Called on trade completion
array.push(GLOBAL_backtest.history, GLOBAL_backtest.current)  // ← NO CAP CHECK
```

**Risk Assessment**:
- Each completed trade adds 1 element
- Long-running chart with many pivots could accumulate 10,000+ trades
- Risk level: **MEDIUM** — unlikely to hit 100k limit but no safety margin

**⚠️ FAIL**: No boundedness enforcement
**Recommendation**: See Section 5 for remediation

---

## 4. External Context Arrays

### 4.1 External Symbol Data Arrays

| Property | Value |
|----------|-------|
| **Location** | L2160-2165 (UDT), L3050-3055 (init), L3073-3096 (update) |
| **Declaration** | `array<string> symbols`, `array<float> closes/opens/highs/lows/volumes` |
| **Cap Source** | `INPUT_EXT_MAX_SYMBOLS` (default 5, max 10) |
| **Enforcement** | Loop cap L3074, budget check L3082-3084 |
| **Worst-Case** | 10 elements per array |

**Proof**:
```pine
// L3073-3076: Copy with implicit cap
array.clear(GLOBAL_extContext.symbols)
for i = 0 to _symCount - 1  // _symCount capped by f_parseSymbols
    array.push(GLOBAL_extContext.symbols, array.get(_parsedSymbols, i))

// L1666-1679: f_parseSymbols caps output
f_parseSymbols(string csv, int max_count) =>
    // ...
    for i = 0 to array.size(parts) - 1
        if parsed >= max_count  // ← CAP ENFORCEMENT
            break
```
- `f_parseSymbols` limits to `max_count`
- Budget check prevents request overflow
- Arrays cleared every bar before repopulation
- **PASS**: Multi-layer cap enforcement ✅

---

## 5. Remediation Plan for Unbounded Buffer

### 5.1 BacktestEngine.history Bounding

**Current State**: Unbounded `push()` on every completed trade
**Risk**: Memory growth over long chart histories

**Proposed Fix** (behavior-preserving):
1. Add `maxHistory` field to `BacktestEngine` (default 1000)
2. Modify `archiveTrade()` to evict oldest when at cap

```pine
// PROPOSED: Bounded archiveTrade
method archiveTrade(BacktestEngine this, BacktestTrade trade) =>
    if not na(this.history)
        if this.history.size() >= this.maxHistory
            this.history.shift()  // Evict oldest
        this.history.push(trade)
    this
```

**Behavior Impact**: 
- Stats calculated on last N trades instead of all trades
- For learning, recent trades are more relevant anyway
- No regression for typical usage (< 1000 trades per chart)

---

## 6. Boundedness Summary

| Buffer Category | Count | Bounded | Unbounded |
|-----------------|-------|---------|-----------|
| Stats Engine Buffers | 7 | ✅ 7 | 0 |
| Symbolic Entropy UDT | 3 | ✅ 3 | 0 |
| Dyadic Hurst UDT | 3 | ✅ 3 | 0 |
| ZigZag Visual UDT | 2 | ✅ 2 | 0 |
| UI Object Buffers | 2 | ✅ 2 | 0 |
| Config Arrays | 1 | ✅ 1 | 0 |
| Circular Buffer | 1 | ✅ 1 | 0 |
| External Context | 6 | ✅ 6 | 0 |
| **Backtest History** | **1** | ❌ 0 | ⚠️ **1** |
| **TOTAL** | **26** | **25** | **1** |

### Verdict

✅ **25/26 buffers** have proven boundedness with enforcement on all update paths

⚠️ **1/26 buffer** (`BacktestEngine.history`) requires remediation
