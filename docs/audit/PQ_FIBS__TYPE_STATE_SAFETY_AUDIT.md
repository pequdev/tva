# PQ_FIBS.pine — Type & State Safety Audit

> **Version**: 1.0  
> **Generated**: 2025-12-22  
> **Source**: `PQ_FIBS.pine` (5,591 lines)  
> **Pine Version**: v6

---

## 1. State Classification Framework

### 1.1 Naming Convention Prefixes

| Prefix | Expected Scope | Persistence | Example |
|--------|---------------|-------------|---------|
| `STATIC_` | Compile-time constant | Immutable | `STATIC_EXT_maxSymbolsCap` |
| `GLOBAL_` | `var` persistent | Across all bars | `GLOBAL_zigzag` |
| `STATE_` | `var` persistent | Across all bars | `STATE_tfDevThresholds` |
| `BUF_` | `var` buffer | Across all bars | `BUF_fibLevels` |
| `UI_` | `var` drawing refs | Across all bars | `UI_linesBuffer` |
| `INPUT_` | User input | Session-scoped | `INPUT_ZZ_DEPTH` |
| `DBG_` | `var` debug counter | Across all bars | `DBG_counterEntropyRuns` |
| (none) | Per-bar series | Recalculated | `i_weightedATR` |

### 1.2 Pine v6 Persistence Rules

| Keyword | Initialization | Persistence | Use Case |
|---------|---------------|-------------|----------|
| `const` | Compile-time | Immutable | Literals, enum values |
| `var` | First bar only | Across bars | Stateful counters, buffers, UDT instances |
| `varip` | First bar, resets on realtime | Intrabar updates | Realtime counters (rarely used) |
| (none) | Every bar | Recalculated | Series data, derived values |

---

## 2. Global State Inventory

### 2.1 UDT Instances (`var`)

| Variable | Type | Location | Initialization | Bounded |
|----------|------|----------|----------------|---------|
| `GLOBAL_zigzag` | `ZigZagEngine` | L3116 | `ZigZagEngine.new()` | N/A (stateful) |
| `GLOBAL_zigzagBuffer` | `ZigZagVisualBuffer` | L3117 | `ZigZagVisualBuffer.new()` | Yes (`maxPolys`, `maxPoints`) |
| `GLOBAL_projection` | `ProjectionState` | L2407 | `ProjectionState.new()` | N/A |
| `GLOBAL_backtest` | `BacktestEngine` | L2727 | `BacktestEngine.new().init()` | Yes (array-backed) |
| `GLOBAL_setupHistory` | `CircularBufferSetupRecord` | L2589 | `CircularBufferSetupRecord.new()` | Yes (`capacity`) |
| `GLOBAL_alertState` | `AlertState` | L2878 | `AlertState.new()` | N/A |
| `GLOBAL_activeTrade` | `SetupState` | L2886 | `f_new_setup_state()` | N/A |
| `GLOBAL_learning` | `LearningMetrics` | L2894 | `LearningMetrics.new()` | N/A |
| `GLOBAL_symbolicEntropy` | `SymbolicEntropyState` | L2919 | `SymbolicEntropyState.new()` | Yes (window-bounded) |
| `GLOBAL_dyadicHurst` | `DyadicHurstState` | L2976 | `DyadicHurstState.new()` | Yes (window-bounded) |
| `GLOBAL_extContext` | `ExternalContextState` | L3046 | `ExternalContextState.new()` | Yes (symbol cap) |
| `GLOBAL_cachedPivots` | `CachedPivots` | L3171 | `CachedPivots.new()` | N/A |
| `GLOBAL_posVisual` | `PositionVisual` | L4015 | `PositionVisual.new()` | N/A |

### 2.2 Map Instances (`var`)

| Variable | Type | Location | Purpose |
|----------|------|----------|---------|
| `STATE_tfDevThresholds` | `map<string, float>` | L1832 | TF → deviation threshold cache |
| `STATE_tfDepths` | `map<string, int>` | L1860 | TF → ZigZag depth cache |
| `GLOBAL_pivotLevelsLogRetracementsMap` | `map<float, float>` | L1892 | Level → retracement log |
| `GLOBAL_pivotLevelsLogCrossedMap` | `map<float, float>` | L1893 | Level → crossed log |

### 2.3 Array Instances (`var`)

| Variable | Type | Location | Max Size | GC Mechanism |
|----------|------|----------|----------|--------------|
| `BUF_fibLevels` | `array<FibLevel>` | L2423 | 15 | Fixed, no growth |
| `UI_linesBuffer` | `array<line>` | L3200 | 500 | Explicit deletion loop |
| `UI_labelsBuffer` | `array<label>` | L3201 | 500 | Explicit deletion loop |

### 2.4 Scalar State (`var`)

| Variable | Type | Location | Purpose |
|----------|------|----------|---------|
| `GLOBAL_prevZzRender` | `string` | L3118 | Detect render mode change |
| `DBG_counterEntropyRuns` | `int` | L2902 | Debug counter |
| `DBG_counterHurstRuns` | `int` | L2903 | Debug counter |
| `DBG_counterZscoreRuns` | `int` | L2904 | Debug counter |
| `DBG_counterLearningRuns` | `int` | L2905 | Debug counter |
| `DBG_counterSpawnAttempts` | `int` | L2906 | Debug counter |
| `DBG_counterSpawnBlocked` | `int` | L2907 | Debug counter |

### 2.5 LevelLog Wrappers (`var`)

| Variable | Type | Location | Backing Map |
|----------|------|----------|-------------|
| `GLOBAL_pivotLevelsLogRetracements` | `LevelLog` | L2466 | `GLOBAL_pivotLevelsLogRetracementsMap` |
| `GLOBAL_pivotLevelsLogCrossed` | `LevelLog` | L2467 | `GLOBAL_pivotLevelsLogCrossedMap` |

---

## 3. Per-Bar Derived State (Non-`var`)

### 3.1 Series Variables

| Variable | Type | Location | Recalculation |
|----------|------|----------|---------------|
| `i_weightedATR` | `float` | L2909 | Every bar via `ta.atr()` |
| `i_vSMA` | `float` | L2910 | Every bar via `ta.sma()` |
| `GLOBAL_regime` | `RegimeState` | L2913 | Every bar (`.new()` per bar) |
| `GLOBAL_projectionPreview` | `SetupState` | L2891 | Every bar (`.new()` per bar) |

### 3.2 Intentional Per-Bar Allocation

| Pattern | Location | Justification |
|---------|----------|---------------|
| `RegimeState.new()` | L2913 | Stateless snapshot, no history needed |
| `SetupState` (projection) | L2891 | Preview-only, not persisted |

---

## 4. Initialization Boundaries

### 4.1 First-Bar Initialization

| Trigger | Variables Affected | Mechanism |
|---------|-------------------|-----------|
| Script load | All `var` declarations | Pine runtime auto-init |
| `barstate.isfirst` | Buffer capacity, config | `.configure()` methods |

### 4.2 Pivot-Change Initialization

| Trigger | Variables Affected | Location |
|---------|-------------------|----------|
| `GLOBAL_zigzag.wasChanged()` | `GLOBAL_activeTrade` | ~L3600+ |
| Pivot spawn | `BacktestTrade.new()` | ~L3700+ |

### 4.3 Session/Symbol Reset

| Trigger | Variables Affected | Notes |
|---------|-------------------|-------|
| Symbol change | All `var` state | Pine runtime reinit |
| Timeframe change | All `var` state | Pine runtime reinit |
| Replay mode | State may differ | TradingView-specific |

### 4.4 Explicit Reset Paths

| Control | Location | Effect |
|---------|----------|--------|
| Render mode change | L3118 | Clears `GLOBAL_zigzagBuffer` |
| Trade closure | `archiveTrade()` L2696 | Moves trade to history |

---

## 5. Type Safety Audit

### 5.1 Explicit Typing Status

| Category | Status | Notes |
|----------|--------|-------|
| UDT field types | ✅ Explicit | All fields have type annotations |
| Function parameters | ✅ Explicit | All params typed |
| Function returns | ⚠️ Mostly inferred | Pine infers from return |
| Local variables | ⚠️ Mostly inferred | Standard Pine pattern |
| `var` declarations | ✅ Explicit | All have type annotations |

### 5.2 Implicit Cast Sites

| Location | Pattern | Risk | Mitigation |
|----------|---------|------|------------|
| Bit flag math | `math.floor(mask / flag) % 2` | Low | Integer division is intentional |
| Time calculations | `time - tPrev` | Low | Both are `int` timestamps |
| Float comparisons | `price > threshold` | Low | Standard float comparison |

### 5.3 Boolean Initialization Patterns

| Variable | Location | Init Value | Safe |
|----------|----------|------------|------|
| `ZigZagEngine.changed` | L1938 | `false` | ✅ |
| `ZigZagEngine.isHighLast` | L1931 | `false` | ✅ |
| `BacktestStats.lowConfidence` | L2663 | `false` | ✅ |
| `BacktestStats.statsReady` | L2664 | `false` | ✅ |

> ✅ **No `na` boolean assignments found** — Pine v6 strict bool compliance verified.

---

## 6. Risky Patterns Inventory

### 6.1 Per-Bar Array Allocation (Low Risk)

| Location | Pattern | Frequency | Mitigation |
|----------|---------|-----------|------------|
| L1666 | `array.new<string>(0)` in `f_parseSymbols` | First bar only | Called in init path |
| L2450 | `array.new<string>(0)` in fib parsing | Per pivot | Small, transient |
| L3050-3055 | External context arrays | `barstate.isfirst` | Gated to first bar |

### 6.2 Unbounded Growth (Mitigated)

| Location | Container | Bound Mechanism | Max Size |
|----------|-----------|-----------------|----------|
| L2589 | `GLOBAL_setupHistory.data` | `CircularBufferSetupRecord.capacity` | 50-100 |
| L2681 | `GLOBAL_backtest.history` | Trade count (implicit) | ~200 |
| L2046 | `ZigZagVisualBuffer.points` | `maxPoints` config | 9500 |
| L2047 | `ZigZagVisualBuffer.sealed` | `maxPolys` + GC | 50 |

### 6.3 State Reset Risks

| Risk | Location | Trigger | Impact |
|------|----------|---------|--------|
| Render mode switch | L3118 | User changes `INPUT_ZZ_RENDER_MODE` | Visual buffer cleared (intended) |
| Stats recalc | L2685 | `statsReady` flag prevents | None (flag protects) |

### 6.4 Type Coercion Hotspots

| Location | Pattern | Type Flow | Safe |
|----------|---------|-----------|------|
| Flag encoding | `state_mask / flag` | `int → float → int` | ⚠️ Precision OK for small flags |
| JSON builders | `str.format()` | Various → `string` | ✅ Explicit |
| Array indexing | `math.min(idx, size-1)` | `int` throughout | ✅ |

---

## 7. Immovable Constraints

### 7.1 Global Scope Requirements

| Constraint | Location | Reason | Impact |
|------------|----------|--------|--------|
| `request.security_lower_tf` | L2476-2477 | Pine v6 requires global scope | Cannot be moved to function or conditional |
| `var` UDT instances | L2407-L4015 | State persistence | Must stay at top-level |
| `input.*` declarations | L511-865 | Pine requirement | Must be at global scope |

### 7.2 Execution Order Dependencies

| Dependency | Location | Constraint |
|------------|----------|------------|
| Intrabar arrays populated | L2476-2477 | Must execute before intrabar logic |
| ZigZag state update | L3116 | Must execute before pivot detection |
| Backtest stats | L2727 | Must execute after trade archive |

---

## 8. Pine v6 Strict Mode Compliance

### 8.1 Compliance Checklist

| Requirement | Status | Evidence |
|-------------|--------|----------|
| Explicit `@version=6` | ✅ | L1 |
| No implicit `bool` → `na` | ✅ | All bools initialized |
| No deprecated functions | ✅ | No v5 constructs found |
| Strict type annotations on UDTs | ✅ | All fields typed |
| No `security()` without `lookahead` | ✅ | Uses `barmerge.lookahead_on/off` |

### 8.2 Lazy Evaluation Awareness

| Location | Pattern | Lazy-Safe |
|----------|---------|-----------|
| `perf_shouldComputeHeavy()` | Gating heavy compute | ✅ |
| `barstate.isconfirmed` checks | Loop gating | ✅ |
| `barstate.islast` checks | Debug/learning | ✅ |

---

## 9. State Lifetime Matrix

| State Category | Lifetime | Reset Trigger | Location |
|----------------|----------|---------------|----------|
| Engine UDTs | Script lifetime | Symbol/TF change | L2407-L4015 |
| Learning metrics | Script lifetime | Manual clear only | L2894 |
| Debug counters | Script lifetime | None (accumulate) | L2902-L2907 |
| Active trade | Until archived | Pivot change + closure | L2886 |
| Projection preview | Per bar | Every bar recalc | L2891 |
| Regime snapshot | Per bar | Every bar recalc | L2913 |
| Drawing buffers | Script lifetime | GC on overflow | L3200-3201 |
| Polyline sealed | Script lifetime | GC on overflow | L3117 |

---

## 10. Risk Register (Type/State)

### R1: Circular Buffer Capacity Mismatch

| Attribute | Value |
|-----------|-------|
| **Description** | `GLOBAL_setupHistory.capacity` may not match `INPUT_LEARNING_SAMPLES` |
| **Location** | L2589 (buffer), L763 (input) |
| **Risk** | Learning engine expects N samples, buffer may be smaller |
| **Mitigation** | Buffer `.init()` called with input value |
| **Detection** | Learning metrics show fewer samples than expected |

### R2: Map Unbounded Growth

| Attribute | Value |
|-----------|-------|
| **Description** | `STATE_tfDevThresholds` and `STATE_tfDepths` grow per unique TF |
| **Location** | L1832, L1860 |
| **Risk** | Switching many TFs could grow maps |
| **Mitigation** | Typical use is single TF; map size << 100 |
| **Detection** | N/A (very low risk) |

### R3: BacktestEngine.history Unbounded

| Attribute | Value |
|-----------|-------|
| **Description** | Trade history array has no explicit cap |
| **Location** | L2681 |
| **Risk** | Many trades could grow array |
| **Mitigation** | Trade count limited by pivot frequency; ~200 max practical |
| **Detection** | `GLOBAL_backtest.getTradeCount()` monitoring |

### R4: Debug Counter Overflow

| Attribute | Value |
|-----------|-------|
| **Description** | `DBG_counter*` variables accumulate indefinitely |
| **Location** | L2902-L2907 |
| **Risk** | Integer overflow after billions of bars (theoretical) |
| **Mitigation** | Debug mode only; counters display on `barstate.islast` |
| **Detection** | N/A (negligible) |

### R5: Per-Bar UDT Allocation

| Attribute | Value |
|-----------|-------|
| **Description** | `GLOBAL_regime` and `GLOBAL_projectionPreview` allocated per bar |
| **Location** | L2891, L2913 |
| **Risk** | Memory churn if Pine doesn't optimize |
| **Mitigation** | Intentional design for stateless snapshots; Pine GC handles |
| **Detection** | Performance monitoring |

---

## 11. Recommendations (Documentation Only)

### 11.1 Consider for Future Tasks

| ID | Recommendation | Priority | Task |
|----|----------------|----------|------|
| T1 | Add explicit capacity init for `BacktestEngine.history` | Low | Optimization |
| T2 | Document `CircularBuffer.capacity` ↔ `INPUT_LEARNING_SAMPLES` contract | Medium | Documentation |
| T3 | Add map size monitoring in debug overlay | Low | Debug enhancement |
| T4 | Consider `varip` for realtime-only counters if needed | Low | Future feature |

### 11.2 No Action Required

| Pattern | Reason |
|---------|--------|
| Per-bar `RegimeState.new()` | Intentional stateless design |
| Inferred function returns | Standard Pine pattern |
| Local variable type inference | Standard Pine pattern |

---

## 12. Pine v6 Documentation References

| Topic | Doc Reference | Key Rule |
|-------|---------------|----------|
| `var` keyword | [Variable declarations](https://www.tradingview.com/pine-script-docs/en/v6/language/Variable-declarations.html) | Initialized on first bar only |
| `varip` keyword | [Variable declarations](https://www.tradingview.com/pine-script-docs/en/v6/language/Variable-declarations.html) | Updates on realtime bars |
| Strict bool | [Type system](https://www.tradingview.com/pine-script-docs/en/v6/language/Type-system.html) | Bool cannot be `na` |
| UDT types | [Objects](https://www.tradingview.com/pine-script-docs/en/v6/language/Objects.html) | Explicit field types required |
| Maps | [Maps](https://www.tradingview.com/pine-script-docs/en/v6/language/Maps.html) | Typed key-value pairs |
| Arrays | [Arrays](https://www.tradingview.com/pine-script-docs/en/v6/language/Arrays.html) | Dynamic sizing, ~100k limit |

---

*Document generated as part of Task 2: Architecture & Runtime Constraints Audit*
