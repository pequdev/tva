# PQ_FIBS Signal Gate Tree — Canonical Evaluation Order

> **Task 5 Deliverable**: One-Page Gate Dependency Map
> **Branch**: `50-pq_fibs-28-boolean-semantics-lazy-wiring`
> **Generated**: 2024-12-22

---

## Overview

This document maps the **complete signal generation pipeline** in `PQ_FIBS.pine`, showing:
- Gate dependencies and evaluation order
- Cheap vs expensive operations
- Where lazy evaluation is exploited
- "Do not move" boundaries (request constraints)

---

## Gate Tree Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         PQ_FIBS SIGNAL GATE TREE                                │
│                    (Evaluation Order: Top → Bottom)                             │
└─────────────────────────────────────────────────────────────────────────────────┘

╔═══════════════════════════════════════════════════════════════════════════════╗
║ LAYER 0: GLOBAL SCOPE REQUESTS (MUST RUN EVERY BAR)                           ║
║ ─────────────────────────────────────────────────────────────────────────────  ║
║ ⚡ EXPENSIVE — Cannot be gated, always execute                                 ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║ L2476  REQ_ltfHighs = request.security_lower_tf(high)     │ ⛔ DO NOT MOVE    ║
║ L2477  REQ_ltfLows  = request.security_lower_tf(low)      │ ⛔ DO NOT MOVE    ║
║ (2 base contexts)                                                              ║
╚═══════════════════════════════════════════════════════════════════════════════╝
                                     │
                                     ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ LAYER 1: DATA PREREQUISITES (Cheap Checks)                                    │
├───────────────────────────────────────────────────────────────────────────────┤
│ L1043  f_readyForRegime(window_bars)                                          │
│        └─► bar_index >= window_bars                              💚 CHEAP     │
│                                                                               │
│ L2918  Sufficient history for log returns                                     │
│        └─► close[1] > 0                                          💚 CHEAP     │
└───────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ LAYER 2: REGIME COMPUTATION (Runs Every Bar — Heavy)                          │
├───────────────────────────────────────────────────────────────────────────────┤
│ ⚡ EXPENSIVE — Statistical engines must run for history continuity            │
│                                                                               │
│ L2938-2967  ENTROPY ENGINE                                                    │
│             ├─► f_rollingEntropy() or SymbolicEntropy.update()   🔴 EXPENSIVE │
│             └─► Output: GLOBAL_regime.entropyOk                  💚 BOOL      │
│                                                                               │
│ L2982-3012  HURST ENGINE                                                      │
│             ├─► f_rollingHurst() or DyadicHurst.update()         🔴 EXPENSIVE │
│             └─► Output: GLOBAL_regime.hurstOk                    💚 BOOL      │
│                                                                               │
│ L3015-3021  Z-SCORE MOMENTUM                                                  │
│             ├─► f_rollingZScore()                                🟡 MODERATE  │
│             └─► Output: GLOBAL_regime.momentumOk                 💚 BOOL      │
└───────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ LAYER 3: EXTERNAL CONTEXT (Gated by Regime)                                   │
├───────────────────────────────────────────────────────────────────────────────┤
│ L3063  _extGated = INPUT_EXT_ENABLED and f_computeExtGate(...)                │
│        └─► Short-circuit: if !INPUT_EXT_ENABLED, skip f_computeExtGate        │
│                                                                  💚 CHEAP     │
│                                                                               │
│ L3066-3105  if _extGated and not budgetExceeded:                              │
│             ├─► f_parseSymbols()                                 🟡 MODERATE  │
│             ├─► Budget check: 2 + symCount > 40?                 💚 CHEAP     │
│             └─► f_reqExtClose/f_reqExtOHLCV (0-20 contexts)      🔴 EXPENSIVE │
│                                                                               │
│ L3107-3111  Output: GLOBAL_extContext.extOk                      💚 BOOL      │
│             └─► true if disabled OR (gated AND validCount > 0)                │
└───────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
╔═══════════════════════════════════════════════════════════════════════════════╗
║ LAYER 4: ZIGZAG PIVOT DETECTION (Confirmed Bar Logic)                         ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║ L3275-3350  ZigZag Engine processes pivotHigh/pivotLow                        ║
║             └─► Output: GLOBAL_zigzag.changed                    💚 BOOL      ║
║                                                                               ║
║ L3768-3769  ta.valuewhen() for RSI/Price at pivot                             ║
║             └─► Pre-computed for divergence detection            🟡 MODERATE  ║
╚═══════════════════════════════════════════════════════════════════════════════╝
                                     │
                                     ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ LAYER 5: SPAWN GATE (Lazy Evaluation — Cheap First)                           │
│ ═══════════════════════════════════════════════════════════════════════════   │
│ 🎯 PRIMARY SIGNAL GATE — This is the "trade or no trade" decision point       │
├───────────────────────────────────────────────────────────────────────────────┤
│ L3842  _spawnGatesPassed = (                                                  │
│            GLOBAL_zigzag.changed                           💚 1. cheap bool   │
│            and na(GLOBAL_backtest.current)                 💚 2. cheap na     │
│            and not na(GLOBAL_cachedPivots.iMidPivot)       💚 3. cheap na     │
│            and not na(GLOBAL_cachedPivots.pEndBase)        💚 4. cheap na     │
│        )                                                                      │
│                                                                               │
│ L3843  _regimeGatesPassed = (                                                 │
│            GLOBAL_regime.entropyOk                         💚 pre-computed    │
│            and GLOBAL_regime.hurstOk                       💚 pre-computed    │
│            and GLOBAL_regime.momentumOk                    💚 pre-computed    │
│            and GLOBAL_extContext.extOk                     💚 pre-computed    │
│        )                                                                      │
│                                                                               │
│ L3848  if _spawnGatesPassed and _regimeGatesPassed:                           │
│            └─► SPAWN NEW TRADE (BacktestTrade.new())       🟡 MODERATE        │
└───────────────────────────────────────────────────────────────────────────────┘
                                     │
         ┌───────────────────────────┴───────────────────────────┐
         │                                                       │
         ▼                                                       ▼
┌─────────────────────────────────┐       ┌─────────────────────────────────────┐
│ LAYER 6A: BACKTEST SIMULATION   │       │ LAYER 6B: LIVE TRADE STATE          │
│ (Historical Trade Processing)   │       │ (Visual Position Management)        │
├─────────────────────────────────┤       ├─────────────────────────────────────┤
│ L3872  skip_this_bar guard      │       │ L4015-4350  Position Rendering      │
│        └─► Newly spawned trades │       │             ├─► Zone calculation    │
│            skip first bar       │       │             ├─► SL/TP computation   │
│                                 │       │             ├─► Time decay          │
│ L3875-4000  Trade lifecycle:    │       │             └─► RSI divergence      │
│        ├─► PENDING → zone touch │       │                                     │
│        ├─► ACTIVE → MAE/MFE     │       │ L4355-4357  Divergence booleans     │
│        └─► EXIT → SL/TP hit     │       │             (⚠️ NA_RISK mitigated)  │
│                                 │       │                                     │
│ L3936-3939  Intrabar Resolution │       │                                     │
│        └─► f_resolveIntrabar()  │       │                                     │
│            🔴 EXPENSIVE (loop)  │       │                                     │
│            but ONLY when both   │       │                                     │
│            SL and TP hit same   │       │                                     │
│            bar (rare)           │       │                                     │
└─────────────────────────────────┘       └─────────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ LAYER 7: LEARNING ENGINE (Flag-Gated Lazy Execution)                          │
├───────────────────────────────────────────────────────────────────────────────┤
│ L4392  if INPUT_LEARNING_ENABLED                           💚 1. input toggle │
│           and GLOBAL_setupHistory.size() >= STATIC_LEARN_* 💚 2. sample count │
│           and GLOBAL_learning.needsRecalculation:          💚 3. dirty flag   │
│                                                                               │
│        └─► LEARNING LOOP (L4459-4680)                      🔴 EXPENSIVE       │
│            for idx = 0 to samples_to_check - 1                                │
│            ├─► Win rate calculation                                           │
│            ├─► Regime stratification                                          │
│            ├─► Sortino ratio                                                  │
│            └─► Monte Carlo (if enabled)                                       │
│                                                                               │
│ 🎯 LAZY EVAL: Loop only runs when needsRecalculation = true                   │
│    (set when new trade recorded, cleared after calculation)                   │
└───────────────────────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│ LAYER 8: VISUAL RENDERING (Last-Bar Gated)                                    │
├───────────────────────────────────────────────────────────────────────────────┤
│ L5365-5533  Debug Tables (only on barstate.islast)         💚 GATED           │
│             ├─► perf_shouldShowDebugOverlay(INPUT_ENTROPY_DEBUG)              │
│             ├─► perf_shouldShowDebugOverlay(INPUT_HURST_DEBUG)                │
│             ├─► perf_shouldShowDebugOverlay(INPUT_EXT_DEBUG)                  │
│             └─► perf_shouldShowDashboard()                                    │
│                                                                               │
│ L5189-5350  Position Visual Updates                        💚 GATED           │
│             └─► Only when needsRecreate or exists()                           │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## Gate Dependency Matrix

| Gate | Depends On | Blocks | Cost |
|------|------------|--------|------|
| `bar_index >= window` | — | Regime engines | 💚 Cheap |
| `entropyOk` | Entropy engine output | `_regimeGatesPassed` | 💚 Pre-computed |
| `hurstOk` | Hurst engine output | `_regimeGatesPassed` | 💚 Pre-computed |
| `momentumOk` | Z-score engine output | `_regimeGatesPassed` | 💚 Pre-computed |
| `extOk` | External context validation | `_regimeGatesPassed` | 💚 Pre-computed |
| `GLOBAL_zigzag.changed` | ZigZag pivot detection | `_spawnGatesPassed` | 💚 Cheap |
| `na(GLOBAL_backtest.current)` | No active simulation | `_spawnGatesPassed` | 💚 Cheap |
| `not na(cachedPivots.*)` | Valid pivot data | `_spawnGatesPassed` | 💚 Cheap |
| `_spawnGatesPassed` | 4 cheap checks | Trade spawn | 💚 Cheap |
| `_regimeGatesPassed` | 4 pre-computed bools | Trade spawn | 💚 Cheap |
| `needsRecalculation` | Trade outcome recorded | Learning loop | 💚 Cheap |
| `barstate.islast` | Last bar of chart | Debug rendering | 💚 Cheap |

---

## Critical "Do Not Move" Boundaries

### 1. Request Calls (Global Scope Required)

```pine
// ⛔ MUST remain at global scope — Pine v6 requirement
L2476  REQ_ltfHighs = request.security_lower_tf(...)
L2477  REQ_ltfLows  = request.security_lower_tf(...)

// ⛔ MUST remain at global scope for ta.valuewhen history
L3768  GLOBAL_rsi_at_pivot = ta.valuewhen(...)
L3769  GLOBAL_priceAtPivot = ta.valuewhen(...)
```

### 2. Regime Engine Calls (History Continuity)

```pine
// ⚠️ Must run every bar for internal state continuity
L2938-2967  Entropy engine
L2982-3012  Hurst engine
L3015-3021  Z-score engine
```

### 3. Pivot Change Detection (ta.change/ta.valuewhen)

```pine
// ⚠️ Must run every bar for proper change detection
L3245-3248  _pivotChange_* = ta.change(...)
L3251-3260  ta.valuewhen(_pivotChange_*, ...)
```

---

## Lazy Evaluation Summary

| Site | Pattern | Savings |
|------|---------|---------|
| L3063 | `INPUT_EXT_ENABLED and f_computeExtGate(...)` | Skip gate calc if disabled |
| L3066 | `if _extGated and not budgetExceeded` | Skip requests if not gated |
| L3842 | `changed and na(current) and not na(pivot)...` | Skip pivot checks if unchanged |
| L3848 | `_spawnGatesPassed and _regimeGatesPassed` | Skip spawn if data invalid |
| L4041 | `INPUT_USE_LEARNED_SL and ... and mult > 0` | Skip learned SL if disabled |
| L4073 | `INPUT_LEARN_TP and ... and size() >= min` | Skip learned TP if disabled |
| L4392 | `ENABLED and size() >= min and needsRecalculation` | Skip loop if not dirty |
| L5365+ | `perf_shouldShowDebugOverlay(...)` | Skip table on non-last bars |

---

## Signal Flow Summary

```
DATA → REGIME → CONTEXT → PIVOT → GATES → SPAWN → MANAGE → LEARN → RENDER
 ↑        ↑        ↑        ↑       ↑       ↑        ↑        ↑        ↑
 │        │        │        │       │       │        │        │        │
EVERY   EVERY   GATED   EVERY   EVERY   GATED    GATED    LAZY    LAST
 BAR     BAR    (ext)    BAR     BAR   (spawn)  (active)  (flag)   BAR
```

---

## Conclusion

The PQ_FIBS signal gate tree is **well-structured for lazy evaluation**:

1. **Cheap gates first**: All spawn gates use pre-computed booleans
2. **Expensive ops gated**: Learning loop uses `needsRecalculation` flag
3. **Rendering isolated**: Debug tables only run on `barstate.islast`
4. **Request boundaries preserved**: LTF arrays at global scope, never moved

**No reordering changes required** — current implementation is optimal.
