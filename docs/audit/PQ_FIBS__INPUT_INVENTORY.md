# PQ_FIBS.pine — Input Inventory

> **Version**: 1.0  
> **Generated**: 2025-12-22  
> **Source**: `PQ_FIBS.pine` (5,591 lines)  
> **Pine Version**: v6  
> **Total Inputs**: 211

---

## 1. Input Summary by Group

| Group | Count | Primary Subsystem |
|-------|-------|-------------------|
| Performance Controller | 5 | Runtime gating |
| Pick a Fibonacci Tool | 3 | Feature toggles |
| Fibonacci Pivot Points Settings | 5 | Pivot display |
| Fibonacci Ext/Ret/TZ Settings | 8 | Fib rendering |
| Statistical Position Engine | 48 | Learning/regime/position |
| External Context | 8 | Dynamic requests |
| Fibonacci Levels | 66 | Level config (22 levels × 3) |
| ZigZag Settings | 5 | ZigZag rendering |
| Volume/Volatility AddOns | 6 | ATR/volume overlays |
| Alerts | 3 | Alert system |
| Threshold (per TF) | 25 | ZZ deviation thresholds |
| Depth (per TF) | 25 | ZZ pivot depths |
| **Total** | **211** | |

---

## 2. Performance Controller (5 inputs)

| Variable | Type | Default | Range | Location | Risk |
|----------|------|---------|-------|----------|------|
| `INPUT_PERF_PRESET` | string | `"🟢 Balanced"` | [Balanced, Fast, Debug] | L511 | PERF |
| `INPUT_PERF_SHOW_DASHBOARD` | bool | `false` | — | L512 | PERF |
| `INPUT_PERF_DEGRADE_HURST` | bool | `true` | — | L513 | PERF |
| `INPUT_PERF_DEGRADE_ENTROPY` | bool | `true` | — | L514 | PERF |
| `INPUT_PERF_SKIP_ALERTS` | bool | `false` | — | L515 | PERF |

---

## 3. Feature Toggles — Pick a Fibonacci Tool (3 inputs)

| Variable | Type | Default | Location | Risk |
|----------|------|---------|----------|------|
| `INPUT_SHOW_FIB_TIME` | bool | `true` | L659 | — |
| `INPUT_CUSTOM_THRESHOLD` | bool | `true` | L660 | — |
| `INPUT_CUSTOM_DEPTH` | bool | `true` | L661 | — |

---

## 4. Fibonacci Pivot Points Settings (5 inputs)

| Variable | Type | Default | Options/Range | Location | Risk |
|----------|------|---------|---------------|----------|------|
| `INPUT_HTF_MODE` | string | `"Auto"` | [Auto, User Defined] | L664 | — |
| `INPUT_HTF_USER_RAW` | string | `"15m"` | [15m, 1H, 4H, D, W, M, Q, Y] | L665 | — |
| `INPUT_LEVELS_PVT` | string | `"Prices"` | [Levels, Prices, Both, None] | L676 | — |
| `INPUT_LEVELS_PVT_POS` | string | `"Pivot End"` | [Last Bar, Pivot End] | L677 | — |
| `INPUT_LEVELS_PVT_SIZE` | string | `"Small"` | [Small, Normal] | L678 | — |
| `INPUT_EXTEND_PVT` | bool | `false` | — | L679 | — |

---

## 5. Fibonacci Extension/Retracement/TimeZone Settings (8 inputs)

| Variable | Type | Default | Range | Location | Risk |
|----------|------|---------|-------|----------|------|
| `INPUT_LEVELS_LABEL` | string | `"Prices"` | [Levels, Prices, Both, None] | L685 | — |
| `INPUT_LEVELS_POS` | string | `"Pivot Start"` | [Last Bar, Pivot Start] | L686 | — |
| `INPUT_LEVELS_SIZE` | string | `"Small"` | [Small, Normal] | L687 | — |
| `INPUT_REVERSE` | bool | `false` | — | L688 | — |
| `INPUT_EXTEND_ER` | bool | `false` | — | L689 | — |
| `INPUT_HIST_PIVOT` | int | `0` | min=0 | L690 | — |
| `INPUT_HIST_PIVOT_2` | int | `0` | min=0 | L691 | — |
| `INPUT_FIB_TZ_LABEL` | bool | `false` | — | L692 | — |
| `INPUT_FIB_TZ_POS_X` | string | `"Left"` | [Left, Right] | L693 | — |
| `INPUT_FIB_TZ_POS_Y` | string | `"Bottom"` | [Bottom, Top] | L695 | — |

---

## 6. Statistical Position Engine (48 inputs)

### 6.1 Core Position Settings

| Variable | Type | Default | Range | Location | Risk |
|----------|------|---------|-------|----------|------|
| `INPUT_SHOW_POS` | bool | `true` | — | L698 | — |
| `INPUT_POS_SL_MULT` | float | `1.5` | min=0.1, step=0.1 | L699 | BEHAVIOR |
| `INPUT_POS_SL_DYNAMIC` | bool | `true` | — | L700 | BEHAVIOR |
| `INPUT_POS_RSI_DIV` | bool | `true` | — | L701 | BEHAVIOR |
| `INPUT_POS_RSI_LEN` | int | `14` | 2–50 | L702 | SAFETY |
| `INPUT_POS_TIME_DECAY` | bool | `true` | — | L703 | BEHAVIOR |
| `INPUT_POS_TP_MODE` | string | `"Conservative"` | [Conservative, Aggressive] | L704 | BEHAVIOR |
| `INPUT_POS_WIN_RATE` | float | `61.8` | 1–99 | L705 | BEHAVIOR |
| `INPUT_POS_KELLY_FRAC` | float | `0.5` | 0.1–1.0 | L706 | BEHAVIOR |

### 6.2 Learning Engine

| Variable | Type | Default | Range | Location | Risk |
|----------|------|---------|-------|----------|------|
| `INPUT_LEARNING_ENABLED` | bool | `true` | — | L708 | PERF |
| `INPUT_LEARNING_SAMPLES` | int | `50` | 10–500 | L709 | SAFETY, PERF |
| `INPUT_ZONE_BUFFER_BASE` | float | `0.0` | 0–100 | L710 | BEHAVIOR |
| `INPUT_NEAR_MISS_THRESH` | float | `0.5` | 0.1–100 | L711 | BEHAVIOR |
| `INPUT_USE_LEARNED_SL` | bool | `true` | — | L712 | BEHAVIOR |
| `INPUT_LEARN_TP` | bool | `true` | — | L714 | BEHAVIOR |
| `INPUT_LEARN_DECAY` | bool | `true` | — | L715 | BEHAVIOR |
| `INPUT_LEARN_RSI_WEIGHT` | bool | `true` | — | L716 | BEHAVIOR |
| `INPUT_LEARN_MIN_SAMPLES` | int | `30` | 10–100 | L717 | SAFETY |

### 6.3 Projection Mode

| Variable | Type | Default | Range | Location | Risk |
|----------|------|---------|-------|----------|------|
| `INPUT_PROJECTION_MODE` | bool | `true` | — | L719 | BEHAVIOR |
| `INPUT_PROJ_SENSITIVITY` | float | `0.5` | 0.1–1.0 | L720 | BEHAVIOR |

### 6.4 Intrabar Precision

| Variable | Type | Default | Range | Location | Risk |
|----------|------|---------|-------|----------|------|
| `INPUT_INTRABAR_ENABLED` | bool | `true` | — | L722 | PERF |
| `INPUT_INTRABAR_TF` | string | `"5"` | [1, 5, 15] | L723 | PERF |
| `INPUT_INTRABAR_TIEBREAK` | string | `"Conservative"` | [Conservative, Optimistic] | L724 | BEHAVIOR |

### 6.5 Entropy Regime Filter

| Variable | Type | Default | Range | Location | Risk |
|----------|------|---------|-------|----------|------|
| `INPUT_ENTROPY_ENABLED` | bool | `true` | — | L726 | PERF |
| `INPUT_ENTROPY_MODE` | string | `"LEGACY_BINS"` | [LEGACY_BINS, BINARY_UPDOWN, BINARY_KGRAM] | L727 | PERF |
| `INPUT_ENTROPY_WINDOW` | int | `20` | 5–100 | L728 | SAFETY, PERF |
| `INPUT_ENTROPY_BINS` | int | `5` | 3–10 | L729 | SAFETY |
| `INPUT_ENTROPY_K` | int | `3` | 1–8 | L730 | SAFETY |
| `INPUT_ENTROPY_THRESH` | float | `0.7` | 0.1–1.0 | L731 | BEHAVIOR |
| `INPUT_ENTROPY_TRANSITION` | bool | `false` | — | L732 | BEHAVIOR |
| `INPUT_ENTROPY_DEBUG` | bool | `false` | — | L733 | PERF |

### 6.6 Hurst Regime Filter

| Variable | Type | Default | Range | Location | Risk |
|----------|------|---------|-------|----------|------|
| `INPUT_HURST_ENABLED` | bool | `true` | — | L735 | PERF |
| `INPUT_HURST_MODE` | string | `"🔸 Legacy"` | [Legacy, Dyadic Stable, Dyadic Fast] | L736 | PERF |
| `INPUT_HURST_WINDOW` | int | `100` | 20–500 | L737 | SAFETY, PERF |
| `INPUT_HURST_TREND` | float | `0.55` | 0.50–0.80 | L738 | BEHAVIOR |
| `INPUT_HURST_MEANREV` | float | `0.45` | 0.20–0.50 | L739 | BEHAVIOR |
| `INPUT_HURST_BLOCK_RANDOM` | bool | `false` | — | L740 | BEHAVIOR |
| `INPUT_HURST_DEBUG` | bool | `false` | — | L741 | PERF |

### 6.7 Momentum Z-Score Filter

| Variable | Type | Default | Range | Location | Risk |
|----------|------|---------|-------|----------|------|
| `INPUT_ZSCORE_ENABLED` | bool | `true` | — | L743 | PERF |
| `INPUT_ZSCORE_LR_LEN` | int | `14` | 5–50 | L744 | SAFETY |
| `INPUT_ZSCORE_ATR_LEN` | int | `14` | 5–50 | L745 | SAFETY |
| `INPUT_ZSCORE_Z_LEN` | int | `50` | 20–200 | L746 | SAFETY |
| `INPUT_ZSCORE_THRESH` | float | `2.0` | 1.0–4.0 | L747 | BEHAVIOR |

### 6.8 Bayesian Win-Rate

| Variable | Type | Default | Range | Location | Risk |
|----------|------|---------|-------|----------|------|
| `INPUT_BAYES_ENABLED` | bool | `true` | — | L749 | — |
| `INPUT_BAYES_ALPHA_PRIOR` | float | `1.0` | 0.1–10.0 | L750 | BEHAVIOR |
| `INPUT_BAYES_BETA_PRIOR` | float | `1.0` | 0.1–10.0 | L751 | BEHAVIOR |
| `INPUT_BAYES_CONFIDENCE` | float | `0.95` | 0.80–0.99 | L752 | BEHAVIOR |

### 6.9 Sortino Ratio

| Variable | Type | Default | Range | Location | Risk |
|----------|------|---------|-------|----------|------|
| `INPUT_SORTINO_ENABLED` | bool | `true` | — | L753 | — |
| `INPUT_SORTINO_MAR` | float | `0.0` | -0.10–0.10 | L754 | BEHAVIOR |

### 6.10 Monte Carlo Validation

| Variable | Type | Default | Range | Location | Risk |
|----------|------|---------|-------|----------|------|
| `INPUT_permtestEnabled` | bool | `true` | — | L756 | PERF |
| `INPUT_permtestSims` | int | `100` | 20–500 | L757 | PERF, SAFETY |
| `INPUT_permtestPvalue` | float | `0.05` | 0.01–0.20 | L758 | BEHAVIOR |

---

## 7. External Context — Dynamic Requests (8 inputs)

| Variable | Type | Default | Range | Location | Risk |
|----------|------|---------|-------|----------|------|
| `INPUT_EXT_ENABLED` | bool | `false` | — | L760 | PERF |
| `INPUT_EXT_TF_MODE` | string | `"CURRENT"` | [CURRENT, CUSTOM] | L761 | — |
| `INPUT_EXT_TF_CUSTOM` | string | `"60"` | — | L762 | — |
| `INPUT_EXT_SYMBOLS` | string | `""` | — | L763 | SAFETY |
| `INPUT_EXT_MAX_SYMBOLS` | int | `5` | 1–5 (capped) | L764 | PERF, SAFETY |
| `INPUT_EXT_FIELDS` | string | `"CLOSE_ONLY"` | [CLOSE_ONLY, OHLCV] | L765 | PERF |
| `INPUT_EXT_GATE_MODE` | string | `"ALWAYS"` | [ALWAYS, ON_ENTROPY, ON_HURST, CUSTOM] | L766 | PERF |
| `INPUT_EXT_DEBUG` | bool | `false` | — | L767 | PERF |
| `INPUT_DEBUG_FLOW` | bool | `false` | — | L769 | PERF |

---

## 8. Fibonacci Levels (66 inputs = 22 levels × 3)

Each level has 3 inputs: Show (bool), Value (float), Color (color).

| Level ID | Location (Show/Val/Color) | Default Show | Default Value |
|----------|---------------------------|--------------|---------------|
| 0 | L773-775 | `true` | `0.0` |
| 0.236 | L776-778 | `true` | `0.236` |
| 0.382 | L779-781 | `true` | `0.382` |
| 0.5 | L782-784 | `true` | `0.5` |
| 0.618 | L785-787 | `true` | `0.618` |
| 0.65 | L788-790 | `true` | `0.65` |
| 0.786 | L791-793 | `true` | `0.786` |
| 1.0 | L794-796 | `true` | `1.0` |
| 1.272 | L797-799 | `true` | `1.272` |
| 1.414 | L800-802 | `false` | `1.414` |
| 1.618 | L803-805 | `true` | `1.618` |
| 1.65 | L806-808 | `false` | `1.65` |
| 2.618 | L809-811 | `false` | `2.618` |
| 2.65 | L812-814 | `true` | `2.65` |
| 3.618 | L815-817 | `false` | `3.618` |
| 3.65 | L818-820 | `true` | `3.65` |
| 4.236 | L821-823 | `true` | `4.236` |
| 4.618 | L824-826 | `true` | `4.669` |
| -0.236 | L827-829 | `true` | `-0.236` |
| -0.382 | L830-832 | `true` | `-0.382` |
| -0.618 | L833-835 | `false` | `-0.618` |
| -0.65 | L836-838 | `true` | `-0.65` |

---

## 9. ZigZag Settings (5 inputs)

| Variable | Type | Default | Range | Location | Risk |
|----------|------|---------|-------|----------|------|
| `INPUT_SHOW_ZIGZAG` | bool | `true` | — | L841 | — |
| `INPUT_ZZ_RENDER_MODE` | string | `"Lines"` | [Lines, Polyline] | L849 | PERF |
| `INPUT_ZZ_POLY_MAX_KEEP` | int | `50` | 1–100 | L850 | SAFETY |
| `INPUT_ZZ_POLY_MAX_POINTS` | int | `9500` | 100–10000 | L851 | SAFETY |
| `INPUT_ZZ_LIVE_SEGMENT` | bool | `true` | — | L852 | — |

---

## 10. Volume/Volatility AddOns (6 inputs)

| Variable | Type | Default | Range | Location | Risk | Validation |
|----------|------|---------|-------|----------|------|------------|
| `INPUT_SHOW_HIGH_ATR` | bool | `false` | — | L860 | — | ✅ |
| `INPUT_ATR_LENGTH` | int | `7` | min=1 | L861 | SAFETY | ✅ Fixed |
| `INPUT_ATR_MULT` | float | `2.0` | min=0.1 | L862 | — | ✅ |
| `INPUT_VOL_SMA_LEN` | int | `89` | min=1 | L863 | SAFETY | ✅ Fixed |
| `INPUT_SHOW_VOL_SPIKE` | bool | `false` | — | L864 | — | ✅ |
| `INPUT_VOL_SPIKE_THRESH` | float | `2.5` | min=0.1 | L865 | — | ✅ |

---

## 11. Alerts (3 inputs)

| Variable | Type | Default | Location | Risk |
|----------|------|---------|----------|------|
| `INPUT_ALERT_ENABLED` | bool | `false` | L855 | — |
| `INPUT_ALERT_MODE` | string | `"Updates"` | L856 | — |
| `INPUT_ALERT_SEPARATOR` | string | `"\|"` | L857 | — |

---

## 12. Threshold per Timeframe (25 inputs)

All use `input.float(2.0, ..., minval=0)` pattern at L1833-1857.

| Timeframe | Variable Pattern | Location | Validation |
|-----------|------------------|----------|------------|
| Default | `dev_threshold` | L1833 | ✅ minval=0 |
| 1S–45S (6) | `dev_threshold` | L1834-1839 | ✅ minval=0 |
| 1M–45M (8) | `dev_threshold` | L1840-1847 | ✅ minval=0 |
| 1H–4H (4) | `dev_threshold` | L1848-1851 | ✅ minval=0 |
| 1D–7D (2) | `dev_threshold` | L1852-1853 | ✅ minval=0 |
| 4W (1) | `dev_threshold` | L1854 | ✅ minval=0 |
| 3Mo–12Mo (3) | `dev_threshold` | L1855-1857 | ✅ minval=0 |

---

## 13. Depth per Timeframe (25 inputs)

All use `input.int(10, ..., minval=1)` pattern at L1861-1885.

| Timeframe | Variable Pattern | Location | Validation |
|-----------|------------------|----------|------------|
| Default | `depth` | L1861 | ✅ minval=1 |
| 1S–45S (6) | `depth` | L1862-1867 | ✅ minval=1 |
| 1M–45M (8) | `depth` | L1868-1875 | ✅ minval=1 |
| 1H–4H (4) | `depth` | L1876-1879 | ✅ minval=1 |
| 1D–7D (2) | `depth` | L1880-1881 | ✅ minval=1 |
| 4W (1) | `depth` | L1882 | ✅ minval=1 |
| 3Mo–12Mo (3) | `depth` | L1883-1885 | ✅ minval=1 |

---

## 14. Validation Summary

All 211 inputs now have proper validation constraints. The following gaps were identified and **fixed**:

| Input | Before | After | Rationale |
|-------|--------|-------|-----------|
| `INPUT_ATR_LENGTH` | No minval | `minval=1` | Prevents `ta.atr(0)` which returns `na` |
| `INPUT_VOL_SMA_LEN` | No minval | `minval=1` | Prevents `ta.sma(volume, 0)` which returns `na` |

---

*Document generated as part of Task 3: User Inputs & UX Validation + Consolidation*
