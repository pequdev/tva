# STATE_MACHINE_AUDIT.md

> **Task 8**: Learning/Preview + Backtest Engine (Determinism & Finalisation)  
> **Author**: Copilot Audit  
> **Date**: 23 December 2025  
> **File**: PQ_FIBS.pine  

---

## 1. Executive Summary

This audit documents the **trade lifecycle state machine** in PQ_FIBS.pine, proving:

1. **Single-entry semantics**: Trade spawn occurs exactly once per zigzag confirmation
2. **Monotonic state transitions**: States progress PENDING → ACTIVE → CLOSED (never backwards)
3. **Exactly-once close**: Archive and learning update occur at single call sites
4. **Dual-track isolation**: BacktestEngine and SetupState (Learning) track independently

---

## 2. Dual Tracking Architecture

PQ_FIBS maintains **two independent trade tracking systems**:

| System | UDT | Purpose | Lifecycle |
|--------|-----|---------|-----------|
| **BacktestEngine** | `BacktestTrade` | Simulation statistics, Smart SL | Archive to `history[]` on close |
| **Learning Engine** | `SetupState` | Pattern learning, metrics | Record to `GLOBAL_setupHistory` on zigzag change |

### 2.1 BacktestEngine (L2681-2738)
```
GLOBAL_backtest.current : BacktestTrade  // Active simulation
GLOBAL_backtest.history : array<BacktestTrade>  // Completed trades (bounded)
GLOBAL_backtest.stats   : BacktestStats  // Computed at barstate.islast
```

### 2.2 Learning Engine (L2741+)
```
GLOBAL_activeTrade : SetupState  // Current confirmed setup
GLOBAL_setupHistory : CircularBuffer<SetupRecord>  // Learning data (bounded)
GLOBAL_learning : LearningMetrics  // Computed from buffer
```

---

## 3. State Flags Architecture (L875-907)

### 3.1 Flag Definitions (Powers of 2)
```pine
STATIC_FLAG_IS_LONG        = 1      // Bit 0: Direction
STATIC_FLAG_ZONE_TOUCHED   = 2      // Bit 1: Entry occurred
STATIC_FLAG_SL_HIT         = 4      // Bit 2: Stop loss hit
STATIC_FLAG_TP_HIT         = 8      // Bit 3: Take profit hit
STATIC_FLAG_HAD_RSI_DIV    = 16     // Bit 4: RSI divergence present
STATIC_FLAG_TP_AGGRESSIVE  = 32     // Bit 5: TP mode selection
STATIC_FLAG_CONS_TP_HIT    = 64     // Bit 6: Conservative TP hit (analytics)
STATIC_FLAG_AGGR_TP_HIT    = 128    // Bit 7: Aggressive TP hit (analytics)
STATIC_FLAG_PENDING        = 256    // Bit 8: Awaiting zone touch
STATIC_FLAG_ACTIVE         = 512    // Bit 9: Zone touched, awaiting exit
STATIC_FLAG_CLOSED         = 1024   // Bit 10: Exit complete
STATIC_FLAG_WON            = 2048   // Bit 11: Trade was profitable
STATIC_FLAG_LTF_RESOLVED   = 4096   // Bit 12: Used LTF for outcome
```

### 3.2 Composite Masks
```pine
STATIC_MASK_OUTCOME   = 12    // SL_HIT | TP_HIT (bits 2-3)
STATIC_MASK_LIFECYCLE = 1792  // PENDING | ACTIVE | CLOSED (bits 8-10)
```

### 3.3 Invariant: Mutual Exclusion
- **Lifecycle flags** (PENDING/ACTIVE/CLOSED): Exactly one set at any time
- **Outcome flags** (SL_HIT/TP_HIT): Exactly one set when trade closes
- Enforced via `setLifecycle()` and `setOutcome()` methods (L2549-2557, L2653-2661)

---

## 4. State Machine Diagram

```
                              ┌─────────────────────────────────────────────────────────────┐
                              │                    BACKTEST ENGINE                           │
                              │                  (GLOBAL_backtest.current)                   │
                              └─────────────────────────────────────────────────────────────┘
                                                          │
    SPAWN                                                 │
    ────────────────────────────────────────────────────────────────────────────────────────
    Guards:                                               ▼
    • GLOBAL_zigzag.changed == true     ┌─────────────────────────────────────────────────┐
    • na(GLOBAL_backtest.current)       │              (1) PENDING                        │
    • skip_this_bar guards              │    Entry conditions not yet met                 │
    Line: L3854-3877                    │    Zone: defined, SL/TP: defined                │
                                        └─────────────────────────────────────────────────┘
                                                          │
    ACTIVATE                                              │ Zone touch detected
    ────────────────────────────────────────────────────────────────────────────────────────
    Guard:                                                ▼
    • Zone touch (high/low crosses)     ┌─────────────────────────────────────────────────┐
    • sim_pending == true               │              (2) ACTIVE                         │
    Line: L3890-3909                    │    Tracking MAE/MFE, awaiting exit              │
                                        │    Flag: ZONE_TOUCHED set                       │
                                        └─────────────────────────────────────────────────┘
                                                          │
    CLOSE                                                 │ SL or TP hit
    ────────────────────────────────────────────────────────────────────────────────────────
    Guards:                                               ▼
    • sl_hit or tp_hit                  ┌─────────────────────────────────────────────────┐
    Lines: L3933-3985                   │              (3) CLOSED                         │
                                        │    Outcome resolved (LTF or HTF heuristic)      │
                                        │    Flags: CLOSED + (SL_HIT xor TP_HIT) + WON?   │
                                        └─────────────────────────────────────────────────┘
                                                          │
    ARCHIVE                                               │ archiveTrade() called
    ────────────────────────────────────────────────────────────────────────────────────────
    Call site: L3979 (SINGLE)                             ▼
    Then: current := na (L3984)         ┌─────────────────────────────────────────────────┐
                                        │         Archived to history[]                   │
                                        │    Bounded array, evicts oldest at capacity     │
                                        └─────────────────────────────────────────────────┘
```

---

## 5. Trade Spawn Guards

### 5.1 BacktestEngine Spawn (L3854-3877)
```pine
// Primary spawn condition
if GLOBAL_zigzag.changed and na(GLOBAL_backtest.current)
    // Secondary regime gates (optional filtering)
    bool skip_regime = INPUT_ENTROPY_ENABLED and not (GLOBAL_regime.entropyRegime == ...)
    bool skip_hurst = INPUT_HURST_ENABLED and not (GLOBAL_regime.hurstRegime == ...)
    
    if not skip_regime and not skip_hurst
        GLOBAL_backtest.current := BacktestTrade.new(...)
        GLOBAL_backtest.current.setLifecycle(STATIC_FLAG_PENDING)
```

**Guard Invariant**: `na(GLOBAL_backtest.current)` ensures no active trade exists before spawn.

### 5.2 Learning Engine Spawn (L5140-5168)
```pine
if GLOBAL_zigzag.changed
    GLOBAL_activeTrade.setup_bar := GLOBAL_cachedPivots.iMidPivot
    GLOBAL_activeTrade.state_mask := new_state
    // ... reset all fields
```

**Key Difference**: Learning spawn happens on **every** zigzag change (no regime gates).

---

## 6. Close & Archive Analysis

### 6.1 Close Transition (L3933-3985)
```pine
if sl_hit or tp_hit
    GLOBAL_backtest.current.setLifecycle(STATIC_FLAG_CLOSED)
    GLOBAL_backtest.current.exit_bar := bar_index
    
    // Outcome resolution (LTF or HTF heuristic)
    // ... see INTRABAR_RESOLVER_BOUNDARY.md
    
    // Set outcome via mutual-exclusion method
    if trade_won
        GLOBAL_backtest.current.setOutcome(STATIC_FLAG_TP_HIT).setFlag(STATIC_FLAG_WON)
    else
        GLOBAL_backtest.current.setOutcome(STATIC_FLAG_SL_HIT)
    
    // SINGLE CALL SITE for archiveTrade
    GLOBAL_backtest.archiveTrade(GLOBAL_backtest.current)  // L3979
    
    // Clear current (prevents re-processing)
    GLOBAL_backtest.current := na  // L3984
```

### 6.2 Archive Method (L2699-2707)
```pine
method archiveTrade(BacktestEngine this, BacktestTrade trade) =>
    if not na(this.history)
        // Bounded push: evict oldest when at capacity
        if this.history.size() >= this.maxHistory
            this.history.shift()
        this.history.push(trade)
    this
```

**Idempotency**: Method is deterministic—same trade pushed twice would duplicate.  
**Protection**: `current := na` after archive prevents any re-invocation path.

---

## 7. Learning Update Analysis

### 7.1 Update Trigger (L5082-5138)
```pine
bool should_record = INPUT_LEARNING_ENABLED 
                 and GLOBAL_zigzag.changed 
                 and not na(GLOBAL_activeTrade.setup_bar)

if should_record
    // Compute SetupRecord from GLOBAL_activeTrade state
    SetupRecord completed_setup = SetupRecord.new(...)  // L5105
    
    // SINGLE CALL SITE for learning push
    GLOBAL_setupHistory.push(completed_setup)  // L5135
    
    // Set recalculation flag
    GLOBAL_learning.needsRecalculation := true  // L5138
```

### 7.2 Exactly-Once Guarantees

| Invariant | Mechanism | Line |
|-----------|-----------|------|
| Single push site | Only one `GLOBAL_setupHistory.push()` exists | L5135 |
| Single recalc flag site | Only one `needsRecalculation := true` exists | L5138 |
| Guard protection | `GLOBAL_zigzag.changed` is edge-detected (true for one bar only) | L5082 |
| State reset | After recording, `GLOBAL_activeTrade` is reset on same bar | L5140-5168 |

---

## 8. Double-Processing Prevention

### 8.1 BacktestEngine Protection
```
Guard 1: na(GLOBAL_backtest.current) required for spawn
Guard 2: current := na executed immediately after archive
Result:  No bar can both close an old trade and spawn a new trade in SAME execution path
```

### 8.2 Learning Engine Protection
```
Guard 1: should_record requires not na(setup_bar) (old trade exists)
Guard 2: Recording happens BEFORE reset (execution order)
Guard 3: GLOBAL_zigzag.changed is edge-detected, true for exactly one bar
Result:  Each zigzag change records old trade then resets—never skips, never doubles
```

### 8.3 Same-Bar Skip Logic (L3805-3815)
```pine
// Prevent double-processing on confirmation bar
bool skip_this_bar = GLOBAL_zigzag.changed
if not skip_this_bar
    // Normal trade management (PENDING → ACTIVE → CLOSED transitions)
```

---

## 9. State Transition Call Sites

### 9.1 setLifecycle() Calls
| Line | Context | Transition |
|------|---------|------------|
| L3877 | BacktestTrade spawn | → PENDING |
| L3906 | BacktestTrade zone touch | PENDING → ACTIVE |
| L3936 | BacktestTrade exit | ACTIVE → CLOSED |

### 9.2 setOutcome() Calls
| Line | Context | Outcome |
|------|---------|---------|
| L3974 | BacktestTrade TP won | TP_HIT + WON |
| L3976 | BacktestTrade SL loss | SL_HIT |
| L4217 | SetupState TP won (long) | TP_HIT + WON |
| L4219 | SetupState SL loss (long) | SL_HIT |
| L4221 | SetupState SL loss (long, tiebreak) | SL_HIT |
| L4223 | SetupState TP won (long, tiebreak) | TP_HIT + WON |
| L4243 | SetupState TP won (short) | TP_HIT + WON |
| L4245 | SetupState SL loss (short) | SL_HIT |
| L4247 | SetupState SL loss (short, tiebreak) | SL_HIT |
| L4249 | SetupState TP won (short, tiebreak) | TP_HIT + WON |

**Note**: SetupState (L4200+) tracks outcomes but does NOT archive—it serves the Learning Engine.

---

## 10. Verification Matrix

| Invariant | Status | Evidence |
|-----------|--------|----------|
| Single spawn per zigzag | ✅ PASS | `na(GLOBAL_backtest.current)` guard at L3854 |
| Monotonic lifecycle | ✅ PASS | `setLifecycle()` clears old + sets new atomically |
| Mutual-exclusive outcome | ✅ PASS | `setOutcome()` clears old + sets new atomically |
| Single archive call site | ✅ PASS | Only L3979 calls `archiveTrade()` |
| Single learning push site | ✅ PASS | Only L5135 calls `GLOBAL_setupHistory.push()` |
| Single recalc trigger | ✅ PASS | Only L5138 sets `needsRecalculation := true` |
| Double-close prevention | ✅ PASS | `current := na` at L3984 after archive |
| Bounded history | ✅ PASS | `archiveTrade()` evicts at capacity (L2702) |

---

## 11. Risks & Mitigations

### 11.1 Potential Risk: Projection Contamination
**Risk**: Projection data could be mistakenly recorded to learning.  
**Mitigation**: 
- Learning uses `GLOBAL_activeTrade` (confirmed only), never projection vars
- Guard: `not isProjection` on all tracking updates (L4196, L4261, L4269)

### 11.2 Potential Risk: Stale State on Regime Skip
**Risk**: If regime gates block spawn, old `GLOBAL_activeTrade` state persists.  
**Mitigation**: 
- Learning spawn at L5140 has no regime gates—always resets on zigzag change
- BacktestEngine and Learning are independent—one blocking doesn't affect the other

---

## 12. Recommendations

1. **LGTM**: State machine is well-structured with proper guards
2. **LGTM**: Exactly-once semantics verified for both archive and learning
3. **LGTM**: Bounded arrays prevent unbounded memory growth
4. **Documentation**: This audit serves as canonical reference for future modifications

---

## Appendix A: Code References

| Component | Line Range | Purpose |
|-----------|------------|---------|
| State Flags | L875-907 | Flag definitions and masks |
| f_resolveIntrabar | L922-960 | LTF chronological resolution |
| SetupRecord UDT | L2489-2553 | Learning data structure |
| BacktestTrade UDT | L2605-2657 | Simulation data structure |
| BacktestEngine UDT | L2681-2738 | Simulation manager |
| SetupState UDT | L2741-2810 | Live setup tracking |
| BacktestEngine loop | L3805-3990 | Spawn/manage/close logic |
| SetupState tracking | L4180-4290 | Zone/SL/TP/MAE tracking |
| Learning update | L5082-5168 | Record + reset logic |
