# PQ_FIBS.pine — Dependency Flow Map

> **Version**: 1.0  
> **Generated**: 2024-12-22  
> **Source**: `PQ_FIBS.pine` (5,590 lines)

---

## 1. High-Level Data Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           INPUTS & CONSTANTS                                 │
│  (L10-865) INPUT_*, STATIC_*, UI_* constants and user inputs                │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          GLOBAL SCOPE REQUESTS                               │
│  (L2471-2480) REQ_intrabarHighs, REQ_intrabarLows                           │
│  ⚠ MUST remain at global scope - cannot be inside functions                │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          ▼                           ▼                           ▼
┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
│  ENTROPY ENGINE     │   │   HURST ENGINE      │   │  EXTERNAL CONTEXT   │
│  (L1095-1340)       │   │   (L1339-1620)      │   │  (L1661-1730)       │
│                     │   │                     │   │                     │
│  f_rollingEntropy() │   │  f_rollingHurst()   │   │  f_reqExtClose()    │
│  f_binaryState()    │   │  f_computeRS()      │   │  f_reqExtOHLCV()    │
│                     │   │                     │   │                     │
│  → entropyNorm      │   │  → hurst exponent   │   │  → ext closes[]     │
│  → regime signal    │   │  → regime signal    │   │  → gate signal      │
└─────────────────────┘   └─────────────────────┘   └─────────────────────┘
          │                           │                           │
          └───────────────────────────┼───────────────────────────┘
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            REGIME COMPUTATION                                │
│  (L2913-3040) RegimeState aggregation                                       │
│                                                                              │
│  Inputs:  entropy, hurst, zscore, ATR percentile, RSI                       │
│  Output:  GLOBAL_regime with all regime metrics                             │
│                                                                              │
│  Gated by: perf_shouldComputeHeavy(), f_readyForRegime()                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            ZIGZAG ENGINE                                     │
│  (L3113-3205) Runtime execution (types at L1924-2130)                       │
│                                                                              │
│  Inputs:  high, low, depth, deviation                                       │
│  Output:  GLOBAL_zigzag with pivot coordinates                              │
│           GLOBAL_cachedPivots with indexed references                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            PROJECTION STATE                                  │
│  (L3404-3495) Early pivot detection before confirmation                     │
│                                                                              │
│  Inputs:  GLOBAL_zigzag, deviation threshold, high/low                      │
│  Output:  GLOBAL_projection with tentative pivot                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          ▼                           ▼                           ▼
┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
│   FIB LEVEL CALC    │   │  STATISTICAL POS    │   │   LEARNING ENGINE   │
│   (L3744-3800)      │   │  (L3796-4010)       │   │   (L4390-4750)      │
│                     │   │                     │   │                     │
│  Retracement levels │   │  Golden Pocket      │   │  Win rate adapt     │
│  Extension levels   │   │  TP/SL targets      │   │  SL multiplier      │
│                     │   │  Entry zones        │   │  MAE/MFE tracking   │
│                     │   │                     │   │                     │
│  → BUF_fibLevels    │   │  → SetupState       │   │  → LearningMetrics  │
└─────────────────────┘   └─────────────────────┘   └─────────────────────┘
          │                           │                           │
          │                           │                           │
          ▼                           ▼                           ▼
┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
│   FIB RENDERING     │   │  POSITION VISUAL    │   │  BACKTEST ENGINE    │
│   (L3744-3900)      │   │  (L4010-4250)       │   │  (L5050-5160, types L2600-2730) │
│                     │   │                     │   │                     │
│  line.new for lvls  │   │  box.new entry      │   │  Trade recording    │
│  label.new for lbls │   │  line.new SL/TP     │   │  Win/loss tracking  │
│                     │   │                     │   │                     │
│  Gated: INPUT_SHOW  │   │  Gated: INPUT_POS   │   │  → BacktestStats    │
└─────────────────────┘   └─────────────────────┘   └─────────────────────┘
          │                           │                           │
          └───────────────────────────┼───────────────────────────┘
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            ALERT DISPATCHING                                 │
│  (L5248-5360) Alert condition checks and message firing                     │
│                                                                              │
│  Inputs:  AlertState (populated from all above)                             │
│  Output:  alert() calls with JSON payloads                                  │
│                                                                              │
│  Gated by: INPUT_ENABLE_ALERTS, zone entry/exit conditions                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            DEBUG OVERLAYS                                    │
│  (L5359-5590) Tables and dashboards                                         │
│                                                                              │
│  Gated by: perf_shouldRenderDebug(), INPUT_*_DEBUG flags                   │
│  ⚠ RENDER ONLY - NO SIGNAL MUTATION                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Subsystem Dependency Matrix

| Subsystem | Depends On | Provides To | Gating |
|-----------|-----------|-------------|--------|
| **Inputs** | — | All | Always |
| **Global Requests** | Inputs | Intrabar Resolver | Always (must be global) |
| **Entropy Engine** | Inputs, close | Regime | `perf_shouldComputeHeavy()` |
| **Hurst Engine** | Inputs, close | Regime | `perf_shouldComputeHeavy()` |
| **Z-Score Engine** | Inputs, close | Regime | `perf_shouldComputeHeavy()` |
| **External Context** | Inputs | Regime | `INPUT_EXTERNAL_CONTEXT_ENABLED` |
| **Regime Computation** | Entropy, Hurst, Z-Score, Ext | Learning, Alerts | `f_readyForRegime()` |
| **ZigZag Engine** | Inputs, H/L | Pivot Refs, Fib Calc | Always |
| **Cached Pivots** | ZigZag | Fib Calc, Position | Always |
| **Projection State** | ZigZag, H/L | Alerts, Rendering | Always |
| **Fib Level Calc** | Cached Pivots | Rendering | Always |
| **Statistical Position** | Cached Pivots, Learning | Position Visual, Alerts | `INPUT_SHOW_STAT_POS` |
| **Learning Engine** | Setup History, Regime | Position Calc, Backtest | `f_canRunLearning()` |
| **Position Visual** | Setup State | — | `INPUT_SHOW_POS` |
| **Backtest Engine** | Setup State, Price Action | Dashboard | `INPUT_BACKTEST_ENABLED` |
| **Alert Dispatch** | Alert State | — | `INPUT_ENABLE_ALERTS` |
| **Debug Overlays** | All State | — | Debug flags |

---

## 3. Critical Upstream/Downstream Chains

### 3.1 Pivot → Signal Chain (Core Path)
```
ZigZag Engine (pivot detection)
    │
    ├── GLOBAL_zigzag.changed (pivot confirmed)
    │       │
    │       ▼
    │   CachedPivots Update
    │       │
    │       ├── iMidPivot, pMidPivot (retracement anchor)
    │       └── iEndBase, pEndBase (extension anchor)
    │               │
    │               ▼
    │       Fib Level Calculation
    │           │
    │           ├── Retracement levels (0, 0.236, 0.382, 0.5, 0.618, 0.786, 1)
    │           └── Extension levels (1.272, 1.618, 2.618, 4.236)
    │                   │
    │                   ▼
    │           Golden Pocket Detection
    │               │
    │               └── Zone entry → SetupState spawn
    │
    └── GLOBAL_projection (early signal)
            │
            ▼
        Tentative pivot before confirmation
            │
            └── Projection alert (if enabled)
```

### 3.2 Regime → Learning Chain
```
Regime Computation
    │
    ├── GLOBAL_regime.entropy      ──┐
    ├── GLOBAL_regime.hurst        ──┼──► Regime Classification
    ├── GLOBAL_regime.zscore       ──┤    (trending/mean-revert/chaotic)
    └── GLOBAL_regime.atrPercentile──┘
                │
                ▼
        Learning Engine Modulation
            │
            ├── winRateHighVolatility  (regime-specific win rates)
            ├── winRateLowVolatility
            ├── stopLossMultiplier     (adaptive SL sizing)
            └── divergenceEdge         (regime-adjusted edge)
                    │
                    ▼
            Position Sizing / Risk Adjustment
```

### 3.3 Setup → Outcome Chain
```
Zone Entry Detection
    │
    ├── Price enters Golden Pocket
    │       │
    │       ▼
    │   SetupState Spawn (if f_canSpawnSetup() == true)
    │       │
    │       ├── GLOBAL_activeTrade created
    │       ├── entry_price, sl_price, tp_price set
    │       └── AlertState.isZoneEntry = true
    │               │
    │               ▼
    │       Position Tracking Loop
    │           │
    │           ├── MAE/MFE tracking per bar
    │           └── TP/SL hit detection
    │                   │
    │                   ├── TP hit → AlertState.takeProfitHit = true
    │                   └── SL hit → AlertState.stopLossHit = true
    │                           │
    │                           ▼
    │               SetupRecord Created
    │                   │
    │                   └── Push to GLOBAL_setupHistory (circular buffer)
    │                           │
    │                           ▼
    │                   Learning Metrics Update
    │                       │
    │                       └── Win rate, avg MAE/MFE recalculated
    │
    └── BacktestEngine Recording
            │
            └── BacktestStats updated
```

---

## 4. Gating Hierarchy

```
Level 0: Always Active
├── Input parsing
├── ZigZag pivot detection
├── Cached pivot updates
└── Fib level calculation

Level 1: Performance Gated
├── perf_shouldComputeHeavy()
│   ├── Entropy engine
│   ├── Hurst engine
│   └── Z-score engine
│
├── f_readyForRegime()
│   └── Regime aggregation
│
├── f_canRunLearning()
│   └── Learning metrics update
│
└── f_canSpawnSetup()
    └── New setup creation

Level 2: Feature Gated
├── INPUT_SHOW_STAT_POS
│   └── Statistical position rendering
│
├── INPUT_SHOW_POS
│   └── Position visual drawing
│
├── INPUT_EXTERNAL_CONTEXT_ENABLED
│   └── External symbol requests
│
├── INPUT_BACKTEST_ENABLED
│   └── Backtest engine
│
└── INPUT_ENABLE_ALERTS
    └── Alert dispatching

Level 3: Debug Gated
├── INPUT_ENTROPY_DEBUG
│   └── Entropy debug table
│
├── INPUT_HURST_DEBUG
│   └── Hurst debug table
│
├── INPUT_EXT_DEBUG
│   └── External context debug table
│
└── INPUT_PERF_SHOW_DASHBOARD
    └── Performance dashboard
```

---

## 5. Mutation Boundaries

### Read-Only After Computation
| Variable | Computed At | Readers | Never Modified After |
|----------|-------------|---------|---------------------|
| `GLOBAL_zigzag.changed` | ZigZag engine | Fib calc, Alerts | Pivot confirmation |
| `GLOBAL_cachedPivots.*` | Pivot caching | Fib calc, Position | Pivot update |
| `GLOBAL_regime.*` | Regime comp | Learning, Alerts | Bar close |
| `GLOBAL_learning.*` | Learning engine | Position calc | Bar close |

### State Machines (Controlled Mutation)
| State | Transitions | Trigger |
|-------|-------------|---------|
| `SetupState.active` | false → true → false | Zone entry → TP/SL hit |
| `GLOBAL_projection.active` | false ↔ true | Tentative pivot detection |
| `BacktestEngine.activeTrade` | na → trade → na | Trade open → Trade close |

---

## 6. Request Dependency Graph

```
GLOBAL SCOPE (L2471-2480)
│
├── request.security_lower_tf(syminfo.tickerid, REQ_ltf, high)
│   └── → REQ_intrabarHighs (array<float>)
│
├── request.security_lower_tf(syminfo.tickerid, REQ_ltf, low)
│   └── → REQ_intrabarLows (array<float>)
│
└── Used by: f_resolveIntrabar() → TP/SL order resolution

FUNCTION SCOPE (External Context)
│
├── f_reqExtClose(symbol)
│   └── request.security(symbol, tf, close)
│       └── → single float
│
└── f_reqExtOHLCV(symbol)
    └── request.security(symbol, tf, [o,h,l,c,v])
        └── → tuple<float, float, float, float, float>

⚠ CRITICAL: Global scope requests cannot be moved into functions
```

---

## 7. Rendering Isolation

### Principle: Rendering MUST NOT mutate signals

```
SIGNAL LAYER (Immutable by render time)
│
├── GLOBAL_zigzag
├── GLOBAL_cachedPivots
├── GLOBAL_regime
├── GLOBAL_learning
├── GLOBAL_alertState
└── GLOBAL_activeTrade

        │
        │ READ ONLY
        ▼

RENDER LAYER (Pure I/O)
│
├── Fib level drawing    → line.new, label.new
├── Position visual      → box.new, line.new, label.new
├── ZigZag visual        → polyline.new, line.new
├── Debug tables         → table.new, table.cell
└── Dashboard            → table.new, table.cell
```

### Render Functions MUST:
1. Only READ from GLOBAL_ state
2. Only WRITE to UI_* buffers and drawing objects
3. Never modify signal/setup/learning state
4. Be gated by INPUT_SHOW_* flags

---

## 8. Critical Invariants

1. **Request Scope Lock**: `request.security_lower_tf` MUST remain at global scope
2. **Signal Immutability**: Rendering code MUST NOT modify any GLOBAL_* signal state
3. **Gating Hierarchy**: Heavy computations MUST be gated by performance functions
4. **Circular Buffer Bound**: Setup history buffer size is fixed to prevent memory growth
5. **Alert Idempotence**: Alert state MUST be fully populated before dispatch
6. **Learning Lag**: Learning metrics use N-1 data (no look-ahead)

---

## 9. Potential Refactoring Zones

| Zone | Lines | Risk | Notes |
|------|-------|------|-------|
| Entropy Engine | L1095-1340 | Low | Self-contained, clear interface |
| Hurst Engine | L1339-1620 | Low | Self-contained, clear interface |
| JSON Functions | L3490-3705 | Low | Pure functions, no state |
| Fib Level Drawing | L3744-3900 | Medium | Many drawing calls, object limits |
| Debug Tables | L5359-5530 | Low | Isolated, feature-gated |

---

*Document generated as part of Task 1: Codebase Index + Dependency Flow Map*
