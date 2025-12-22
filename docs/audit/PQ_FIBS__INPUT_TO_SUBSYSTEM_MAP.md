# PQ_FIBS.pine — Input to Subsystem Map

> **Version**: 1.0  
> **Generated**: 2025-12-22  
> **Source**: `PQ_FIBS.pine` (5,591 lines)  
> **Pine Version**: v6

---

## 1. Subsystem Classification

| Subsystem ID | Name | Primary Function |
|--------------|------|------------------|
| **PERF** | Performance Controller | Runtime gating, feature degradation |
| **ZZ** | ZigZag Engine | Pivot detection, visual segments |
| **FIB** | Fibonacci Rendering | Level drawing, time zones |
| **STAT** | Statistical Position | Entry zones, SL/TP, Kelly sizing |
| **LEARN** | Learning Engine | Adaptive parameters from history |
| **ENTROPY** | Entropy Regime | Disorder/order classification |
| **HURST** | Hurst Regime | Persistence/mean-reversion classification |
| **ZSCORE** | Momentum Z-Score | Velocity-based filtering |
| **BAYES** | Bayesian Stats | Win-rate estimation, Sortino |
| **PERM** | Permutation Test | Monte Carlo validation |
| **EXT** | External Context | Dynamic request orchestration |
| **ALERT** | Alert System | JSON serialization, notifications |
| **VOL** | Volume/Volatility | ATR bands, volume spikes |
| **INTRA** | Intrabar Resolver | LTF TP/SL resolution |

---

## 2. Input → Subsystem Mapping

### 2.1 Performance Controller (PERF)

| Input | Subsystem(s) | Consumer Functions/Variables | Risk Tag |
|-------|--------------|------------------------------|----------|
| `INPUT_PERF_PRESET` | PERF, ZZ, ENTROPY, HURST | `perf_*` functions, `STATIC_PERF_*` gates | PERF |
| `INPUT_PERF_SHOW_DASHBOARD` | PERF | Dashboard table render | PERF |
| `INPUT_PERF_DEGRADE_HURST` | HURST | `perf_getHurstWindow()`, `perf_getHurstMode()` | PERF |
| `INPUT_PERF_DEGRADE_ENTROPY` | ENTROPY | `perf_getEntropyWindow()` | PERF |
| `INPUT_PERF_SKIP_ALERTS` | ALERT | Alert JSON builder | PERF |

### 2.2 ZigZag Engine (ZZ)

| Input | Subsystem(s) | Consumer Functions/Variables | Risk Tag |
|-------|--------------|------------------------------|----------|
| `INPUT_SHOW_ZIGZAG` | ZZ, FIB | ZigZag render conditional | — |
| `INPUT_ZZ_RENDER_MODE` | ZZ | `GLOBAL_zigzag`, polyline vs lines | PERF |
| `INPUT_ZZ_POLY_MAX_KEEP` | ZZ | `ZigZagVisualBuffer.maxPolys` | SAFETY |
| `INPUT_ZZ_POLY_MAX_POINTS` | ZZ | `ZigZagVisualBuffer.maxPoints` | SAFETY |
| `INPUT_ZZ_LIVE_SEGMENT` | ZZ | Live line creation toggle | — |
| `INPUT_CUSTOM_THRESHOLD` | ZZ | `STATE_tfDevThresholds` usage | — |
| `INPUT_CUSTOM_DEPTH` | ZZ | `STATE_tfDepths` usage | — |
| Threshold inputs (×25) | ZZ | `STATE_tfDevThresholds` map | — |
| Depth inputs (×25) | ZZ | `STATE_tfDepths` map | SAFETY |

### 2.3 Fibonacci Rendering (FIB)

| Input | Subsystem(s) | Consumer Functions/Variables | Risk Tag |
|-------|--------------|------------------------------|----------|
| `INPUT_HTF_MODE` | FIB | `REQ_htf` timeframe selection | — |
| `INPUT_HTF_USER_RAW` | FIB | `REQ_htf` when user-defined | — |
| `INPUT_LEVELS_*` | FIB | Level label/position rendering | — |
| `INPUT_EXTEND_*` | FIB | Line extension behavior | — |
| `INPUT_HIST_PIVOT` | FIB | Historical pivot count | — |
| `INPUT_HIST_PIVOT_2` | FIB | Historical TZ count | — |
| `INPUT_FIB_TZ_*` | FIB | Time zone label positioning | — |
| `INPUT_SHOW_FIB_TIME` | FIB | Time zone toggle | — |
| `INPUT_REVERSE` | FIB | Level direction flip | — |
| Fib Level inputs (×66) | FIB | `BUF_fibLevels` array population | — |

### 2.4 Statistical Position (STAT)

| Input | Subsystem(s) | Consumer Functions/Variables | Risk Tag |
|-------|--------------|------------------------------|----------|
| `INPUT_SHOW_POS` | STAT | Position visual render toggle | — |
| `INPUT_POS_SL_MULT` | STAT | `GLOBAL_activeTrade.sl_price` | BEHAVIOR |
| `INPUT_POS_SL_DYNAMIC` | STAT | Dynamic SL scaling | BEHAVIOR |
| `INPUT_POS_RSI_DIV` | STAT | RSI divergence filter | BEHAVIOR |
| `INPUT_POS_RSI_LEN` | STAT | `ta.rsi()` length | SAFETY |
| `INPUT_POS_TIME_DECAY` | STAT | Zone decay calculation | BEHAVIOR |
| `INPUT_POS_TP_MODE` | STAT | TP target selection | BEHAVIOR |
| `INPUT_POS_WIN_RATE` | STAT, BAYES | Kelly calculation base | BEHAVIOR |
| `INPUT_POS_KELLY_FRAC` | STAT | Kelly fraction multiplier | BEHAVIOR |
| `INPUT_ZONE_BUFFER_BASE` | STAT, LEARN | Zone boundary expansion | BEHAVIOR |
| `INPUT_NEAR_MISS_THRESH` | STAT, LEARN | Near-miss detection | BEHAVIOR |
| `INPUT_PROJECTION_MODE` | STAT | Early pivot projection toggle | BEHAVIOR |
| `INPUT_PROJ_SENSITIVITY` | STAT | Projection threshold scalar | BEHAVIOR |

### 2.5 Learning Engine (LEARN)

| Input | Subsystem(s) | Consumer Functions/Variables | Risk Tag |
|-------|--------------|------------------------------|----------|
| `INPUT_LEARNING_ENABLED` | LEARN | Learning gate | PERF |
| `INPUT_LEARNING_SAMPLES` | LEARN | `GLOBAL_setupHistory.capacity` | SAFETY, PERF |
| `INPUT_USE_LEARNED_SL` | LEARN, STAT | Learned SL application | BEHAVIOR |
| `INPUT_LEARN_TP` | LEARN | TP mode learning | BEHAVIOR |
| `INPUT_LEARN_DECAY` | LEARN | Decay rate learning | BEHAVIOR |
| `INPUT_LEARN_RSI_WEIGHT` | LEARN | RSI weight learning | BEHAVIOR |
| `INPUT_LEARN_MIN_SAMPLES` | LEARN | Minimum samples for learning | SAFETY |

### 2.6 Entropy Regime (ENTROPY)

| Input | Subsystem(s) | Consumer Functions/Variables | Risk Tag |
|-------|--------------|------------------------------|----------|
| `INPUT_ENTROPY_ENABLED` | ENTROPY | Entropy gate | PERF |
| `INPUT_ENTROPY_MODE` | ENTROPY | Mode selection (Legacy/Binary/Kgram) | PERF |
| `INPUT_ENTROPY_WINDOW` | ENTROPY | Rolling window size | SAFETY, PERF |
| `INPUT_ENTROPY_BINS` | ENTROPY | Bin count (legacy mode) | SAFETY |
| `INPUT_ENTROPY_K` | ENTROPY | K-gram length | SAFETY |
| `INPUT_ENTROPY_THRESH` | ENTROPY | Regime threshold | BEHAVIOR |
| `INPUT_ENTROPY_TRANSITION` | ENTROPY | Transition detection | BEHAVIOR |
| `INPUT_ENTROPY_DEBUG` | ENTROPY, PERF | Debug overlay | PERF |

### 2.7 Hurst Regime (HURST)

| Input | Subsystem(s) | Consumer Functions/Variables | Risk Tag |
|-------|--------------|------------------------------|----------|
| `INPUT_HURST_ENABLED` | HURST | Hurst gate | PERF |
| `INPUT_HURST_MODE` | HURST | Mode selection (Legacy/Dyadic) | PERF |
| `INPUT_HURST_WINDOW` | HURST | Rolling window size | SAFETY, PERF |
| `INPUT_HURST_TREND` | HURST | Trend threshold | BEHAVIOR |
| `INPUT_HURST_MEANREV` | HURST | Mean-revert threshold | BEHAVIOR |
| `INPUT_HURST_BLOCK_RANDOM` | HURST | Random regime blocking | BEHAVIOR |
| `INPUT_HURST_DEBUG` | HURST, PERF | Debug overlay | PERF |

### 2.8 Momentum Z-Score (ZSCORE)

| Input | Subsystem(s) | Consumer Functions/Variables | Risk Tag |
|-------|--------------|------------------------------|----------|
| `INPUT_ZSCORE_ENABLED` | ZSCORE | Z-score gate | PERF |
| `INPUT_ZSCORE_LR_LEN` | ZSCORE | Linear regression length | SAFETY |
| `INPUT_ZSCORE_ATR_LEN` | ZSCORE | ATR normalization length | SAFETY |
| `INPUT_ZSCORE_Z_LEN` | ZSCORE | Z-score window | SAFETY |
| `INPUT_ZSCORE_THRESH` | ZSCORE | Significance threshold | BEHAVIOR |

### 2.9 Bayesian Stats (BAYES)

| Input | Subsystem(s) | Consumer Functions/Variables | Risk Tag |
|-------|--------------|------------------------------|----------|
| `INPUT_BAYES_ENABLED` | BAYES | Bayesian mode toggle | — |
| `INPUT_BAYES_ALPHA_PRIOR` | BAYES | Beta prior α | BEHAVIOR |
| `INPUT_BAYES_BETA_PRIOR` | BAYES | Beta prior β | BEHAVIOR |
| `INPUT_BAYES_CONFIDENCE` | BAYES | Credible interval | BEHAVIOR |
| `INPUT_SORTINO_ENABLED` | BAYES | Sortino calculation toggle | — |
| `INPUT_SORTINO_MAR` | BAYES | Minimum acceptable return | BEHAVIOR |

### 2.10 Permutation Test (PERM)

| Input | Subsystem(s) | Consumer Functions/Variables | Risk Tag |
|-------|--------------|------------------------------|----------|
| `INPUT_permtestEnabled` | PERM | Permutation test toggle | PERF |
| `INPUT_permtestSims` | PERM | Simulation count | PERF, SAFETY |
| `INPUT_permtestPvalue` | PERM | Significance threshold | BEHAVIOR |

### 2.11 External Context (EXT)

| Input | Subsystem(s) | Consumer Functions/Variables | Risk Tag |
|-------|--------------|------------------------------|----------|
| `INPUT_EXT_ENABLED` | EXT | External request gate | PERF |
| `INPUT_EXT_TF_MODE` | EXT | Timeframe selection mode | — |
| `INPUT_EXT_TF_CUSTOM` | EXT | Custom timeframe string | — |
| `INPUT_EXT_SYMBOLS` | EXT | CSV symbol list | SAFETY |
| `INPUT_EXT_MAX_SYMBOLS` | EXT | Request budget cap | PERF, SAFETY |
| `INPUT_EXT_FIELDS` | EXT | OHLCV vs Close mode | PERF |
| `INPUT_EXT_GATE_MODE` | EXT | Conditional request gating | PERF |
| `INPUT_EXT_DEBUG` | EXT, PERF | Debug overlay | PERF |
| `INPUT_DEBUG_FLOW` | PERF | Execution flow debug | PERF |

### 2.12 Alert System (ALERT)

| Input | Subsystem(s) | Consumer Functions/Variables | Risk Tag |
|-------|--------------|------------------------------|----------|
| `INPUT_ALERT_ENABLED` | ALERT | Alert trigger gate | — |
| `INPUT_ALERT_MODE` | ALERT | Data collection mode | — |
| `INPUT_ALERT_SEPARATOR` | ALERT | CSV separator | — |

### 2.13 Volume/Volatility (VOL)

| Input | Subsystem(s) | Consumer Functions/Variables | Risk Tag |
|-------|--------------|------------------------------|----------|
| `INPUT_SHOW_HIGH_ATR` | VOL | ATR band toggle | — |
| `INPUT_ATR_LENGTH` | VOL, STAT | `ta.atr()` length | SAFETY |
| `INPUT_ATR_MULT` | VOL, STAT | ATR multiplier | — |
| `INPUT_VOL_SMA_LEN` | VOL | `ta.sma(volume)` length | SAFETY |
| `INPUT_SHOW_VOL_SPIKE` | VOL | Volume spike toggle | — |
| `INPUT_VOL_SPIKE_THRESH` | VOL | Spike threshold | — |

### 2.14 Intrabar Resolver (INTRA)

| Input | Subsystem(s) | Consumer Functions/Variables | Risk Tag |
|-------|--------------|------------------------------|----------|
| `INPUT_INTRABAR_ENABLED` | INTRA | LTF request toggle | PERF |
| `INPUT_INTRABAR_TF` | INTRA | LTF resolution | PERF |
| `INPUT_INTRABAR_TIEBREAK` | INTRA | Tie-break policy | BEHAVIOR |

---

## 3. Risk Classification Legend

| Tag | Description | Mitigation |
|-----|-------------|------------|
| **SAFETY** | Can cause runtime error, na cascade, or bounds issue | Add minval/maxval or safe clamp |
| **PERF** | Enables heavy loops, requests, objects, or strings | Gate with `barstate.isconfirmed/islast`, use Fast preset |
| **BEHAVIOR** | Affects signals, execution, or learning outputs | Document, ensure parity for valid values |

---

## 4. Heavy Feature Gating Map

| Heavy Feature | Primary Toggle | Secondary Gates | Request/Loop Impact |
|---------------|----------------|-----------------|---------------------|
| Entropy (Legacy) | `INPUT_ENTROPY_ENABLED` | `INPUT_ENTROPY_MODE`, `perf_shouldComputeHeavy()` | O(N²) binning loop |
| Hurst (Legacy) | `INPUT_HURST_ENABLED` | `INPUT_HURST_MODE`, `perf_shouldComputeHeavy()` | O(N×scales) R/S loop |
| Permutation Test | `INPUT_permtestEnabled` | `barstate.islast` | 100+ Fisher-Yates shuffles |
| Learning Engine | `INPUT_LEARNING_ENABLED` | `barstate.islast` | Circular buffer iteration |
| External Requests | `INPUT_EXT_ENABLED` | `INPUT_EXT_MAX_SYMBOLS` | Up to 10 request contexts |
| Debug Overlays | `INPUT_*_DEBUG` | `perf_shouldShowDebugOverlay()` | Table cell population |
| ZigZag Polylines | `INPUT_ZZ_RENDER_MODE` | `INPUT_ZZ_POLY_MAX_KEEP` | Polyline GC loop |

---

## 5. Validation Requirements by Input

### 5.1 Inputs with Existing Validation ✅

| Input | Validation | Status |
|-------|------------|--------|
| `INPUT_POS_RSI_LEN` | minval=2, maxval=50 | ✅ |
| `INPUT_LEARNING_SAMPLES` | minval=10, maxval=500 | ✅ |
| `INPUT_ENTROPY_WINDOW` | minval=5, maxval=100 | ✅ |
| `INPUT_ENTROPY_BINS` | minval=3, maxval=10 | ✅ |
| `INPUT_ENTROPY_K` | minval=1, maxval=8 | ✅ |
| `INPUT_HURST_WINDOW` | minval=20, maxval=500 | ✅ |
| `INPUT_ZSCORE_*_LEN` | minval=5–20, maxval=50–200 | ✅ |
| `INPUT_ZZ_POLY_*` | minval=1/100, maxval=100/10000 | ✅ |
| Depth inputs (×25) | minval=1 | ✅ |
| Threshold inputs (×25) | minval=0 | ✅ |

### 5.2 Validation Gaps — FIXED ✅

| Input | Before | After | Rationale |
|-------|--------|-------|-----------|
| `INPUT_ATR_LENGTH` | No validation | `minval=1` | Prevents `ta.atr(0)` returning `na` |
| `INPUT_VOL_SMA_LEN` | No validation | `minval=1` | Prevents `ta.sma(volume, 0)` returning `na` |

---

## 6. Derived Constants Inventory

### 6.1 Timeframe-Derived

| Constant | Computation | Location | Consumers |
|----------|-------------|----------|-----------|
| `REQ_htf` | TF string resolution | ~L1900 | FIB, ZZ |
| `i_depth` | TF→depth lookup | ~L1905 | ZZ pivot calculation |
| `dev_threshold` | TF→threshold lookup | ~L1900 | ZZ deviation check |

### 6.2 Performance-Derived

| Constant | Computation | Location | Consumers |
|----------|-------------|----------|-----------|
| `perf_getEntropyWindow()` | Preset-adjusted window | L1061 | ENTROPY |
| `perf_getHurstWindow()` | Preset-adjusted window | L1066 | HURST |
| `perf_getHurstMode()` | Preset-forced mode | L1074 | HURST |
| `perf_getVisualMaxKeep()` | Preset-adjusted object cap | L1078 | ZZ, FIB |

### 6.3 ATR-Derived

| Constant | Computation | Location | Consumers |
|----------|-------------|----------|-----------|
| `i_weightedATR` | `ta.atr(INPUT_ATR_LENGTH) * INPUT_ATR_MULT` | L2909 | STAT, VOL |
| `i_vSMA` | `ta.sma(volume, INPUT_VOL_SMA_LEN)` | L2910 | VOL |

---

## 7. Group String Consolidation Status

| Group Constant | Location | Usage Count | Status |
|----------------|----------|-------------|--------|
| `UI_GROUP_perfControl` | L510 | 5 | ✅ Consolidated |
| `UI_GROUP_pick` | L201 | 3 | ✅ Consolidated |
| `UI_GROUP_pivot` | L202 | 5 | ✅ Consolidated |
| `UI_GROUP_fibTool` | L215 | 8 | ✅ Consolidated |
| `UI_GROUP_statPos` | L220 | 48 | ✅ Consolidated |
| `UI_GROUP_extContext` | L221 | 8 | ✅ Consolidated |
| `UI_GROUP_fibLevels` | L216 | 66 | ✅ Consolidated |
| `UI_GROUP_zigzag` | L217 | 5 | ✅ Consolidated |
| `UI_GROUP_volVol` | L218 | 6 | ✅ Consolidated |
| `UI_GROUP_alerts` | L219 | 3 | ✅ Consolidated |
| `UI_GROUP_thresh*` (6) | L203-208 | 25 | ✅ Consolidated |
| `UI_GROUP_depth*` (6) | L209-214 | 25 | ✅ Consolidated |

---

*Document generated as part of Task 3: User Inputs & UX Validation + Consolidation*
