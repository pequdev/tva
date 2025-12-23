# INTRABAR_RESOLVER_BOUNDARY.md

> **Task 8**: Learning/Preview + Backtest Engine (Determinism & Finalisation)  
> **Author**: Copilot Audit  
> **Date**: 23 December 2025  
> **File**: PQ_FIBS.pine  

---

## 1. Executive Summary

This document proves that the **Intrabar Resolver** (`f_resolveIntrabar`) is:

1. **Isolated**: Only called by BacktestEngine for outcome determination
2. **Pure Function**: No side effects, deterministic output for same inputs
3. **Non-Signal-Gating**: Never used to filter trade entry or spawn
4. **Graceful Fallback**: HTF heuristic used when LTF unavailable

---

## 2. Intrabar Resolver Overview

### 2.1 Purpose
When both SL and TP are hit within the **same HTF bar**, determine which was hit **first** using Lower Timeframe (LTF) price data for chronological ordering.

### 2.2 Data Source
```pine
// L2481-2482: LTF arrays populated via request.security_lower_tf
REQ_ltfHighs = request.security_lower_tf(syminfo.tickerid, "1", high)
REQ_ltfLows  = request.security_lower_tf(syminfo.tickerid, "1", low)
```

---

## 3. Function Signature & Implementation

### 3.1 Location: L922-960

```pine
f_resolveIntrabar(
    array<float> ltf_highs, 
    array<float> ltf_lows, 
    bool is_long, 
    float sl_price, 
    float tp_price, 
    bool conservative_tiebreak
) =>
```

### 3.2 Return Values
```pine
[int outcome, bool resolved, int hitIndex]
```

| Return | Type | Meaning |
|--------|------|---------|
| `outcome` | int | `1` = TP hit first, `-1` = SL hit first, `0` = unresolved |
| `resolved` | bool | `true` if chronological order determined |
| `hitIndex` | int | Index in LTF array where hit occurred |

### 3.3 Algorithm (Simplified)
```
FOR each LTF bar in chronological order:
    IF is_long:
        IF ltf_low <= sl_price: RETURN (SL first, -1)
        IF ltf_high >= tp_price: RETURN (TP first, +1)
    ELSE (short):
        IF ltf_high >= sl_price: RETURN (SL first, -1)
        IF ltf_low <= tp_price: RETURN (TP first, +1)
        
IF both hit in same LTF bar AND conservative_tiebreak:
    RETURN (SL first, -1)  // Capital preservation
```

---

## 4. Call Site Analysis

### 4.1 Single Call Site: L3951

```pine
// Context: BacktestEngine close logic (L3933-3985)
if sl_hit and tp_hit
    // Both TP and SL hit in same HTF bar - need to determine which came first
    if INPUT_INTRABAR_ENABLED and array.size(REQ_ltfHighs) > 0
        bool conservative = INPUT_INTRABAR_TIEBREAK == UI_OPT_conservative
        [ltf_outcome, ltf_resolved, hit_idx] = f_resolveIntrabar(
            REQ_ltfHighs, 
            REQ_ltfLows, 
            sim_is_long, 
            GLOBAL_backtest.current.sl_price, 
            GLOBAL_backtest.current.tp_price, 
            conservative
        )
        
        if ltf_resolved
            trade_won := ltf_outcome == 1
            used_ltf_resolution := true
            ltf_hit_idx := hit_idx
            GLOBAL_backtest.current.setFlag(STATIC_FLAG_LTF_RESOLVED)
        else
            // LTF data available but no decisive outcome - use HTF fallback
            float dist_to_sl = math.abs(open - GLOBAL_backtest.current.sl_price)
            float dist_to_tp = math.abs(open - GLOBAL_backtest.current.tp_price)
            trade_won := dist_to_tp < dist_to_sl * 0.5
    else
        // Intrabar disabled or no LTF data - use HTF heuristic
        float dist_to_sl = math.abs(open - GLOBAL_backtest.current.sl_price)
        float dist_to_tp = math.abs(open - GLOBAL_backtest.current.tp_price)
        trade_won := dist_to_tp < dist_to_sl * 0.5
```

### 4.2 Verification: No Other Callers

```bash
# grep result confirms single call site
$ grep -n "f_resolveIntrabar" PQ_FIBS.pine
922:f_resolveIntrabar(array<float> ltf_highs, ...) =>
3951:    [ltf_outcome, ltf_resolved, hit_idx] = f_resolveIntrabar(...)
```

---

## 5. Boundary Proof: Non-Signal-Gating

### 5.1 What the Resolver DOES
- Determines **outcome** (TP vs SL) when both levels hit in same bar
- Sets `STATIC_FLAG_LTF_RESOLVED` for analytics tracking
- Returns data used only for `trade_won` boolean

### 5.2 What the Resolver Does NOT Do
| Prohibited Usage | Verified Absent |
|------------------|-----------------|
| Gate trade spawn | ✅ Not used in spawn logic (L3854-3877) |
| Filter entry conditions | ✅ Not used in PENDING→ACTIVE (L3890-3909) |
| Modify SetupState | ✅ Not called anywhere for GLOBAL_activeTrade |
| Affect learning data | ✅ Learning uses SetupState outcomes, not resolver |
| Influence visual display | ✅ Only BacktestEngine consumes output |

### 5.3 Code Path Isolation

```
TRADE SPAWN (L3854-3877)
├── Guards: zigzag.changed, na(current), regime checks
├── Uses: is_long_setup, golden pocket levels
└── NEVER calls: f_resolveIntrabar ✅

ZONE ENTRY (L3890-3909)
├── Guards: zone touch detection (high/low vs zone boundaries)
├── Uses: sim_sl_price, sim_tp_price from spawn
└── NEVER calls: f_resolveIntrabar ✅

TRADE CLOSE (L3933-3985)
├── Guards: sl_hit or tp_hit (HTF level check)
├── Uses: f_resolveIntrabar ONLY when sl_hit AND tp_hit
└── Purpose: Outcome determination ONLY ✅
```

---

## 6. Fallback Behavior

### 6.1 HTF Heuristic (When LTF Unavailable)
```pine
// L3964-3967 and L3969-3971
float dist_to_sl = math.abs(open - GLOBAL_backtest.current.sl_price)
float dist_to_tp = math.abs(open - GLOBAL_backtest.current.tp_price)
trade_won := dist_to_tp < dist_to_sl * 0.5  // TP only wins if Open is 50%+ closer
```

**Conservative Default**: When ambiguous, assumes SL hit first (capital preservation).

### 6.2 Fallback Triggers
| Condition | Fallback Used |
|-----------|---------------|
| `INPUT_INTRABAR_ENABLED = false` | HTF heuristic |
| `array.size(REQ_ltfHighs) == 0` | HTF heuristic |
| LTF can't determine order | HTF heuristic |

---

## 7. Determinism Proof

### 7.1 Pure Function Properties
```
f_resolveIntrabar(A, B, C, D, E, F) → [X, Y, Z]

Where:
- Inputs are immutable (arrays passed by reference, not modified)
- No global state read or written
- No side effects
- Same inputs always produce same outputs
```

### 7.2 Input Determinism
- `REQ_ltfHighs`/`REQ_ltfLows`: Populated by `request.security_lower_tf()` once per bar
- `sl_price`/`tp_price`: Stored at spawn time, immutable during trade
- `is_long`: Stored at spawn time, immutable during trade
- `conservative_tiebreak`: User input, constant during execution

### 7.3 Output Determinism
- For any given HTF bar with same LTF data and prices, outcome is deterministic
- Historical replay produces identical results

---

## 8. Learning Engine Isolation

### 8.1 SetupState (GLOBAL_activeTrade) Outcome Detection
The Learning Engine tracks outcomes **independently** at L4200-4260:

```pine
// SetupState outcome detection (NO LTF resolver usage)
if not GLOBAL_activeTrade.hasAny(STATIC_MASK_OUTCOME)
    if sl_touched and tp_touched
        // Uses HTF heuristic ONLY (consistent with non-LTF path)
        float dist_to_sl = math.abs(open - GLOBAL_activeTrade.sl_price)
        float dist_to_tp = math.abs(open - GLOBAL_activeTrade.tp_price)
        if dist_to_tp < dist_to_sl * 0.5
            GLOBAL_activeTrade.setOutcome(STATIC_FLAG_TP_HIT).setFlag(STATIC_FLAG_WON)
        else
            GLOBAL_activeTrade.setOutcome(STATIC_FLAG_SL_HIT)
```

### 8.2 Reason for Separation
- **Learning Engine** uses consistent HTF heuristic for pattern learning
- **BacktestEngine** optionally uses LTF for higher simulation accuracy
- Keeps learning data comparable across different intrabar settings

---

## 9. Verification Matrix

| Invariant | Status | Evidence |
|-----------|--------|----------|
| Single call site | ✅ PASS | Only L3951 calls `f_resolveIntrabar` |
| Pure function | ✅ PASS | No global state access, no side effects |
| Non-signal-gating | ✅ PASS | Not used in spawn or entry logic |
| Graceful fallback | ✅ PASS | HTF heuristic when LTF unavailable |
| Conservative default | ✅ PASS | Assumes SL when ambiguous |
| Learning isolation | ✅ PASS | SetupState uses HTF only |
| Deterministic | ✅ PASS | Same inputs → same outputs |

---

## 10. Input Configuration

### 10.1 User Inputs (L335-340)
```pine
INPUT_INTRABAR_ENABLED = input.bool(true, "Enable LTF Resolution")
INPUT_INTRABAR_TIEBREAK = input.string("Conservative", "Tiebreak Mode", 
                                        options=["Conservative", "Aggressive"])
```

### 10.2 Behavior Matrix

| Intrabar Enabled | Tiebreak Mode | Both Hit Same Bar Behavior |
|------------------|---------------|---------------------------|
| true | Conservative | LTF resolution, SL on tie |
| true | Aggressive | LTF resolution, TP on tie |
| false | (ignored) | HTF heuristic only |

---

## 11. Risks & Mitigations

### 11.1 Risk: LTF Data Gaps
**Risk**: `request.security_lower_tf()` may return empty arrays on some bars.  
**Mitigation**: `array.size(REQ_ltfHighs) > 0` guard before calling resolver.

### 11.2 Risk: LTF/HTF Divergence
**Risk**: BacktestEngine and Learning Engine may produce different outcomes.  
**Mitigation**: This is by design—documented behavior, not a bug.

---

## 12. Recommendations

1. **LGTM**: Resolver is properly isolated and boundary-checked
2. **LGTM**: Fallback behavior is conservative and deterministic
3. **Documentation**: This audit serves as canonical reference

---

## Appendix A: Code References

| Component | Line Range | Purpose |
|-----------|------------|---------|
| f_resolveIntrabar | L922-960 | Chronological TP/SL resolution |
| REQ_ltfHighs/Lows | L2481-2482 | LTF data arrays |
| BacktestEngine close | L3933-3985 | Only consumer of resolver |
| SetupState outcome | L4200-4260 | Independent HTF-only detection |
| INPUT_INTRABAR_* | L335-340 | User configuration |
