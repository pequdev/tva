# PQ_FIBS.pine — Codebase Index

> **Version**: 1.0  
> **Generated**: 2024-12-22  
> **Source**: `PQ_FIBS.pine` (5,590 lines)  
> **Pine Version**: v6

---

## 1. Script Identity & Runtime Constraints

| Attribute | Value | Code Anchor |
|-----------|-------|-------------|
| Version | `//@version=6` | L1 |
| Indicator Title | `'Shox Fibonacci Pivots [PQ_MOD]'` | L8 |
| Short Title | `'FIBS/2.0'` | L8 |
| Overlay | `true` | L8 |
| max_lines_count | 500 | L8 |
| max_labels_count | 500 | L8 |
| max_boxes_count | 500 | L8 |
| max_polylines_count | 100 | L8 |
| max_bars_back | 5000 | L8 |
| dynamic_requests | `true` | L8 |
| calc_bars_count | 0 (all bars) | L8 |

### Runtime Limits (Documented in Header)
```
- Execution: 40s max, 500ms per bar loop
- Plots: 64 total
- Drawing objects: 500 line/box/label, 100 polyline
- Compiled tokens: 100k (libraries: 1M combined)
- Dynamic requests: 40 max per script
```

---

## 2. Section Index

| # | Section Name | Line Range | Responsibility |
|---|--------------|------------|----------------|
| 1 | Constants & Theme | L10-425 | UI colors, text tokens, style constants |
| 2 | UI Groups & Options | L426-453 | Input group definitions, option constants |
| 3 | Static Defaults | L450-570 | Default thresholds, window sizes, regime constants |
| 4 | Performance Control | L494-570 | Perf presets, degradation flags, static limits |
| 5 | Input Declarations | L511-865 | All user inputs (211 inputs across 22 groups) |
| 6 | Helper Functions (Math) | L867-1095 | Intrabar resolver, log returns, volatility, perf gates |
| 7 | Entropy Engine | L1095-1340 | Shannon entropy, symbolic entropy, rolling entropy |
| 8 | Hurst Engine | L1339-1620 | R/S analysis, dyadic Hurst, rescaled range |
| 9 | Z-Score Engine | L1617-1700 | Linear regression slope, z-score momentum |
| 10 | External Context | L1661-1730 | Symbol parsing, external requests, gating |
| 11 | Bayesian & Stats | L1727-1890 | Beta distribution, Sortino, permutation tests |
| 12 | Timeframe Maps | L1887-1925 | Deviation thresholds, depth maps per TF |
| 13 | Core UDTs | L1896-2470 | All type definitions (20 types) |
| 14 | Global Scope Requests | L2471-2480 | Intrabar LTF requests (must be global) |
| 15 | Buffers & State Init | L2479-2730 | Circular buffers, backtest engine, setup history |
| 16 | Alert State | L2814-2880 | AlertState type and populate method |
| 17 | Global State Init | L2880-2920 | GLOBAL_ var declarations |
| 18 | Debug Counters | L2896-2915 | DBG_ counter variables |
| 19 | Regime Computation | L2913-3040 | Entropy/Hurst/Z-score execution |
| 20 | External Context Exec | L3038-3115 | External symbol request loop |
| 21 | ZigZag Engine Exec | L3113-3205 | ZigZag runtime execution (types at L1924-2130) |
| 22 | Cached Pivots | L3135-3200 | CachedPivots type, pivot coordinate refs |
| 23 | HTF OHLC | L3203-3390 | HTF data fetching function |
| 24 | Pivot Coordinate Refs | L3388-3410 | GLOBAL_ pivot index/price assignments |
| 25 | Projection Integration | L3404-3495 | Projection pivot state |
| 26 | JSON Serialization | L3490-3705 | Alert message building functions |
| 27 | Fib Time Zones | L3704-3745 | Time zone drawing logic |
| 28 | Fib Level Drawing | L3744-3800 | FibLevel methods, level computation |
| 29 | Level Crossing Detection | L3764-3800 | Pre-computed crossing for all levels |
| 30 | Statistical Position Engine | L3796-4010 | Golden Pocket, TP/SL calculation |
| 31 | Position Visuals | L4010-4250 | PositionVisual rendering |
| 32 | Learning Engine | L4390-4750 | Adaptive learning, win rate, MAE/MFE |
| 33 | Display & Labels | L4940-5050 | Label text building, EV display |
| 34 | Backtest Integration | L5050-5160 | Setup recording, history management (types at L2600-2730) |
| 35 | Alert Dispatching | L5248-5360 | Alert condition checks, message firing |
| 36 | Debug Overlays | L5359-5530 | Entropy/Hurst/Ext debug tables |
| 37 | Performance Dashboard | L5527-5590 | Dashboard table on last bar |

---

## 3. Type Definitions Index

| Type Name | Line | Subsystem | Key Fields | Purpose |
|-----------|------|-----------|------------|---------|
| `SymbolicEntropyState` | L1164 | Stats/Entropy | `binaryBuffer`, `kgramCounts`, `entropy`, `entropyNorm` | Rolling entropy computation state |
| `DyadicHurstState` | L1450 | Stats/Hurst | `logReturns`, `hurst`, `regime` | Rolling Hurst exponent state |
| `LevelLog` | L1896 | Rendering | `levels` (map) | Fib level price logging |
| `PivotContext` | L1912 | Signals | `pMid`, `pEnd`, `isLong` | Pivot direction context |
| `ZigZagEngine` | L1924 | Core | `ln`, `pLast`, `iPrevPivot`, `changed` | ZigZag state machine |
| `ZigZagVisualBuffer` | L2035 | Rendering | `segments`, `active` (polyline), `liveLine` | ZigZag polyline buffer |
| `RegimeState` | L2131 | Regime | `entropy`, `hurst`, `zscore`, `atrPercentile`, `rsi` | Computed regime metrics |
| `ExternalContextState` | L2155 | Requests | `symbols`, `closes`, `enabled`, `gated` | External symbol data |
| `LearningMetrics` | L2187 | Learning | `winRate`, `stopLossMultiplier`, `divergenceEdge`, `confidence` | Learned parameters |
| `PositionVisual` | L2286 | Rendering | `boxEntry`, `lineSl`, `lineTp`, `lblRisk` | Position drawing objects |
| `ProjectionState` | L2354 | Signals | `active`, `isLong`, `tentativeIndex` | Early pivot projection |
| `FibLevel` | L2409 | Rendering | `level`, `col`, `ln`, `lb` | Individual fib level state |
| `SetupRecord` | L2484 | Learning | `entryBar`, `outcome`, `mae`, `mfe`, `barsHeld` | Historical setup data |
| `CircularBufferSetupRecord` | L2554 | Memory | `buffer`, `head`, `count` | O(1) circular buffer |
| `BacktestTrade` | L2600 | Backtest | `isLong`, `entryPrice`, `exitPrice`, `isWin` | Trade record |
| `BacktestStats` | L2655 | Backtest | `trades`, `winRate`, `profitFactor` | Aggregate stats |
| `BacktestEngine` | L2675 | Backtest | `stats`, `activeTrade` | Backtest state machine |
| `SetupState` | L2733 | Execution | `setup_bar`, `entry_price`, `sl_price`, `tp_price` | Active trade state |
| `AlertState` | L2817 | Alerts | `isProjection`, `isLong`, `stopLossHit`, `takeProfitHit` | Alert payload state |
| `CachedPivots` | L3135 | Core | `iMidPivot`, `pMidPivot`, `iEndBase`, `pEndBase` | Pivot coordinate cache |

---

## 4. Function Catalog

### 4.1 Performance & Gating Functions
| Function | Line | Subsystem | Purpose |
|----------|------|-----------|---------|
| `f_readyForRegime()` | L1043 | Perf | Check if enough bars for regime calc |
| `perf_shouldComputeHeavy()` | L1048 | Perf | Gate for heavy computations |
| `perf_shouldRenderDebug()` | L1051 | Perf | Gate for debug overlays |
| `perf_shouldShowDebugOverlay()` | L1054 | Perf | Explicit debug toggle check |
| `perf_shouldShowDashboard()` | L1057 | Perf | Dashboard toggle check |
| `perf_getEntropyWindow()` | L1060 | Perf | Get effective entropy window |
| `perf_getHurstWindow()` | L1063 | Perf | Get effective Hurst window |
| `f_canRunLearning()` | L1086 | Perf | Learning engine gate |
| `f_canComputeRegime()` | L1089 | Perf | Regime computation gate |
| `f_canSpawnSetup()` | L1092 | Perf | Setup spawn gate |

### 4.2 Statistics Engine Functions
| Function | Line | Subsystem | Purpose |
|----------|------|-----------|---------|
| `f_shannonEntropy()` | L1099 | Entropy | Compute Shannon entropy from bins |
| `f_rollingEntropy()` | L1111 | Entropy | Incremental entropy with state |
| `f_binaryState()` | L1243 | Entropy | Binary up/down classification |
| `f_computeRS()` | L1343 | Hurst | Rescaled range for one segment |
| `f_rollingHurst()` | L1375 | Hurst | Rolling Hurst with dyadic modes |
| `f_linregSlope()` | L1621 | Z-Score | Linear regression slope |
| `f_rollingZScore()` | L1640 | Z-Score | Z-score of normalized slope |

### 4.3 Math & Utility Functions
| Function | Line | Subsystem | Purpose |
|----------|------|-----------|---------|
| `f_resolveIntrabar()` | L919 | Intrabar | Resolve TP/SL order from LTF data |
| `f_logReturn()` | L961 | Math | Compute log return |
| `f_logToSimpleReturn()` | L967 | Math | Convert log to simple return |
| `f_getPeriodsPerYear()` | L970 | Math | Timeframe to periods/year |
| `f_annualizedVol()` | L993 | Math | Annualized volatility |
| `f_betaMean()` | L1703 | Bayesian | Beta distribution mean |
| `f_betaVariance()` | L1707 | Bayesian | Beta distribution variance |
| `f_betaLowerBound()` | L1714 | Bayesian | Beta credible interval |
| `f_sortino()` | L1731 | Stats | Sortino ratio calculation |
| `f_lcgNext()` | L1766 | RNG | Linear congruential generator |
| `f_fisherYatesShuffle()` | L1772 | Stats | Array shuffle for permutation test |
| `f_maxDrawdown()` | L1784 | Stats | Max drawdown from returns |
| `f_permutationTestDD()` | L1800 | Stats | Monte Carlo permutation test |
| `f_calcDev()` | L1947 | ZigZag | Calculate price deviation |

### 4.4 External Context Functions
| Function | Line | Subsystem | Purpose |
|----------|------|-----------|---------|
| `f_parseSymbols()` | L1665 | ExtContext | Parse CSV symbol list |
| `f_getExtTimeframe()` | L1682 | ExtContext | Resolve external timeframe |
| `f_computeExtGate()` | L1685 | ExtContext | Compute external gating |
| `f_reqExtClose()` | L1693 | ExtContext | Request external close |
| `f_reqExtOHLCV()` | L1696 | ExtContext | Request external OHLCV |

### 4.5 Rendering Functions
| Function | Line | Subsystem | Purpose |
|----------|------|-----------|---------|
| `f_drawLineTZ()` | L3437 | Rendering | Draw time zone line |
| `f_drawLabelTZ()` | L3441 | Rendering | Draw time zone label |
| `f_drawLinePVT()` | L3445 | Rendering | Draw pivot line |
| `f_drawLabelPVT()` | L3449 | Rendering | Draw pivot label |
| `f_crossingLevel()` | L3426 | Signals | Detect level crossing |
| `f_fibLevelKey()` | L2418 | Rendering | Generate fib level map key |

### 4.6 JSON & Alert Functions
| Function | Line | Subsystem | Purpose |
|----------|------|-----------|---------|
| `f_jsonFloat()` | L3494 | Alerts | Format float for JSON |
| `f_jsonInt()` | L3497 | Alerts | Format int for JSON |
| `f_jsonBool()` | L3500 | Alerts | Format bool for JSON |
| `f_jsonStr()` | L3503 | Alerts | Format string for JSON |
| `f_jsonEnvelope()` | L3507 | Alerts | Create JSON envelope |
| `f_jsonPivots()` | L3546 | Alerts | Serialize pivot data |
| `f_jsonSetup()` | L3550 | Alerts | Serialize setup state |
| `f_jsonZone()` | L3557 | Alerts | Serialize zone data |
| `f_jsonPosition()` | L3563 | Alerts | Serialize position data |
| `f_jsonTimeDecay()` | L3571 | Alerts | Serialize time decay |
| `f_jsonRegime()` | L3576 | Alerts | Serialize regime state |
| `f_jsonLearning()` | L3582 | Alerts | Serialize learning metrics |
| `f_jsonBacktest()` | L3586 | Alerts | Serialize backtest stats |
| `f_getAlertMessageFull()` | L3594 | Alerts | Build full alert message |
| `f_getAlertMessage()` | L3643 | Alerts | Build simple alert message |

### 4.7 State Management Functions
| Function | Line | Subsystem | Purpose |
|----------|------|-----------|---------|
| `f_htf_ohlc()` | L3359 | HTF | Get HTF OHLC values |
| `f_new_setup_state()` | L2793 | State | Create new SetupState instance |

---

## 5. Global State Variables (var/varip)

| Variable | Line | Type | Subsystem | Purpose |
|----------|------|------|-----------|---------|
| `STATE_tfDevThresholds` | L1832 | `map<string, float>` | Config | TF → deviation threshold |
| `STATE_tfDepths` | L1860 | `map<string, int>` | Config | TF → depth |
| `GLOBAL_pivotLevelsLogRetracementsMap` | L1892 | `map<float, float>` | Rendering | Fib retracement cache |
| `GLOBAL_pivotLevelsLogCrossedMap` | L1893 | `map<float, float>` | Rendering | Fib crossed cache |
| `GLOBAL_projection` | L2407 | `ProjectionState` | Signals | Projection state |
| `BUF_fibLevels` | L2423 | `array<FibLevel>` | Rendering | Fib level objects |
| `GLOBAL_pivotLevelsLogRetracements` | L2466 | `LevelLog` | Rendering | Retracement log |
| `GLOBAL_pivotLevelsLogCrossed` | L2467 | `LevelLog` | Rendering | Crossed log |
| `GLOBAL_setupHistory` | L2589 | `CircularBufferSetupRecord` | Learning | Setup history buffer |
| `GLOBAL_backtest` | L2727 | `BacktestEngine` | Backtest | Backtest state |
| `GLOBAL_alertState` | L2878 | `AlertState` | Alerts | Alert payload |
| `GLOBAL_activeTrade` | L2886 | `SetupState` | Execution | Active trade state |
| `GLOBAL_learning` | L2894 | `LearningMetrics` | Learning | Learned parameters |
| `DBG_counterEntropyRuns` | L2902 | `int` | Debug | Entropy call counter |
| `DBG_counterHurstRuns` | L2903 | `int` | Debug | Hurst call counter |
| `DBG_counterZscoreRuns` | L2904 | `int` | Debug | Z-score call counter |
| `DBG_counterLearningRuns` | L2905 | `int` | Debug | Learning call counter |
| `DBG_counterSpawnAttempts` | L2906 | `int` | Debug | Spawn attempt counter |
| `DBG_counterSpawnBlocked` | L2907 | `int` | Debug | Spawn blocked counter |
| `GLOBAL_symbolicEntropy` | L2919 | `SymbolicEntropyState` | Stats | Entropy engine state |
| `GLOBAL_dyadicHurst` | L2976 | `DyadicHurstState` | Stats | Hurst engine state |
| `GLOBAL_extContext` | L3046 | `ExternalContextState` | Requests | External context state |
| `GLOBAL_zigzag` | L3116 | `ZigZagEngine` | Core | ZigZag state machine |
| `GLOBAL_zigzagBuffer` | L3117 | `ZigZagVisualBuffer` | Rendering | ZigZag visual buffer |
| `GLOBAL_prevZzRender` | L3118 | `string` | Rendering | Previous render mode |
| `GLOBAL_cachedPivots` | L3171 | `CachedPivots` | Core | Cached pivot coords |
| `UI_linesBuffer` | L3200 | `array<line>` | Rendering | Line object buffer |
| `UI_labelsBuffer` | L3201 | `array<label>` | Rendering | Label object buffer |
| `GLOBAL_posVisual` | L4015 | `PositionVisual` | Rendering | Position visual objects |

---

## 6. Request Contexts

| Request Type | Line | Scope | Purpose | Budget Impact |
|--------------|------|-------|---------|---------------|
| `request.security_lower_tf` (high) | L2476 | Global | Intrabar highs | 1 context |
| `request.security_lower_tf` (low) | L2477 | Global | Intrabar lows | 1 context |
| `request.security` (ext close) | L1694 | Function | External symbol close | Per symbol |
| `request.security` (ext OHLCV) | L1697 | Function | External symbol OHLCV | Per symbol |

**Total Budget**: 2 base + up to 5 external = 7 contexts (well under 40 limit)

---

## 7. Drawing Object Creation Sites

| Object Type | Line | Context | Gating |
|-------------|------|---------|--------|
| `line.new` (ZigZag segment) | L1977 | ZigZag extend | `LINES` mode |
| `line.new` (ZigZag segment) | L2014 | ZigZag new | `LINES` mode |
| `polyline.new` (ZigZag) | L2082 | ZigZag visual | `POLYLINE` mode |
| `line.new` (live segment) | L2121 | ZigZag live | `INPUT_ZZ_LIVE_SEGMENT` |
| `box.new` (entry zone) | L2315 | Position visual | `INPUT_SHOW_POS` |
| `line.new` (SL) | L2316 | Position visual | `INPUT_SHOW_POS` |
| `line.new` (smart SL) | L2318 | Position visual | Smart SL enabled |
| `line.new` (TP) | L2319 | Position visual | `INPUT_SHOW_POS` |
| `label.new` (risk label) | L2320 | Position visual | `INPUT_SHOW_POS` |
| `line.new` (time zone) | L3439 | Fib time zones | `INPUT_SHOW_FIB_TIME` |
| `label.new` (time zone) | L3443 | Fib time zones | `INPUT_FIB_TZ_LABEL` |
| `line.new` (pivot) | L3447 | Fib levels | Level enabled |
| `label.new` (pivot) | L3451 | Fib levels | Level enabled |
| `line.new` (fib level) | L3468 | Fib level draw | Level enabled |
| `label.new` (fib level) | L3478 | Fib level draw | Level enabled |
| `table.new` (entropy debug) | L5366 | Debug | `INPUT_ENTROPY_DEBUG` |
| `table.new` (Hurst debug) | L5417 | Debug | `INPUT_HURST_DEBUG` |
| `table.new` (ext debug) | L5466 | Debug | `INPUT_EXT_DEBUG` |
| `table.new` (flow debug) | L5500 | Debug | Debug preset |
| `table.new` (dashboard) | L5534 | Debug | `INPUT_PERF_SHOW_DASHBOARD` |

---

## 8. Input Groups

### Main Feature Groups

| Group Constant | Purpose | Input Count |
|----------------|---------|-------------|
| `UI_GROUP_perfControl` | Performance presets | 5 |
| `UI_GROUP_pick` | Feature toggles | 1 |
| `UI_GROUP_pivot` | Pivot point settings | 6 |
| `UI_GROUP_fibTool` | Fib tool settings | 10 |
| `UI_GROUP_statPos` | Statistical position | 52 |
| `UI_GROUP_extContext` | External context | 9 |
| `UI_GROUP_fibLevels` | Fib level toggles | 66 |
| `UI_GROUP_zigzag` | ZigZag settings | 5 |
| `UI_GROUP_alerts` | Alert settings | 1 |
| `UI_GROUP_volVol` | Volume/volatility | 6 |

### Timeframe-Specific Threshold Groups

| Group Constant | Purpose | Input Count |
|----------------|---------|-------------|
| `UI_GROUP_threshSec` | Deviation thresholds (seconds TFs) | 7 |
| `UI_GROUP_threshMin` | Deviation thresholds (minutes TFs) | 8 |
| `UI_GROUP_threshHour` | Deviation thresholds (hours TFs) | 4 |
| `UI_GROUP_threshDay` | Deviation thresholds (daily TFs) | 2 |
| `UI_GROUP_threshWeek` | Deviation thresholds (weekly TFs) | 1 |
| `UI_GROUP_threshMonth` | Deviation thresholds (monthly TFs) | 3 |

### Timeframe-Specific Depth Groups

| Group Constant | Purpose | Input Count |
|----------------|---------|-------------|
| `UI_GROUP_depthSec` | Depth settings (seconds TFs) | 7 |
| `UI_GROUP_depthMin` | Depth settings (minutes TFs) | 8 |
| `UI_GROUP_depthHour` | Depth settings (hours TFs) | 4 |
| `UI_GROUP_depthDay` | Depth settings (daily TFs) | 2 |
| `UI_GROUP_depthWeek` | Depth settings (weekly TFs) | 1 |
| `UI_GROUP_depthMonth` | Depth settings (monthly TFs) | 3 |

### Summary

| Category | Input Count |
|----------|-------------|
| Main Feature Groups | 161 |
| Threshold Groups (TF-specific) | 25 |
| Depth Groups (TF-specific) | 25 |
| **Total Inputs** | **211** |

---

## 9. Critical Invariants

1. **Request Scope**: `request.security_lower_tf` MUST be at global scope (L2474-2477)
2. **Object Limits**: max 500 lines/labels/boxes, 100 polylines (L8)
3. **Request Budget**: 40 max dynamic requests (L6)
4. **Non-Repaint**: Intrabar resolver uses confirmed LTF data only
5. **Lazy Evaluation**: Heavy engines gated by `perf_shouldComputeHeavy()` and bar confirmation
6. **Circular Buffer**: O(1) push, fixed size prevents memory growth (L2554)
7. **Debug Isolation**: All debug tables use `table.new()` (no plot count impact)

---

## 10. Prefix Distribution

> **Note**: Counts are unique identifiers, not total occurrences.

| Prefix | Unique Identifiers | Purpose |
|--------|-------------------|---------|
| `GLOBAL_` | 43 | Cross-section state |
| `INPUT_` | 165 | User inputs |
| `STATIC_` | 185 | Compile-time constants |
| `STATE_` | 2 | Persistent var state |
| `UI_` | 292 | UI/rendering tokens |
| `REQ_` | 8 | Request contexts |
| `DBG_` | 6 | Debug counters |
| `TMP_` | 8 | Per-bar temporaries |
| `BUF_` | 1 | Array buffers |

---

*Document generated as part of Task 1: Codebase Index + Dependency Flow Map*
