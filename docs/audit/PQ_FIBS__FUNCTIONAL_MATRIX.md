# PQ_FIBS.pine — Functional Matrix

> **Version**: 1.0  
> **Generated**: 2024-12-22  
> **Source**: `PQ_FIBS.pine` (5,590 lines)

---

## 1. Subsystem Inventory

| # | Subsystem | Lines | Inputs | Outputs | Frequency | Complexity | Failure Modes |
|---|-----------|-------|--------|---------|-----------|------------|---------------|
| 1 | **Constants & Theme** | L10-425 | — | Colors, styles | Compile-time | Low | None |
| 2 | **Input Declarations** | L511-865 | User UI | INPUT_* vars (211 inputs) | Compile-time | Low | Invalid input ranges |
| 3 | **Performance Gating** | L1043-1092 | Inputs, bar_index | bool flags | Every bar | Low | False negatives block features |
| 4 | **Entropy Engine** | L1095-1340 | close series | entropyNorm (0-1) | Gated bars | High | Division by zero if window=0 |
| 5 | **Hurst Engine** | L1339-1620 | close series | hurst (0-1) | Gated bars | High | log(0) if segment empty |
| 6 | **Z-Score Engine** | L1617-1700 | close series | zscore (-3 to 3) | Gated bars | Medium | StdDev = 0 edge case |
| 7 | **External Context** | L1661-1730 | symbol CSV | closes[] | Gated bars | Medium | Invalid symbol errors |
| 8 | **ZigZag Engine** | L1924-2130 (types), L3113-3205 (exec) | H/L, depth, dev | pivot coords | Every bar | Medium | Insufficient history |
| 9 | **Cached Pivots** | L3135-3200 | ZigZag output | indexed refs | Pivot change | Low | None |
| 10 | **Projection State** | L2354-2406 | ZigZag, H/L | tentative pivot | Every bar | Low | False positives |
| 11 | **Fib Level Calc** | L2409-2470 | Pivot coords | level prices | Pivot change | Low | None |
| 12 | **Statistical Position** | L3796-4010 | Pivots, Learning | TP/SL prices | Zone entry | Medium | Division by ATR=0 |
| 13 | **Learning Engine** | L4390-4750 | Setup history | adaptive params | Bar close | High | Empty history buffer |
| 14 | **Position Visual** | L4010-4250 | Setup state | drawing objects | Zone entry | Low | Object limit exceeded |
| 15 | **Backtest Engine** | L2600-2730 (types), L5050-5160 (integration) | Price action | win/loss stats | Trade close | Medium | Insufficient trades |
| 16 | **Alert Dispatch** | L5248-5360 | Alert state | alert() calls | Zone events | Low | Message too long |
| 17 | **Debug Overlays** | L5359-5530 | All state | table objects | Last bar | Low | Table cell limits |
| 18 | **Dashboard** | L5527-5590 | Perf counters | summary table | Last bar | Low | None |

---

## 2. Runtime Budget Analysis

### 2.1 Request Budget (40 max)

| Request Type | Count | Scope | Notes |
|--------------|-------|-------|-------|
| `request.security_lower_tf` (highs) | 1 | Global | Intrabar data |
| `request.security_lower_tf` (lows) | 1 | Global | Intrabar data |
| `request.security` (ext close) | 0-5 | Function | Per enabled symbol |
| `request.security` (ext OHLCV) | 0-5 | Function | Per enabled symbol |
| **Worst Case Total** | **12** | | Well under 40 limit |

### 2.2 Drawing Object Budget

| Object Type | Limit | Est. Usage | Worst Case | Safety Margin |
|-------------|-------|------------|------------|---------------|
| `line` | 500 | ~100 | 200 | 60% |
| `label` | 500 | ~50 | 150 | 70% |
| `box` | 500 | ~20 | 50 | 90% |
| `polyline` | 100 | ~5 | 20 | 80% |
| `table` | — | 5 | 5 | N/A |

### 2.3 Array Budget

| Array | Type | Max Size | Growth | Notes |
|-------|------|----------|--------|-------|
| `REQ_intrabarHighs` | `array<float>` | ~240 | Per bar | LTF intrabar data |
| `REQ_intrabarLows` | `array<float>` | ~240 | Per bar | LTF intrabar data |
| `GLOBAL_setupHistory.buffer` | `array<SetupRecord>` | 100 | Fixed | Circular, no growth |
| `BUF_fibLevels` | `array<FibLevel>` | 15 | Fixed | One per fib level |
| `GLOBAL_extContext.closes` | `array<float>` | 5 | Fixed | External symbols |
| `GLOBAL_zigzagBuffer.segments` | `array<chart.point>` | 1000 | Pruned | Oldest removed |
| `UI_linesBuffer` | `array<line>` | 500 | Pruned | Oldest deleted |
| `UI_labelsBuffer` | `array<label>` | 500 | Pruned | Oldest deleted |

### 2.4 Loop Budget

| Context | Loop Count | Typical Iterations | Worst Case | Notes |
|---------|------------|-------------------|------------|-------|
| Entropy (k-gram) | 3 | 16 each | 64 | Binary k-grams |
| Hurst (RS calc) | 4 | window/8 each | 100 | Dyadic segments |
| Fib levels | 1 | 15 | 15 | Fixed level count |
| Setup history scan | 1 | 100 | 100 | Circular buffer |
| External context | 1 | 5 | 5 | Max 5 symbols |
| ZigZag visual | 1 | 100 | 500 | Segment pruning |
| **Total per bar** | **~12** | **~250** | **~800** | Well under limits |

---

## 3. Complexity Hotspots

| Rank | Subsystem | Complexity Score | Factors |
|------|-----------|------------------|---------|
| 1 | **Learning Engine** | 🔴 High | Stats aggregation, multiple metrics, history scan |
| 2 | **Hurst Engine** | 🔴 High | Dyadic recursion, log calculations, R/S algorithm |
| 3 | **Entropy Engine** | 🟠 Medium-High | K-gram counting, rolling window, normalization |
| 4 | **Statistical Position** | 🟠 Medium | MAE/MFE tracking, adaptive SL/TP |
| 5 | **ZigZag Engine** | 🟡 Medium | State machine, deviation checks |
| 6 | **Backtest Engine** | 🟡 Medium | Trade lifecycle tracking |
| 7 | **External Context** | 🟡 Medium | Multi-symbol requests, gating |
| 8 | **Alert Dispatch** | 🟢 Low | Condition checks, JSON building |
| 9 | **Fib Level Calc** | 🟢 Low | Simple arithmetic |
| 10 | **Debug Overlays** | 🟢 Low | Table population |

---

## 4. Input/Output Matrix by Subsystem

### 4.1 Entropy Engine
| Input | Source | Output | Destination |
|-------|--------|--------|-------------|
| `close` | Price series | `entropyNorm` | Regime |
| `INPUT_ENTROPY_WINDOW` | User | `entropy` | Debug table |
| `INPUT_ENTROPY_KGRAM_ORDER` | User | `regime` | Dashboard |

### 4.2 Hurst Engine
| Input | Source | Output | Destination |
|-------|--------|--------|-------------|
| `close` | Price series | `hurst` | Regime |
| `INPUT_HURST_WINDOW` | User | `regime` | Learning |
| `INPUT_HURST_MODE` | User | | Dashboard |

### 4.3 ZigZag Engine
| Input | Source | Output | Destination |
|-------|--------|--------|-------------|
| `high`, `low` | Price series | `GLOBAL_zigzag.changed` | Fib calc |
| `INPUT_DEPTH` | User | `pLast`, `iPrevPivot` | Cached pivots |
| `INPUT_DEVIATION` | User | `isUp` | Projection |

### 4.4 Statistical Position
| Input | Source | Output | Destination |
|-------|--------|--------|-------------|
| Cached pivots | ZigZag | `entry_price` | Position visual |
| `GLOBAL_learning` | Learning | `sl_price` | Alerts |
| `INPUT_SL_MULT` | User | `tp_price` | Backtest |
| ATR | Built-in | `smartStopLoss` | Position visual |

### 4.5 Learning Engine
| Input | Source | Output | Destination |
|-------|--------|--------|-------------|
| `GLOBAL_setupHistory` | Setup records | `winRate` | Position calc |
| `GLOBAL_regime` | Regime | `stopLossMultiplier` | Risk sizing |
| Completed setups | Position tracking | `divergenceEdge` | Alerts |
| | | `confidence` | Dashboard |

### 4.6 Alert Dispatch
| Input | Source | Output | Destination |
|-------|--------|--------|-------------|
| `GLOBAL_alertState` | State aggregation | `alert()` calls | External |
| `GLOBAL_activeTrade` | Position tracking | JSON message | Webhooks |
| `GLOBAL_regime` | Regime | | |
| `GLOBAL_learning` | Learning | | |

---

## 5. Failure Mode Analysis

| Subsystem | Failure Mode | Trigger | Impact | Mitigation |
|-----------|--------------|---------|--------|------------|
| Entropy | Division by zero | window=0 | NaN propagation | Default window=50 |
| Hurst | log(0) | Empty segment | NaN propagation | Min segment check |
| Z-Score | StdDev=0 | Flat price | Infinite z-score | Default to 0 |
| External | Invalid symbol | Typo in CSV | Request error | Symbol validation |
| Learning | Empty history | No setups yet | NaN metrics | needsRecalculation flag |
| Position | ATR=0 | No volatility | Division error | Min ATR floor |
| ZigZag | Insufficient bars | bar_index < depth | No pivots | Bar count check |
| Backtest | No trades | Early chart | Empty stats | Trade count check |
| Drawing | Object limit | Many signals | Objects not drawn | Oldest-first pruning |
| Alert | Message >4KB | Large payload | Truncation | Field prioritization |

---

## 6. Performance Characteristics

### 6.1 Per-Bar Cost (Typical)

| Operation | Cost | Gated? | Notes |
|-----------|------|--------|-------|
| Input access | O(1) | No | Constant |
| ZigZag check | O(depth) | No | Always runs |
| Fib level calc | O(levels) | No | ~15 levels |
| Entropy calc | O(window) | Yes | ~50 iterations |
| Hurst calc | O(window*log) | Yes | ~100 iterations |
| Learning update | O(history) | Yes | ~100 iterations |
| Drawing | O(objects) | Yes | Per visible object |

### 6.2 Worst-Case Bar

| Scenario | Trigger | Impact | Mitigation |
|----------|---------|--------|------------|
| All engines fire | barstate.isconfirmed + heavy mode | ~2000 iterations | Performance presets |
| Many drawings | New pivot + full render | ~50 new objects | Object pruning |
| External requests | 5 symbols enabled | 10 request contexts | Max 5 symbols |
| Learning recalc | Setup completed | Full history scan | Circular buffer cap |

### 6.3 Memory Profile

| Component | Allocation | Lifetime | Growth |
|-----------|------------|----------|--------|
| UDT instances | ~20 | Script | None |
| Array buffers | ~10 | Script | Bounded |
| Drawing objects | ~500 max | Pruned | Bounded |
| Map instances | 3 | Script | Bounded |

---

## 7. Integration Points

### 7.1 External Integrations

| Integration | Protocol | Data Flow | Notes |
|-------------|----------|-----------|-------|
| TradingView Alerts | Webhook/JSON | Out | Full payload or simple |
| External Symbols | request.security | In | Up to 5 symbols |
| Chart Drawings | Native API | Out | Lines, boxes, labels, polylines |
| Tables | Native API | Out | Debug and dashboard |

### 7.2 Internal Coupling

| Subsystem A | Subsystem B | Coupling Type | Strength |
|-------------|-------------|---------------|----------|
| ZigZag → Fib Calc | Data | Strong | Direct dependency |
| Regime → Learning | Data | Medium | Modulation only |
| Learning → Position | Data | Medium | SL/TP adjustment |
| Position → Alerts | Data | Strong | Alert payload |
| All → Debug | Read | Weak | Display only |

---

## 8. Testing Scenarios

### 8.1 Unit Test Candidates

| Function | Inputs | Expected Output | Edge Cases |
|----------|--------|-----------------|------------|
| `f_shannonEntropy()` | [1,1,1,1] | 0 (no entropy) | Empty array |
| `f_shannonEntropy()` | [1,0,0,0,0,0,0,0] | 0.375 | Single element |
| `f_computeRS()` | Flat series | 0 | Zero range |
| `f_linregSlope()` | Trending up | Positive | Flat series |
| `f_betaMean()` | alpha=2, beta=2 | 0.5 | alpha/beta=0 |
| `f_calcDev()` | price=100, base=90 | 11.11% | Same price |

### 8.2 Integration Test Scenarios

| Scenario | Setup | Expected | Validation |
|----------|-------|----------|------------|
| New pivot | Price exceeds deviation | `changed=true` | ZigZag state update |
| Zone entry | Price enters GP | Setup spawned | `GLOBAL_activeTrade.active` |
| TP hit | Price reaches TP | Alert fired | `AlertState.takeProfitHit` |
| SL hit | Price reaches SL | Setup closed | Setup added to history |
| Regime shift | Entropy crosses threshold | Learning adjusts | Metrics updated |

### 8.3 Stress Test Scenarios

| Scenario | Conditions | Limits Tested |
|----------|------------|---------------|
| High frequency | 1s chart, fast moves | Bar processing time |
| Many pivots | Volatile market | Object count |
| All features | Everything enabled | Request budget |
| Long history | 5000 bars | Memory usage |

---

## 9. Refactoring Readiness

### 9.1 Safe to Refactor (Low Risk)

| Zone | Lines | Reason |
|------|-------|--------|
| JSON functions | L3490-3705 | Pure functions, no state |
| Debug tables | L5359-5530 | Isolated, read-only |
| Constants | L10-425 | No logic |
| Helper math | L867-1000 | Pure functions |

### 9.2 Refactor with Caution (Medium Risk)

| Zone | Lines | Reason |
|------|-------|--------|
| Fib rendering | L3744-3900 | Object lifecycle |
| Position visual | L4010-4250 | Stateful objects |
| Learning update | L4390-4750 | Complex state |

### 9.3 Do Not Refactor Without Full Testing (High Risk)

| Zone | Lines | Reason |
|------|-------|--------|
| ZigZag engine | L1924-2130 (types), L3113-3205 (exec) | Core state machine |
| Global requests | L2471-2480 | Scope-critical |
| Pivot caching | L3135-3200 | Signal integrity |
| Alert dispatch | L5248-5360 | External integration |

---

## 10. Documentation Gaps

| Area | Status | Action Needed |
|------|--------|---------------|
| Entropy algorithm | Partial | Add paper reference |
| Hurst calculation | Partial | Document dyadic modes |
| Learning adaptation | Minimal | Document update logic |
| Alert payload schema | Missing | Add JSON schema doc |
| External symbol format | Missing | Document CSV format |

---

*Document generated as part of Task 1: Codebase Index + Dependency Flow Map*
