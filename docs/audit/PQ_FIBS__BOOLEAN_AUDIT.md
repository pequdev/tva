# PQ_FIBS Boolean Audit — v6 Semantics & Lazy Evaluation

> **Task 5 Deliverable**: Boolean Inventory + Risk Classification
> **Branch**: `50-pq_fibs-28-boolean-semantics-lazy-wiring`
> **Generated**: 2024-12-22

---

## Executive Summary

Pine Script v6 enforces **strict boolean semantics**: `bool` values must be `true` or `false`, never `na`.  
This audit inventories all boolean symbols in `PQ_FIBS.pine`, classifies their `na` risk, and confirms lazy evaluation compliance.

**Key Findings**:
- ✅ **1 `var bool`** declaration — properly initialized (`false`)
- ✅ **~100+ `bool` locals** — all initialized to literal `true`/`false`
- ⚠️ **2 patterns with NA_RISK** — comparisons involving `ta.valuewhen` results (early bars)
- ✅ **UDT bool fields** — all have explicit default values in type definitions
- ✅ **Lazy evaluation** — already implemented at critical spawn gates (L3838-3843)
- ✅ **No `bool x = na`** assignments anywhere

---

## 1. Pine v6 Boolean Semantics — Doc References

| Rule | Pine v6 Behavior | Source |
|------|------------------|--------|
| Boolean values | Only `true` or `false`, never `na` | Pine v6 Type System docs |
| `ta.change(bool)` | Returns `true` if value differs from previous bar | Pine v6 Functions/ta.change |
| Comparison with `na` series | Comparison returns `na` if either operand is `na` | Pine v6 `na` handling rules |
| Lazy `and/or` | Short-circuit: `A and B` skips `B` if `A` is `false` | Pine v6 Keywords/and |
| `nz()` on bool | **Not recommended** — use explicit `na` checks | Pine v6 best practices |

**Doc-Backed Decision**: Booleans derived from series comparisons where the series may be `na` can produce `na` boolean results on early bars. These must be explicitly guarded.

---

## 2. Boolean Inventory

### 2.1 `var bool` Declarations (Persistent Booleans)

| Anchor | Variable | Default | Risk | Consumer |
|--------|----------|---------|------|----------|
| L5195 | `wasProjection` | `false` | ✅ SAFE | Projection state change detection |

**Assessment**: Only 1 `var bool` in codebase. Properly initialized to `false`. No risk.

---

### 2.2 UDT Boolean Fields

| UDT | Field | Default | Anchor | Risk |
|-----|-------|---------|--------|------|
| `RegimeState` | `entropyOk` | `true` | L2139 | ✅ SAFE |
| `RegimeState` | `hurstOk` | `true` | L2142 | ✅ SAFE |
| `RegimeState` | `strongMomentum` | `false` | L2147 | ✅ SAFE |
| `RegimeState` | `momentumOk` | `true` | L2148 | ✅ SAFE |
| `ExternalContextState` | `enabled` | `false` | L2156 | ✅ SAFE |
| `ExternalContextState` | `gated` | `false` | L2157 | ✅ SAFE |
| `ExternalContextState` | `budgetExceeded` | `false` | L2158 | ✅ SAFE |
| `ExternalContextState` | `extOk` | `true` | L2166 | ✅ SAFE |
| `LearningMetrics` | `useTakeProfitAggressive` | `false` | L2209 | ✅ SAFE |
| `LearningMetrics` | `isPermtestSignificant` | `false` | L2255 | ✅ SAFE |
| `LearningMetrics` | `isHealthy` | `true` | L2258 | ✅ SAFE |
| `LearningMetrics` | `needsRecalculation` | `false` | L2262 | ✅ SAFE |
| `SymbolicEntropyState` | `initialized` | `false` | L1165 | ✅ SAFE |
| `ZigZagEngine` | `liveEnabled` | `true` | L2042 | ✅ SAFE |
| `TradeState` | `isProjection` | `false` | L2819 | ✅ SAFE |
| `TradeState` | `isLong` | `false` | L2820 | ✅ SAFE |
| `TradeState` | `zoneTouched` | `false` | L2821 | ✅ SAFE |
| `TradeState` | `stopLossHit` | `false` | L2822 | ✅ SAFE |
| `TradeState` | `takeProfitHit` | `false` | L2823 | ✅ SAFE |
| `TradeState` | `isInZone` | `false` | L2824 | ✅ SAFE |
| `TradeState` | `isZoneEntry` | `false` | L2825 | ✅ SAFE |
| `TradeState` | `useAggressiveTakeProfit` | `false` | L2826 | ✅ SAFE |
| `TradeState` | `isExpired` | `false` | L2827 | ✅ SAFE |

**Assessment**: All 23 UDT boolean fields have explicit defaults. No risk.

---

### 2.3 Regime Gate Booleans (Critical Path)

| Anchor | Variable | Producer Expression | Risk | Notes |
|--------|----------|---------------------|------|-------|
| L2966 | `entropyOk` | `not INPUT_ENTROPY_ENABLED or _entReg != STATIC_ENTROPY_regimeDisordered` | ✅ SAFE | Comparison with int constant |
| L3011 | `hurstOk` | `not INPUT_HURST_ENABLED or not INPUT_HURST_BLOCK_RANDOM or _hurstReg != STATIC_HURST_regimeRandom` | ✅ SAFE | Chain of bools + int comparison |
| L3021 | `momentumOk` | `not INPUT_ZSCORE_ENABLED or _strongMom` | ✅ SAFE | Bool input + bool variable |
| L3063 | `_extGated` | `INPUT_EXT_ENABLED and f_computeExtGate(...)` | ✅ SAFE | Lazy eval: cheap input first |
| L3107 | `_extOk` | Assigned in conditional block | ✅ SAFE | Defaults to `true`, conditionally set |
| L3843 | `_regimeGatesPassed` | `GLOBAL_regime.entropyOk and ... and GLOBAL_extContext.extOk` | ✅ SAFE | All operands are pre-computed bools |

**Assessment**: All regime gates are composed of strict booleans. No `na` risk.

---

### 2.4 Pivot Change Detection Booleans

| Anchor | Variable | Producer | Risk | Notes |
|--------|----------|----------|------|-------|
| L3245 | `_pivotChange_iPrev` | `ta.change(GLOBAL_zigzag.iPrevPivot != 0)` | ✅ SAFE | Comparison with 0 always bool |
| L3246 | `_pivotChange_pPrev` | `ta.change(GLOBAL_zigzag.pPrevPivot != 0)` | ✅ SAFE | Comparison with 0 always bool |
| L3247 | `_pivotChange_iLast` | `ta.change(GLOBAL_zigzag.iLastPivot != 0)` | ✅ SAFE | Comparison with 0 always bool |
| L3248 | `_pivotChange_pLast` | `ta.change(GLOBAL_zigzag.pLastPivot != 0)` | ✅ SAFE | Comparison with 0 always bool |

**Assessment**: These use `ta.change()` on boolean expressions (comparisons). Since the comparisons are against literal `0` (never `na`), the result is always a valid boolean.

---

### 2.5 RSI Divergence Booleans (⚠️ NA_RISK Pattern)

| Anchor | Variable | Producer | Risk | Notes |
|--------|----------|----------|------|-------|
| L3768 | `GLOBAL_rsi_at_pivot` | `ta.valuewhen(GLOBAL_zigzag.changed, GLOBAL_regime.rsi, 0)` | ⚠️ NA source | Returns `na` until first pivot |
| L3769 | `GLOBAL_priceAtPivot` | `ta.valuewhen(GLOBAL_zigzag.changed, close, 0)` | ⚠️ NA source | Returns `na` until first pivot |
| L4355 | `bullish_div` | `check_long and close < price_at_pivot and rsi > rsi_at_pivot` | ⚠️ NA_RISK | Uses potentially `na` operands |
| L4357 | `bearish_div` | `not check_long and close > price_at_pivot and rsi < rsi_at_pivot` | ⚠️ NA_RISK | Uses potentially `na` operands |

**Risk Analysis**:
- `GLOBAL_rsi_at_pivot` and `GLOBAL_priceAtPivot` return `na` on bars before the first pivot change
- Comparisons like `rsi > rsi_at_pivot` where `rsi_at_pivot` is `na` produce `na` boolean
- However, this is **mitigated by context**: these booleans are only used within the `if not na(GLOBAL_activeTrade.setup_bar)` block (L4360), which only executes when there's an active trade — implying pivots have already occurred

**Mitigation Status**: ✅ Contextually safe — active trade implies prior pivots exist

---

### 2.6 Trade State Booleans

| Anchor | Variable | Default | Risk |
|--------|----------|---------|------|
| L921 | `ltf_resolved` | `false` | ✅ SAFE |
| L934-935 | `sl_hit`, `tp_hit` | Ternary expression | ✅ SAFE |
| L1951 | `newSegment` | `false` | ✅ SAFE |
| L3030-3036 | `bullCandle`, `bearCandle`, `exhaustVol`, `highVolatility` | Direct comparisons | ✅ SAFE |
| L3789 | `needsFullRedraw` | `GLOBAL_zigzag.changed or ...` | ✅ SAFE |
| L3872 | `skip_this_bar` | `GLOBAL_zigzag.changed` | ✅ SAFE |
| L3919-3920 | `sl_hit`, `tp_hit` (backtest) | Ternary with OHLC | ✅ SAFE |

---

## 3. Lazy Evaluation Compliance

### 3.1 Documented Lazy Evaluation Guidance

**Anchor**: L1039-1042

```pine
// ── LAZY EVALUATION GUARDS - Execution flow optimization
//   Pine v6: strict bool (true|false only, never na), lazy and/or evaluation
//   Order conditions cheapest → expensive to exploit short-circuit
//   Side-effect calls must NOT be in and/or operands (evaluation may be skipped)
```

### 3.2 Critical Lazy Evaluation Sites

| Anchor | Pattern | Ordering | Assessment |
|--------|---------|----------|------------|
| L3842 | `_spawnGatesPassed` | `changed and na(current) and not na(pivot)...` | ✅ Cheap→Expensive |
| L3843 | `_regimeGatesPassed` | `entropyOk and hurstOk and momentumOk and extOk` | ✅ All pre-computed bools |
| L3848 | Spawn guard | `_spawnGatesPassed and _regimeGatesPassed` | ✅ Pre-computed bools |
| L4041 | `use_learned` | `INPUT_USE_LEARNED_SL and INPUT_LEARNING_ENABLED and size() >= min and mult > 0` | ✅ Inputs first, then size check |
| L4073 | `has_learned_tp` | `INPUT_LEARN_TP and INPUT_LEARNING_ENABLED and size() >= min` | ✅ Inputs first |
| L4392 | Learning loop guard | `INPUT_LEARNING_ENABLED and size() >= min and needsRecalculation` | ✅ Flag-gated lazy execution |

### 3.3 Performance Gate Functions

| Anchor | Function | Purpose | Ordering |
|--------|----------|---------|----------|
| L1043 | `f_readyForRegime(window_bars)` | Data sufficiency check | Simple int comparison |
| L1048 | `perf_shouldComputeHeavy(enabled, window_bars)` | Heavy compute guard | `enabled and isconfirmed and bar_index >= window` |
| L1051 | `perf_shouldRenderDebug()` | Debug render guard | `preset == debug and islast` |
| L1057 | `perf_shouldShowDashboard()` | Dashboard guard | `show and islast` |

---

## 4. NA-as-False Pattern Audit

### 4.1 `nz()` Usage on Booleans

**Search Result**: No `nz()` calls on boolean variables or expressions.

All `nz()` calls are on numeric types:
- L1630: `nz(src[length - 1 - i])` — float
- L2375-2378: `nz(this.localHigh, currentHigh)` — float
- L2911: `nz(volume)` — float
- L3029: `nz(volume)` — float
- L3780-3783: `nz(GLOBAL_cachedPivots.*)` — float
- L3818: `nz(ta.atr(...), 1.0)` — float
- L5076-5226: Various `nz()` on trade fields — float/int

**Assessment**: ✅ No boolean `nz()` misuse found.

### 4.2 Implicit Falsy Patterns

No patterns found where `na` boolean would be used as implicit `false`.

---

## 5. Risk Classification Summary

| Risk Tag | Count | Status |
|----------|-------|--------|
| ✅ SAFE | 95+ | All properly initialized |
| ⚠️ NA_RISK | 2 | `bullish_div`, `bearish_div` — contextually mitigated |
| ⚠️ INIT_RISK | 0 | All `var bool` initialized |
| ⚠️ DRIFT_RISK | 0 | No stateful bool reset issues found |
| 🎯 PERF_GATE | 6 | Already implemented (L3842-3848, L4041, L4073, L4392) |

---

## 6. Recommendations

### 6.1 Optional Hardening (Low Priority)

The `bullish_div` and `bearish_div` expressions at L4355-4357 could be hardened with explicit `na` guards:

```pine
// Current (works due to context):
bool bullish_div = check_long and close < price_at_pivot and rsi > rsi_at_pivot

// Hardened (explicit guard):
bool bullish_div = check_long and not na(price_at_pivot) and not na(rsi_at_pivot) and close < price_at_pivot and rsi > rsi_at_pivot
```

**Recommendation**: Optional — current code is safe due to the enclosing `if not na(GLOBAL_activeTrade.setup_bar)` guard.

### 6.2 Documentation Alignment

The lazy evaluation guidance at L1039-1042 is excellent. Consider adding a reference to this in the spawn gate comments for cross-linking.

---

## 7. Conclusion

**PQ_FIBS.pine passes the Boolean Semantics Audit for Pine v6**:

1. ✅ All `var bool` declarations are explicitly initialized
2. ✅ All UDT boolean fields have explicit defaults
3. ✅ No boolean is assigned `na` anywhere
4. ✅ No `nz()` misuse on booleans
5. ✅ Lazy evaluation is already implemented at critical gates
6. ⚠️ Two comparison-derived booleans have theoretical NA_RISK but are contextually safe

**No code changes required** — the codebase already follows Pine v6 strict boolean semantics.
