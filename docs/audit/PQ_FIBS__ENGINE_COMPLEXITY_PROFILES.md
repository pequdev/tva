# PQ_FIBS Engine Complexity Profiles

> **Task 6 Deliverable**: Per-engine documentation of inputs, outputs, state, gates, and failure modes
> **Branch**: `51-pq_fibs-29-stats-engines-complexity-consolidation`
> **Generated**: 2024-12-23

---

## 1. Shannon Entropy Engine

### 1.1 Overview

| Property | Value |
|----------|-------|
| **Purpose** | Measures price action randomness to filter choppy markets |
| **UDT** | `SymbolicEntropyState` (L1164) |
| **Functions** | `f_shannonEntropy`, `f_rollingEntropy`, `SymbolicEntropyState.update` |
| **Global Instance** | `GLOBAL_symbolicEntropy` (L2919) |
| **Output Consumer** | `GLOBAL_regime.entropy*` fields → spawn gate |

### 1.2 Inputs

| Input | Type | Default | Range | Anchor |
|-------|------|---------|-------|--------|
| `INPUT_ENTROPY_ENABLED` | bool | true | — | L727 |
| `INPUT_ENTROPY_MODE` | string | "LEGACY_BINS" | LEGACY/BINARY/KGRAM | L728 |
| `INPUT_ENTROPY_WINDOW` | int | 20 | 5-100 | L728 |
| `INPUT_ENTROPY_BINS` | int | 5 | 3-10 | L729 |
| `INPUT_ENTROPY_K` | int | 3 | 1-8 | L730 |
| `INPUT_ENTROPY_THRESH` | float | 0.7 | 0.5-0.95 | L731 |
| `INPUT_ENTROPY_TRANSITION` | bool | true | — | L732 |

### 1.3 Outputs

| Field | Type | Range | Consumer | Anchor |
|-------|------|-------|----------|--------|
| `GLOBAL_regime.entropy` | float | 0 to log2(bins) | Debug display | L2963 |
| `GLOBAL_regime.entropyNorm` | float | 0.0 - 1.0 | Learning metrics | L2964 |
| `GLOBAL_regime.entropyRegime` | int | 0/1/2 | Spawn gate | L2965 |
| `GLOBAL_regime.entropyOk` | bool | true/false | Spawn gate | L2966 |

### 1.4 State / Buffers

| Buffer | Type | Size | Persistence | Anchor |
|--------|------|------|-------------|--------|
| `return_buffer` (legacy) | `array<float>` | window | `var` | L1112 |
| `bin_counts` (legacy) | `array<int>` | num_bins | `var` | L1113 |
| `patternBuf` (symbolic) | `array<int>` | window | UDT field | L1171 |
| `counts` (symbolic) | `array<int>` | states (2^k) | UDT field | L1172 |
| `cLog2c` (symbolic) | `array<float>` | window+1 | UDT field | L1173 |

### 1.5 Complexity Profile

| Mode | Complexity | Worst-Case Iterations | Notes |
|------|------------|----------------------|-------|
| Legacy | O(window × bins) | 100 × 10 = 1,000 | Per bar, binned log-return |
| Binary | O(1) | 1 | Incremental update |
| K-gram | O(1) | 1 | Incremental update |

### 1.6 Gating Conditions

```pine
// Effective mode/window from PerfController (Fast preset degrades)
string _effectiveEntropyMode = perf_getEntropyMode(INPUT_ENTROPY_MODE)
int _effectiveEntropyWindow = perf_getEntropyWindow(INPUT_ENTROPY_WINDOW)

// Execution gate (implicit - always runs for history building)
// Output gate:
GLOBAL_regime.entropyOk := not INPUT_ENTROPY_ENABLED or _entReg != STATIC_ENTROPY_regimeDisordered
```

### 1.7 Failure Modes

| Condition | Behavior | Fallback |
|-----------|----------|----------|
| `bar_index < window` | Returns `na` entropy | `entropyOk = true` (disabled filter) |
| All prices identical | `range_ret = 0` | All counts in middle bin |
| Invalid mode string | N/A (compile-time) | — |
| `na` log return | Skipped via `pushBounded` | Buffer may be short |

---

## 2. Hurst Exponent Engine

### 2.1 Overview

| Property | Value |
|----------|-------|
| **Purpose** | Measures market persistence/mean-reversion for regime filtering |
| **UDT** | `DyadicHurstState` (L1455) |
| **Functions** | `f_computeRS`, `f_rollingHurst`, `DyadicHurstState.update` |
| **Global Instance** | `GLOBAL_dyadicHurst` (L2977) |
| **Output Consumer** | `GLOBAL_regime.hurst*` fields → spawn gate |

### 2.2 Inputs

| Input | Type | Default | Range | Anchor |
|-------|------|---------|-------|--------|
| `INPUT_HURST_ENABLED` | bool | true | — | L735 |
| `INPUT_HURST_MODE` | string | "🔸 Legacy" | Legacy/Dyadic Stable/Dyadic Fast | L736 |
| `INPUT_HURST_WINDOW` | int | 100 | 20-500 | L737 |
| `INPUT_HURST_TREND` | float | 0.55 | 0.5-0.7 | L738 |
| `INPUT_HURST_MEANREV` | float | 0.45 | 0.3-0.5 | L739 |
| `INPUT_HURST_BLOCK_RANDOM` | bool | true | — | L740 |

### 2.3 Outputs

| Field | Type | Range | Consumer | Anchor |
|-------|------|-------|----------|--------|
| `GLOBAL_regime.hurst` | float | 0.0 - 1.0 | Learning, debug | L3009 |
| `GLOBAL_regime.hurstRegime` | int | 0/1/2 | Spawn gate | L3010 |
| `GLOBAL_regime.hurstOk` | bool | true/false | Spawn gate | L3011 |

### 2.4 State / Buffers

| Buffer | Type | Size | Persistence | Anchor |
|--------|------|------|-------------|--------|
| `return_buffer` (legacy) | `array<float>` | window | `var` | L1381 |
| `log_n` (legacy) | `array<float>` | scales | `var` | L1382 |
| `log_rs` (legacy) | `array<float>` | scales | `var` | L1383 |
| `returnBuf` (dyadic) | `array<float>` | window | UDT field | L1476 |
| `scales` (dyadic) | `array<int>` | log2(window) | UDT field | L1462 |
| `logScales` (dyadic) | `array<float>` | log2(window) | UDT field | L1465 |

### 2.5 Complexity Profile

| Mode | Complexity | Worst-Case Iterations | Notes |
|------|------------|----------------------|-------|
| Legacy | O(window × scales × chunks) | 100 × 4 × 5 = 2,000 | Per bar |
| Dyadic Stable | O(log(N) × chunks × scale) | 3 × 3 × 64 = 576 | Per bar |
| Dyadic Fast | O(log(N) × scale) | 3 × 64 = 192 | Per bar, single chunk |

### 2.6 Gating Conditions

```pine
// Effective mode/window from PerfController
string _effectiveHurstMode = perf_getHurstMode(INPUT_HURST_MODE)
int _effectiveHurstWindow = perf_getHurstWindow(INPUT_HURST_WINDOW)

// Output gate:
GLOBAL_regime.hurstOk := not INPUT_HURST_ENABLED or not INPUT_HURST_BLOCK_RANDOM or _hurstReg != STATIC_HURST_regimeRandom
```

### 2.7 Failure Modes

| Condition | Behavior | Fallback |
|-----------|----------|----------|
| `bar_index < minWindow` | Returns H=0.5, regime=RANDOM | `hurstOk = true` (disabled filter) |
| `stdev = 0` (constant series) | `f_computeRS` returns 0 | Excluded from regression |
| `< 2 valid scales` | Regression skipped | H=0.5 default |
| `na` log return | Skipped via `pushBounded` | Buffer may be short |

---

## 3. Z-Score Momentum Engine

### 3.1 Overview

| Property | Value |
|----------|-------|
| **Purpose** | Measures momentum strength via ATR-normalized regression slope |
| **Functions** | `f_linregSlope`, `f_rollingZScore` |
| **Output Consumer** | `GLOBAL_regime.zscore`, `momentumOk` → spawn gate |

### 3.2 Inputs

| Input | Type | Default | Range | Anchor |
|-------|------|---------|-------|--------|
| `INPUT_ZSCORE_ENABLED` | bool | true | — | L743 |
| `INPUT_ZSCORE_LR_LEN` | int | 14 | 5-50 | L744 |
| `INPUT_ZSCORE_ATR_LEN` | int | 14 | 5-50 | L745 |
| `INPUT_ZSCORE_Z_LEN` | int | 50 | 20-200 | L746 |
| `INPUT_ZSCORE_THRESH` | float | 2.0 | 1.0-4.0 | L747 |

### 3.3 Outputs

| Field | Type | Range | Consumer | Anchor |
|-------|------|-------|----------|--------|
| `GLOBAL_regime.zscoreAtr` | float | > 0 | Internal normalization | L3015 |
| `GLOBAL_regime.linregSlope` | float | unbounded | Internal | L3016 |
| `GLOBAL_regime.normSlope` | float | ATR-normalized | Z-score input | L3017 |
| `GLOBAL_regime.zscore` | float | unbounded | Debug, learning | L3019 |
| `GLOBAL_regime.strongMomentum` | bool | true/false | Spawn gate | L3020 |
| `GLOBAL_regime.momentumOk` | bool | true/false | Spawn gate | L3021 |

### 3.4 State / Buffers

| Buffer | Type | Size | Persistence | Anchor |
|--------|------|------|-------------|--------|
| `slope_buffer` | `array<float>` | z_len | `var` | L1648 |

### 3.5 Complexity Profile

| Component | Complexity | Worst-Case | Notes |
|-----------|------------|------------|-------|
| `f_linregSlope` | O(lr_len) | 50 | Per bar |
| `f_rollingZScore` | O(z_len) | 200 | Via `meanStdev` |
| `ta.atr` | O(1) (built-in) | — | Pine optimized |

**Total**: O(lr_len + z_len) ≈ 250 iterations max per bar

### 3.6 Gating Conditions

```pine
// Always runs (needed for history building for learning)
// No explicit execution gate - but output gate:
GLOBAL_regime.momentumOk := not INPUT_ZSCORE_ENABLED or _strongMom
```

### 3.7 Failure Modes

| Condition | Behavior | Fallback |
|-----------|----------|----------|
| `bar_index < lr_len` | Slope = 0 | Z-score = 0, not strong |
| `buf_size < min_samples` | Z-score = 0 | `strongMomentum = false` |
| `stdev = 0` | Division guarded | Z-score = 0 |
| `atr = 0` | Division guarded | normSlope = 0 |

---

## 4. Volatility Helpers

### 4.1 Overview

| Property | Value |
|----------|-------|
| **Purpose** | Log-return volatility, ATR, percentile ranking |
| **Functions** | `f_logReturn`, `f_annualizedVol`, `f_getPeriodsPerYear` |
| **Output Consumer** | Entropy/Hurst engines, learning, SL sizing |

### 4.2 Outputs

| Field | Type | Range | Consumer | Anchor |
|-------|------|-------|----------|--------|
| `GLOBAL_regime.logReturn` | float | unbounded | Entropy, Hurst input | L2922 |
| `GLOBAL_regime.logReturnStdev` | float | ≥ 0 | Annualized vol | L2923 |
| `GLOBAL_regime.periodsPerYear` | float | timeframe-dependent | Vol annualization | L2924 |
| `GLOBAL_regime.annualizedVol` | float | ≥ 0 | Display, learning | L2925 |
| `GLOBAL_regime.rawAtr` | float | ≥ 0 | SL sizing | L3024 |
| `GLOBAL_regime.atrPercentile` | float | 0-100 | Vol regime, SL scaling | L3025 |
| `GLOBAL_regime.rsi` | float | 0-100 | Divergence filter | L3026 |

### 4.3 Complexity Profile

All operations are O(1) or use Pine built-ins:
- `f_logReturn`: O(1) — single log computation
- `ta.stdev`: O(window) internally (Pine optimized)
- `ta.atr`: O(window) internally (Pine optimized)
- `ta.percentrank`: O(window) internally (Pine optimized)
- `ta.rsi`: O(window) internally (Pine optimized)

### 4.4 Gating Conditions

None — always computed. These are shared intermediates consumed by gated engines.

### 4.5 Failure Modes

| Condition | Behavior | Fallback |
|-----------|----------|----------|
| `price_prev = 0` | `f_logReturn` returns 0 | Safe division |
| `close[1] = na` | Returns 0 | No log computation |

---

## 5. Learning Engine

### 5.1 Overview

| Property | Value |
|----------|-------|
| **Purpose** | Backtested adaptive parameters (win rate, SL, buffer) |
| **UDT** | `LearningMetrics` (L2194) |
| **Global Instance** | `GLOBAL_learning` |
| **Output Consumer** | Position sizing, confidence score, TP mode |

### 5.2 Inputs

| Input | Type | Default | Range | Anchor |
|-------|------|---------|-------|--------|
| `INPUT_LEARNING_ENABLED` | bool | true | — | L708 |
| `INPUT_LEARNING_SAMPLES` | int | 50 | 10-500 | L709 |
| `INPUT_LEARN_MIN_SAMPLES` | int | 30 | 10-100 | L717 |
| `INPUT_BAYES_ENABLED` | bool | true | — | L749 |
| `INPUT_SORTINO_ENABLED` | bool | true | — | L753 |
| `INPUT_permtestEnabled` | bool | false | — | L754 |
| `INPUT_permtestSims` | int | 100 | 10-500 | L755 |

### 5.3 Outputs

| Field | Type | Consumer | Anchor |
|-------|------|----------|--------|
| `winRate` | float | Kelly, confidence | L2196 |
| `stopLossMultiplier` | float | SL calculation | L2197 |
| `zoneBuffer` | float | Zone expansion | L2195 |
| `bayesianLowerBound` | float | Conservative WR | L2242 |
| `sortinoRatio` | float | Risk-adjusted perf | L2252 |
| `confidence` | float | Display, filtering | L2266 |

### 5.4 Complexity Profile

| Component | Complexity | Frequency | Notes |
|-----------|------------|-----------|-------|
| Near-miss detection | O(samples) × 2 | Per bar (gated) | L4101, L4120 |
| Main learning loop | O(samples) | Last bar only | L4461 |
| Sortino loop | O(samples) | Last bar only | L4780 |
| Permtest | O(sims × samples) | Last bar only | L1815 |

**Worst case**: 500 samples × 500 sims = 250,000 iterations (permtest, last bar only)

### 5.5 Gating Conditions

```pine
// Primary gate: only on last bar with sufficient history
if INPUT_LEARNING_ENABLED and barstate.islast and GLOBAL_setupHistory.size() >= INPUT_LEARN_MIN_SAMPLES
    // Execute learning loops

// Near-miss detection: per bar but gated
if INPUT_LEARNING_ENABLED and GLOBAL_setupHistory.size() >= STATIC_LEARN_minSamplesBuffer
    // Near-miss loops
```

### 5.6 Failure Modes

| Condition | Behavior | Fallback |
|-----------|----------|----------|
| `history_size < min_samples` | Learning skipped | Use default metrics |
| No wins | `winRate = 0` | Health check fails |
| No losses | `profitFactor` capped at 10 | Valid but rare |
| `na` records | Skipped in loops | Reduced sample count |

---

## 6. External Context Engine

### 6.1 Overview

| Property | Value |
|----------|-------|
| **Purpose** | Fetch correlated asset data for regime context |
| **UDT** | `ExternalContextState` (L2155) |
| **Global Instance** | `GLOBAL_extContext` (L3046) |
| **Output Consumer** | `extOk` → spawn gate |

### 6.2 Complexity Profile

| Component | Complexity | Notes |
|-----------|------------|-------|
| Symbol parsing | O(symbols) | Max 20 |
| Request loop | O(symbols) | Max 20 |
| `request.security` | O(1) each | Pine handled |

### 6.3 Gating Conditions

```pine
bool _extGated = INPUT_EXT_ENABLED and f_computeExtGate(INPUT_EXT_GATE_MODE, ...)
// Only execute requests when gated
if _extGated and not GLOBAL_extContext.budgetExceeded
    // Request loop
```

### 6.4 Budget Controls

- Max symbols: 20 (`STATIC_EXT_maxSymbolsCap`)
- Max contexts: 40 (`STATIC_EXT_maxContexts`)
- Existing LTF requests: 2 (reserved)
- Available for external: 38

---

## 7. Performance Gate Functions

| Function | Purpose | Anchor |
|----------|---------|--------|
| `f_readyForRegime(window)` | Data sufficiency check | L1043 |
| `perf_shouldComputeHeavy(enabled, window)` | Enabled + confirmed + history | L1048 |
| `perf_shouldRenderDebug()` | Debug preset + last bar | L1051 |
| `perf_shouldShowDashboard()` | Dashboard toggle + last bar | L1057 |
| `perf_getEntropyWindow(user)` | Fast mode degradation | L1063 |
| `perf_getHurstWindow(user)` | Fast mode degradation | L1066 |
| `perf_getEntropyMode(user)` | Fast mode mode override | L1072 |
| `perf_getHurstMode(user)` | Fast mode mode override | L1069 |
| `f_canRunLearning(enabled, min, size)` | Learning execution gate | L1088 |
| `f_canComputeRegime(enabled, window)` | Regime computation gate | L1091 |

---

## 8. Engine Interaction Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            SHARED INTERMEDIATES                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  GLOBAL_regime.logReturn  ──┬──►  Entropy Engine (all modes)                │
│                             └──►  Hurst Engine (all modes)                   │
│                                                                              │
│  ta.atr(14)               ──┬──►  Z-Score normalization                     │
│                             └──►  SL sizing, volatility regime              │
│                                                                              │
│  ta.stdev(logReturn, 20)  ────►  Annualized volatility                      │
│                                                                              │
│  ta.rsi(close, 14)        ────►  Divergence filter, learning                │
└─────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              REGIME OUTPUTS                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  entropyOk  ────────────────┬──►  Spawn Gate (_regimeGatesPassed)           │
│  hurstOk    ────────────────┤                                                │
│  momentumOk ────────────────┤                                                │
│  extOk      ────────────────┘                                                │
│                                                                              │
│  entropyRegime  ────────────┬──►  Learning stratification                   │
│  hurstRegime    ────────────┤                                                │
│  strongMomentum ────────────┘                                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Summary Recommendations

1. **Default to O(1) modes**: Symbolic entropy + Dyadic Fast hurst for production
2. **Gate learning aggressively**: Already gated to `barstate.islast`
3. **Monitor permtest**: Disable by default, enable only for validation
4. **Shared intermediates**: `logReturn` and `ta.atr` already computed once per bar
5. **No code changes required**: Existing architecture is well-gated
