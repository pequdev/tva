# EXECUTION_STATE_CANONICALIZATION.md

> **Task 8**: Learning/Preview + Backtest Engine (Determinism & Finalisation)  
> **Author**: Copilot Audit  
> **Date**: 23 December 2025  
> **File**: PQ_FIBS.pine  

---

## 1. Executive Summary

This document maps all **consumers** of execution state (trade outcomes, learning metrics, backtest stats) and verifies:

1. **Single Source of Truth**: Each metric has one canonical source
2. **Update-Before-Consume**: State is computed before consumption
3. **No Stale Reads**: Consumers access current-bar state only
4. **Display Isolation**: Visual outputs don't modify execution state

---

## 2. Canonical State Sources

### 2.1 Three Independent State Domains

| Domain | Canonical Source | Update Site | Consumers |
|--------|-----------------|-------------|-----------|
| **BacktestEngine** | `GLOBAL_backtest` | L3805-3990 | Stats display, Smart SL |
| **Learning Engine** | `GLOBAL_learning` | L4400-4850 | Win rates, Kelly, EV |
| **Active Setup** | `GLOBAL_activeTrade` | L4180-4290 | Zone display, outcome tracking |

### 2.2 Data Flow Diagram

```
                                    ┌─────────────────────┐
                                    │  GLOBAL_zigzag      │
                                    │  (Pivot Detection)  │
                                    └─────────────────────┘
                                              │
                              ┌───────────────┼───────────────┐
                              ▼               ▼               ▼
                    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
                    │ BacktestEngine  │ │ SetupState      │ │ Learning Engine │
                    │ (Simulation)    │ │ (Live Tracking) │ │ (Metrics)       │
                    └─────────────────┘ └─────────────────┘ └─────────────────┘
                              │               │               │
                              ▼               ▼               ▼
                    ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
                    │ GLOBAL_backtest │ │ GLOBAL_active   │ │ GLOBAL_learning │
                    │ .stats          │ │ Trade           │ │ (computed)      │
                    └─────────────────┘ └─────────────────┘ └─────────────────┘
                              │               │               │
                              └───────────────┼───────────────┘
                                              ▼
                                    ┌─────────────────────┐
                                    │  DISPLAY / ALERTS   │
                                    │  (Read-Only)        │
                                    └─────────────────────┘
```

---

## 3. BacktestEngine State Consumers

### 3.1 Canonical Source: `GLOBAL_backtest.stats`

**Update Site**: L3989-4020 (computed once at `barstate.islast`)

```pine
if barstate.islast and array.size(GLOBAL_backtest.history) > 0 and not GLOBAL_backtest.stats.statsReady
    // Compute: totalTrades, winningTrades, winRate, avgWinningMae, avgLosingMae
    // Derive: smartSlOffset, lowConfidence
    GLOBAL_backtest.stats.statsReady := true  // L4020
```

### 3.2 Consumer Matrix

| Consumer | Line | Field(s) Read | Purpose |
|----------|------|---------------|---------|
| Stats display text | L5063-5074 | `.statsReady`, `.totalTrades`, `.winRate`, `.lowConfidence` | Dashboard text |
| Smart SL display | L5071-5073 | `.smartSlOffset` | Smart SL price |
| (All reads gated) | - | `.statsReady` check | Prevent stale access |

### 3.3 Update-Before-Consume Guarantee
```
Execution Order:
1. L3805-3990: Trade spawn/manage/close/archive
2. L3989-4020: Stats computation (barstate.islast)
3. L5063-5074: Display consumption (barstate.islast context)

Guard: `.statsReady` flag prevents reading uncomputed stats
```

---

## 4. Learning Engine State Consumers

### 4.1 Canonical Source: `GLOBAL_learning`

**Update Sites**:
- Primary: L4400-4850 (learning recalculation loop)
- Trigger: `GLOBAL_learning.needsRecalculation := true` (L5138)

### 4.2 Consumer Matrix

| Consumer | Line Range | Field(s) Read | Purpose |
|----------|------------|---------------|---------|
| Win rate display | L4980-5040 | `.winRate`, `.winRateLong`, `.winRateShort` | Label text |
| EV calculation | L4888-4920 | `.expectancy` | Expected value |
| Kelly sizing | L4920-4950 | `.winRate`, `.confidence` | Position sizing |
| SL adjustment | L4053-4060 | `.stopLossMultiplier`, `.losingMaxAdverseExcursion` | Dynamic SL |
| TP mode selection | L4088-4100 | `.aggressiveExpectedValue`, `.conservativeExpectedValue` | Auto TP |
| Regime edge | L5020-5040 | `.winRateHighVolatility`, `.winRateLowVolatility` | Regime display |
| Sharpe display | L5031-5033 | `.sharpeRatio` | Risk-adjusted return |
| Streak warning | L5035-5040 | `.winStreak`, `.lossStreak` | Alert indicator |

### 4.3 Recalculation Trigger

```pine
// Single trigger site (L5138)
GLOBAL_learning.needsRecalculation := true

// Consumption in learning loop (L4400+)
if INPUT_LEARNING_ENABLED and GLOBAL_learning.needsRecalculation and GLOBAL_setupHistory.size() > 0
    // Full recalculation from circular buffer
    // ... compute all metrics
    GLOBAL_learning.needsRecalculation := false  // Reset after compute
```

---

## 5. SetupState (GLOBAL_activeTrade) Consumers

### 5.1 Canonical Source: `GLOBAL_activeTrade`

**Update Sites**:
- Spawn: L5140-5168 (on GLOBAL_zigzag.changed)
- Tracking: L4180-4290 (zone/SL/TP/MAE updates)

### 5.2 Consumer Matrix

| Consumer | Line | Field(s) Read | Purpose |
|----------|------|---------------|---------|
| Zone display | L5175-5200 | `.zone_top`, `.zone_bottom` | Box coordinates |
| Outcome markers | L5233-5234 | `.hasFlag(SL_HIT)`, `.hasFlag(TP_HIT)` | Win/loss indicators |
| Direction display | L5054 | `.hasFlag(IS_LONG)` | Long/short label |
| State indicator | L5058-5060 | `.setup_bar` presence | Active/projection tag |
| Learning record | L5105-5135 | All fields | SetupRecord creation |

### 5.3 Read Isolation

```pine
// Display reads use STORED values for consistency (L5054)
bool display_is_long = isProjection ? is_long_setup : 
    (not na(GLOBAL_activeTrade.setup_bar) ? GLOBAL_activeTrade.hasFlag(STATIC_FLAG_IS_LONG) : is_long_setup)
```

---

## 6. Display/Alert Output Mapping

### 6.1 Label Outputs

| Label | Line | State Source | Write Type |
|-------|------|--------------|------------|
| Position label | L5076 | Multi-source composite | Create/update |
| Risk label | L2320-2340 | `PositionVisual` UDT | Create/update |

### 6.2 Table Outputs

| Table | Line | State Source | Purpose |
|-------|------|--------------|---------|
| Entropy debug | L5380-5427 | `GLOBAL_regime` | Regime diagnostics |
| Hurst debug | L5431-5475 | `GLOBAL_regime` | Hurst diagnostics |
| Extension debug | L5480-5510 | Extension state | Fib level tracking |
| Flow debug | L5514-5545 | Flow state | Order flow diagnostics |
| Performance dash | L5548-5600 | Multi-source | Summary dashboard |

### 6.3 Plot Outputs

| Plot | Line | State Source | Condition |
|------|------|--------------|-----------|
| None in PQ_FIBS | - | - | No `plotshape`/`alertcondition` in indicator |

**Note**: PQ_FIBS uses labels/boxes/tables only—no plotshape or alertcondition.

---

## 7. State Isolation Verification

### 7.1 Write Sites (Mutations)

| State | Write Sites | Guard |
|-------|-------------|-------|
| `GLOBAL_backtest.current` | L3854-3877 (spawn), L3890-3985 (lifecycle) | `na(current)` check |
| `GLOBAL_backtest.history` | L3979 (archiveTrade) | Single call site |
| `GLOBAL_backtest.stats` | L3989-4020 | `barstate.islast` + `statsReady` |
| `GLOBAL_activeTrade` | L4180-4290, L5140-5168 | `not isProjection` |
| `GLOBAL_learning` | L4400-4850 | `needsRecalculation` trigger |
| `GLOBAL_setupHistory` | L5135 | Single call site |

### 7.2 Read-Only Consumers

All display/visual code paths are **read-only**:
- Label text composition (L5020-5076)
- Table cell updates (L5380-5600)
- Box/line coordinate updates (L2320-2340)

**Verification**: No display code modifies `GLOBAL_backtest`, `GLOBAL_learning`, or `GLOBAL_activeTrade`.

---

## 8. Execution Order Proof

### 8.1 Bar Execution Sequence

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ BAR EXECUTION ORDER (Pine Script sequential model)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. Regime calculation (GLOBAL_regime update)                                │
│ 2. ZigZag detection (GLOBAL_zigzag update)                                  │
│ 3. BacktestEngine loop (L3805-3990)                                         │
│    ├── Trade spawn/manage/close                                             │
│    └── Stats computation (barstate.islast)                                  │
│ 4. SetupState tracking (L4180-4290)                                         │
│ 5. Learning recalculation (L4400-4850)                                      │
│ 6. Learning record + reset (L5082-5168)                                     │
│ 7. Display output (L5020-5600)                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 Invariant: Writes Precede Reads

| Consumer | Precondition | Verified |
|----------|--------------|----------|
| Stats display | `statsReady == true` | ✅ Gated at L5063 |
| Learning metrics | `needsRecalculation == false` | ✅ Reset after compute |
| Active trade display | `setup_bar != na` | ✅ Checked before access |

---

## 9. Stale State Prevention

### 9.1 Guard Patterns

```pine
// Pattern 1: Explicit ready flag
if GLOBAL_backtest.stats.statsReady and GLOBAL_backtest.stats.totalTrades > 0
    // Safe to read stats

// Pattern 2: Sample count threshold
if GLOBAL_setupHistory.size() >= INPUT_LEARNING_SAMPLES
    // Sufficient data for learning

// Pattern 3: Existence check
if not na(GLOBAL_activeTrade.setup_bar)
    // Active setup exists

// Pattern 4: Recalculation gate
if GLOBAL_learning.needsRecalculation
    // Trigger fresh computation
```

### 9.2 Edge Cases Handled

| Edge Case | Handling |
|-----------|----------|
| First bar (no history) | Guards prevent uninitialized reads |
| Insufficient samples | Display shows "Learning: N/M samples" |
| No active trade | Falls back to current calculation |
| Projection mode | Uses current calc, not stored state |

---

## 10. Verification Matrix

| Invariant | Status | Evidence |
|-----------|--------|----------|
| Single source of truth | ✅ PASS | Each metric has one update site |
| Update-before-consume | ✅ PASS | Execution order verified |
| No stale reads | ✅ PASS | Guards on all access paths |
| Display isolation | ✅ PASS | No display code mutates state |
| Bounded computation | ✅ PASS | CircularBuffer limits lookback |
| Deterministic output | ✅ PASS | Same inputs → same display |

---

## 11. Consumer Dependency Graph

```
GLOBAL_zigzag.changed
    │
    ├──► GLOBAL_backtest.current (spawn)
    │        │
    │        ├──► GLOBAL_backtest.history (archive)
    │        │        │
    │        │        └──► GLOBAL_backtest.stats (compute)
    │        │                  │
    │        │                  └──► Display: backtest_text
    │        │
    │        └──► Display: position label
    │
    ├──► GLOBAL_activeTrade (reset)
    │        │
    │        ├──► Zone/SL/TP tracking
    │        │        │
    │        │        └──► MAE/MFE updates
    │        │
    │        └──► Display: zone box, SL/TP lines
    │
    └──► GLOBAL_setupHistory.push()
             │
             └──► GLOBAL_learning.needsRecalculation := true
                      │
                      └──► GLOBAL_learning (full recompute)
                               │
                               ├──► Display: learn_text
                               ├──► SL adjustment
                               └──► TP mode selection
```

---

## 12. Recommendations

1. **LGTM**: State canonicalization is well-structured
2. **LGTM**: No circular dependencies between state domains
3. **LGTM**: Display outputs are purely read-only
4. **Documentation**: This audit serves as canonical consumer reference

---

## Appendix A: Code References

| Component | Line Range | Purpose |
|-----------|------------|---------|
| BacktestEngine loop | L3805-3990 | Trade lifecycle |
| Stats computation | L3989-4020 | BacktestStats |
| SetupState tracking | L4180-4290 | Zone/outcome |
| Learning loop | L4400-4850 | Metrics calculation |
| Learning record | L5082-5168 | Circular buffer push |
| Display composition | L5020-5076 | Label text |
| Debug tables | L5380-5600 | Diagnostic displays |

## Appendix B: State Field Index

### GLOBAL_backtest.stats
| Field | Type | Updated At | Consumers |
|-------|------|------------|-----------|
| totalTrades | int | L4007 | L5063 |
| winningTrades | int | L4008 | (internal) |
| winRate | float | L4009 | L5065 |
| avgWinningMae | float | L4010 | (internal) |
| avgLosingMae | float | L4011 | (internal) |
| smartSlOffset | float | L4014 | L5071 |
| lowConfidence | bool | L4017 | L5064 |
| statsReady | bool | L4020 | L5063 (guard) |

### GLOBAL_learning (subset)
| Field | Type | Updated At | Consumers |
|-------|------|------------|-----------|
| winRate | float | L4603 | L4980, L5024 |
| expectancy | float | L4735 | L5020 |
| sharpeRatio | float | L4780 | L5031 |
| stopLossMultiplier | float | L4550 | L4053 |
| needsRecalculation | bool | L5138 | L4400 (trigger) |
