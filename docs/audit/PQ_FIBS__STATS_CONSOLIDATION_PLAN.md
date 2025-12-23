# PQ_FIBS Stats Consolidation Plan

> **Task 6 Deliverable**: Shared intermediate identification and behavior-preserving consolidation recommendations
> **Branch**: `51-pq_fibs-29-stats-engines-complexity-consolidation`
> **Generated**: 2024-12-23

---

## Executive Summary

This document analyzes potential shared intermediate computations across stats engines and documents consolidation opportunities. The goal is to eliminate redundant per-bar recomputation while preserving exact behavior.

**Key Findings**:
- ✅ **Log returns**: Already computed once and reused via `GLOBAL_regime.logReturn`
- ✅ **ATR**: Multiple calls exist but with different parameters — consolidation possible
- ✅ **RSI**: Computed once, stored in `GLOBAL_regime.rsi`
- ✅ **Stdev of log returns**: Computed once, stored in `GLOBAL_regime.logReturnStdev`
- ⚠️ **ATR duplication**: Two `ta.atr(INPUT_ATR_LENGTH)` calls identified — L2910 and L3818

**Consolidation Opportunity**: Minor — one ATR call can be eliminated.

---

## 1. Shared Intermediate Inventory

### 1.1 Log Return (Close-to-Close)

| Computation | Anchor | Frequency | Consumer |
|-------------|--------|-----------|----------|
| `f_logReturn(close, close[1])` | L2922 | Per bar | `GLOBAL_regime.logReturn` |

**Consumers**:
- Entropy Engine (all modes) — L2941, L2957
- Hurst Engine (all modes) — L2988, L3002
- Learning log-return analysis — L4747

**Status**: ✅ **Already consolidated** — computed once at L2922, reused everywhere.

---

### 1.2 ATR Computations

| Computation | Anchor | Parameters | Consumer |
|-------------|--------|------------|----------|
| `ta.atr(INPUT_ATR_LENGTH) * INPUT_ATR_MULT` | L2910 | Length from input | `i_weightedATR` (volatility addon) |
| `ta.atr(INPUT_ZSCORE_ATR_LEN)` | L3015 | Z-score length | `GLOBAL_regime.zscoreAtr` |
| `ta.atr(INPUT_ATR_LENGTH)` | L3818 | Same as L2910 | `raw_atr` (spawn logic) |
| `ta.atr(2)` | L1829 | Fixed 2 | `dev_treshold` function |

**Analysis**:
- L2910 and L3818 use the **same parameter** (`INPUT_ATR_LENGTH`)
- L3015 uses a **different parameter** (`INPUT_ZSCORE_ATR_LEN`) — cannot consolidate
- L1829 uses a **fixed value** (2) — different purpose, cannot consolidate

**Consolidation Opportunity**:
- L3818 `raw_atr = nz(ta.atr(INPUT_ATR_LENGTH), 1.0)` duplicates L2910 computation
- Can be replaced with: `raw_atr = nz(i_weightedATR / INPUT_ATR_MULT, 1.0)`
- Or: add `GLOBAL_regime.rawAtr := i_weightedATR / INPUT_ATR_MULT` once

**Current Implementation** (L3024):
```pine
GLOBAL_regime.rawAtr := i_weightedATR / INPUT_ATR_MULT
```

**Status**: ✅ **Already consolidated** — L3024 computes once, L4030 references `GLOBAL_regime.rawAtr`

**Remaining Issue**: L3818 still calls `ta.atr(INPUT_ATR_LENGTH)` directly instead of using `GLOBAL_regime.rawAtr`.

---

### 1.3 RSI

| Computation | Anchor | Parameters | Consumer |
|-------------|--------|------------|----------|
| `ta.rsi(close, INPUT_POS_RSI_LEN)` | L3027 | RSI length input | `GLOBAL_regime.rsi` |

**Consumers**:
- Divergence filter — L4351 (via `GLOBAL_regime.rsi`)
- Learning RSI divergence tracking — stores in SetupRecord

**Status**: ✅ **Already consolidated** — computed once at L3027.

---

### 1.4 Standard Deviation of Log Returns

| Computation | Anchor | Parameters | Consumer |
|-------------|--------|------------|----------|
| `ta.stdev(GLOBAL_regime.logReturn, STATIC_LOG_volWindow)` | L2923 | Window=20 | `GLOBAL_regime.logReturnStdev` |

**Consumers**:
- Annualized volatility calculation — L2925

**Status**: ✅ **Already consolidated** — computed once at L2923.

---

### 1.5 ATR Percentile

| Computation | Anchor | Parameters | Consumer |
|-------------|--------|------------|----------|
| `ta.percentrank(GLOBAL_regime.rawAtr, 100)` | L3025 | 100-bar ranking | `GLOBAL_regime.atrPercentile` |

**Consumers**:
- Dynamic SL scaling — L4034
- Learning volatility regime classification

**Status**: ✅ **Already consolidated** — computed once at L3025.

---

### 1.6 Periods Per Year

| Computation | Anchor | Frequency | Consumer |
|-------------|--------|-----------|----------|
| `f_getPeriodsPerYear()` | L2924 | Per bar | `GLOBAL_regime.periodsPerYear` |

**Status**: ✅ **Already consolidated** — computed once, could be optimized to `var` (timeframe-constant).

**Note**: `f_getPeriodsPerYear()` returns the same value every bar (based on `timeframe.*` which is constant). Could be optimized to `var` initialization, but impact is negligible (single switch statement).

---

## 2. Identified Redundancy

### 2.1 ATR at L3818

**Location**: L3818 (Spawn logic section)
```pine
float raw_atr = nz(ta.atr(INPUT_ATR_LENGTH), 1.0)
```

**Issue**: Duplicates computation already done at L2910 and stored in `GLOBAL_regime.rawAtr` at L3024.

**Recommendation**: Replace with reference to pre-computed value.

**Before**:
```pine
float raw_atr = nz(ta.atr(INPUT_ATR_LENGTH), 1.0)
```

**After**:
```pine
float raw_atr = nz(GLOBAL_regime.rawAtr, 1.0)
```

**Parity Argument**:
- `GLOBAL_regime.rawAtr` is computed at L3024 as `i_weightedATR / INPUT_ATR_MULT`
- `i_weightedATR` is `ta.atr(INPUT_ATR_LENGTH) * INPUT_ATR_MULT` (L2910)
- Therefore: `GLOBAL_regime.rawAtr = ta.atr(INPUT_ATR_LENGTH) * INPUT_ATR_MULT / INPUT_ATR_MULT = ta.atr(INPUT_ATR_LENGTH)`
- The `nz(..., 1.0)` fallback handles edge cases identically

**Behavior**: ✅ **IDENTICAL** — algebraically equivalent computation.

---

## 3. Gating Verification

All expensive engines have explicit gates aligned with Task 5 lazy evaluation:

| Engine | Gate | Cheap Check First | Status |
|--------|------|-------------------|--------|
| Entropy | `perf_shouldComputeHeavy(enabled, window)` | `enabled` bool | ✅ Lazy |
| Hurst | `perf_shouldComputeHeavy(enabled, window)` | `enabled` bool | ✅ Lazy |
| Z-Score | Always runs (cheap) | N/A | ✅ OK |
| Learning | `barstate.islast and size >= min` | `barstate.islast` | ✅ Lazy |
| Permtest | `INPUT_permtestEnabled and outcomes >= min` | `enabled` bool | ✅ Lazy |
| External | `INPUT_EXT_ENABLED and f_computeExtGate(...)` | `enabled` bool | ✅ Lazy |

---

## 4. Consolidation Actions

### 4.1 Required Changes (Behavior-Preserving)

| Priority | Change | Location | Impact | Parity |
|----------|--------|----------|--------|--------|
| LOW | Replace `ta.atr()` with `GLOBAL_regime.rawAtr` | L3818 | 1 less `ta.atr` call | ✅ Algebraically identical |

### 4.2 Optional Optimizations (Not Required)

| Priority | Change | Location | Impact | Parity |
|----------|--------|----------|--------|--------|
| VERY LOW | Make `periodsPerYear` a `var` | L2924 | Avoid per-bar switch | ✅ Same value |

---

## 5. Implementation

### 5.1 ATR Consolidation at L3818

**Current Code** (L3818):
```pine
float raw_atr = nz(ta.atr(INPUT_ATR_LENGTH), 1.0)
```

**Proposed Change**:
```pine
float raw_atr = nz(GLOBAL_regime.rawAtr, 1.0)
```

**Validation**:
1. `GLOBAL_regime.rawAtr` is assigned at L3024, which executes before L3818
2. Both use the same underlying ATR with same length parameter
3. `nz(..., 1.0)` fallback is preserved

### 5.2 Parity Proof

```
Given:
  L2910: i_weightedATR = ta.atr(INPUT_ATR_LENGTH) * INPUT_ATR_MULT
  L3024: GLOBAL_regime.rawAtr = i_weightedATR / INPUT_ATR_MULT

Therefore:
  GLOBAL_regime.rawAtr = ta.atr(INPUT_ATR_LENGTH) * INPUT_ATR_MULT / INPUT_ATR_MULT
                       = ta.atr(INPUT_ATR_LENGTH)

Which equals the original L3818 computation. ∎
```

---

## 6. Non-Consolidation Decisions

### 6.1 Z-Score ATR (Different Parameter)

| Computation | Anchor | Parameter |
|-------------|--------|-----------|
| `ta.atr(INPUT_ATR_LENGTH)` | L2910 | User-defined (default 14) |
| `ta.atr(INPUT_ZSCORE_ATR_LEN)` | L3015 | Z-score specific (default 14) |

**Decision**: Do NOT consolidate — these are intentionally separate inputs. Users may want different ATR periods for:
- SL sizing (price-based)
- Momentum normalization (velocity-based)

Even if defaults are the same, the semantic separation is correct.

### 6.2 Deviation Threshold ATR (Fixed Parameter)

| Computation | Anchor | Parameter |
|-------------|--------|-----------|
| `ta.atr(2)` | L1829 | Fixed 2-bar ATR |

**Decision**: Do NOT consolidate — this is a fundamentally different computation used for ZigZag deviation thresholds. The 2-bar ATR captures immediate volatility for pivot sensitivity, not trend volatility.

---

## 7. Summary

### 7.1 Current State Assessment

The codebase is **already well-consolidated**:
- Core shared intermediates (`logReturn`, `rawAtr`, `rsi`, `atrPercentile`) computed once
- Stored in `GLOBAL_regime` UDT for universal access
- Engines reuse these values rather than recomputing

### 7.2 Remaining Work

| Item | Status | Action |
|------|--------|--------|
| Log return consolidation | ✅ Done | None |
| ATR consolidation | ⚠️ 1 duplicate | Replace L3818 with reference |
| RSI consolidation | ✅ Done | None |
| Stdev consolidation | ✅ Done | None |
| Gating alignment | ✅ Done | None |

### 7.3 Consolidation Impact

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| `ta.atr()` calls | 4 | 3 | -25% (1 call) |
| Per-bar CPU | Baseline | Baseline - ε | Negligible |
| Code clarity | Good | Better | Reference over recompute |

---

## 8. Verification Checklist

- [ ] Compile succeeds after L3818 change
- [ ] No errors from `get_errors`
- [ ] `raw_atr` value unchanged (same ATR computation)
- [ ] Spawn logic behavior unchanged
- [ ] SL/TP calculations unchanged

---

## Appendix: Shared Intermediate Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         COMPUTATION ONCE (L2910-3027)                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ta.atr(INPUT_ATR_LENGTH) ──► i_weightedATR ──► GLOBAL_regime.rawAtr        │
│                                                     │                        │
│  ta.atr(INPUT_ZSCORE_ATR_LEN) ──────────────► GLOBAL_regime.zscoreAtr       │
│                                                     │                        │
│  f_logReturn(close, close[1]) ──────────────► GLOBAL_regime.logReturn       │
│                                                     │                        │
│  ta.stdev(logReturn, 20) ───────────────────► GLOBAL_regime.logReturnStdev  │
│                                                     │                        │
│  ta.rsi(close, 14) ─────────────────────────► GLOBAL_regime.rsi             │
│                                                     │                        │
│  ta.percentrank(rawAtr, 100) ───────────────► GLOBAL_regime.atrPercentile   │
│                                                     │                        │
└─────────────────────────────────────────────────────┼────────────────────────┘
                                                      │
                                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CONSUMERS                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Entropy Engine ◄───────────────────── logReturn                            │
│                                                                              │
│  Hurst Engine ◄─────────────────────── logReturn                            │
│                                                                              │
│  Z-Score Engine ◄───────────────────── zscoreAtr, close series              │
│                                                                              │
│  SL/TP Sizing ◄─────────────────────── rawAtr, atrPercentile                │
│                                                                              │
│  Divergence Filter ◄────────────────── rsi                                  │
│                                                                              │
│  Annualized Vol ◄───────────────────── logReturnStdev                       │
│                                                                              │
│  Spawn Logic (L3818) ◄──────────────── rawAtr (REDUNDANT CALL)              │
│                                        ↑                                     │
│                                        └── CONSOLIDATE THIS                  │
└─────────────────────────────────────────────────────────────────────────────┘
```
