# PQ_MOD Indicators Suite

![Trading View Authority](tva.png)

For nearly half a year I’ve been fine-tuning parameters and wrestling with some truly hellish mathematics to deliver a suite of TradingView scripts that you can parameterize on **any** candle timeframe. They consistently hit around **90% accuracy** in stable, technically-driven market cycles—and leverage non-standard analytical libraries and a serious mathematical engine under the hood.

---

## Table of Contents

1. [PQ_ACPG](#pq_acpg)  
2. [PQ_FIBS](#pq_fibs)  
3. [PQ_FTL](#pq_ftl)  
4. [PQ_HTMP](#pq_htmp)  
5. [PQ_MAVG](#pq_mavg)  
6. [PQ_QUANT](#pq_quant)  
7. [PQ_RNG](#pq_rng)  
8. [PQ_RSI](#pq_rsi)  
9. [PQ_SMIEO](#pq_smieo)  
10. [PQ_VWAP](#pq_vwap)  
11. [Final Note](#final-note)  

---

## How to Add a New Pine Script to TradingView

![Trading View Authority](add.png)

Follow these steps to create and load your own custom indicator in the Pine Editor:

1. **Open the Pine Editor**  
   - At the bottom of TradingView, click **Pine Editor** to reveal the script editor pane.

2. **Expand the Script Menu**  
   - Next to the current script name (e.g. `PQ_DBS`), click the ▼ arrow.  
   - A context menu will appear with options like **Save script**, **Make a copy…**, **Version history…**, and **Create new**.

3. **Create a New Indicator**  
   - In that menu, choose **Create new → Indicator**  
     _(macOS shortcut: `⌘K, ⌘I`)_  
   - TradingView will open a fresh Pine Script template.

4. **Paste Your Code & Save**  
   - Delete the template code and paste in your own Pine Script.  
   - Click the floppy-disk icon (or press `⌘S`) to **Save script**.  
   - Give it a descriptive name (for example, `My_Custom_Indicator`).

5. **Add It to the Chart**  
   - With your new script still open, click **Add to chart** at the top of the Pine Editor.  
   - Your indicator will now appear on the active chart. 

---

## PQ_ACPG

**TL;DR**  
**Rainbow Autocorrelation Periodogram **: using Ehlers Roofing Filter and Pearson correlation analysis to detect dominant market cycles with precision heatmap visualization.

**Description**  
- **Ehlers Roofing Filter**: Two-stage cascade—High-Pass Filter (cutoff period 48 default) removes trend, Super Smoother (period 10) removes noise, isolates pure cyclical component.  
- **Pearson Correlation Matrix**: Calculates autocorrelation across configurable lag range (8-48 bars) using proper statistical correlation formula with mean subtraction and standard deviation normalization.  
- **Dominant Cycle Detection**: Automatically identifies lag period with maximum positive correlation—indicates strongest repeating pattern for timing entries.  
- **Bottom-Right Heatmap Table**: Live correlation matrix displays color-coded cells (red=negative, yellow=moderate, green=strong positive) for up to 20 lag periods with dominant cycle highlighted.  
- **Cycle vs Trend Mode**: Distinguishes cyclic markets (corr > 0.3) from trending markets—optimal for adaptive strategy selection.  
- **Roofed Signal Plot**: Shows detrended/denoised price oscillation scaled by ATR for clean pattern visualization.  
- Configurable high-pass/super-smoother periods, min/max lag range, averaging length, and optional displays for dominant period line and heatmap table.  

---

## PQ_FIBS

**TL;DR**  
**Shox Fibonacci Pivots**: Self-optimizing Fibonacci system with ZigZag pivot detection, Golden Pocket entry zones, walk-forward backtesting engine, adaptive learning, and configurable projection sensitivity for early signal detection with reduced phase lag.

**Description**  
- **Dual-Mode Pivot Detection**: Confirmed ZigZag pivots (immutable, feed learning engine) + Projection Mode (real-time tentative pivots with configurable sensitivity 0.1-1.0 for early warnings).  
- **Statistical Position Engine**: Identifies Golden Pocket retracement zones (0.618-0.65) with dynamic SL/TP based on volatility regime, RSI divergence, and learned parameters.  
- **Walk-Forward Backtesting**: Spawns virtual trades on every historical pivot, tracks MAE/MFE bar-by-bar, calculates "Smart SL" from winning trade drawdown analysis, displays optimization stats (Win Rate, Confidence) on dashboard.  
- **4-Phase Learning Engine**: Records setup outcomes, adapts zone buffer from near-misses, optimizes SL multiplier, learns regime-specific win rates (high/low vol, long/short), calculates unified confidence score (0-100) and Kelly sizing.  
- **Advanced Analytics**: TP mode comparison (Conservative vs Aggressive), time decay rate learning, RSI divergence edge calculation, profit factor tracking, streak detection, health monitoring.  
- **Visual Feedback**: Solid visuals for confirmed setups, dashed+transparent for projections (transparency scales inversely with sensitivity—lower sensitivity = more transparent = higher false-positive risk).  
- Fibonacci time zones, volume/volatility overlays, multi-timeframe support, and alert system included.  

---

## PQ_FTL

**TL;DR**  
**Predictive Trend Swags**: Draws dynamic trendlines and channels based on recent highs/lows, highlights breakouts, and plots short-term forecasts to preview possible price paths.

**Description**  
- Detects swing highs/lows over a user-defined lookback.  
- Computes dynamic slopes via ATR, standard deviation, or linear regression.  
- Auto-selects the optimal lookback using multi-period correlation analysis.  
- Integrates an external forecasting library for price projections + variance bands.  
- Visualizes results in scatter or line plots, with extended trend channels.  
- Marks breakout events and optionally shows model stats (correlation, R², forecast extremes).  
- Alerts trigger on significant trendline breaches.  

---

## PQ_HTMP

**TL;DR**  
**Horny Heatmap Scalper**: with UDT-based Adaptive Kalman Filter, Z-score deviation analysis, and 6-stop turbo gradient coloring for visual momentum tracking.

**Description**  
- **Triple Baseline Options**: Choose between Adaptive Kalman Filter (AKF with volatility-ratio adjustment), Linear Regression (ta.linreg on hl2), or standard Kalman Filter for dynamic centerline.  
- **Z-Score Normalization**: Calculates deviation from baseline (50-bar default), computes mean and standard deviation of deviations, produces statistical Z-score clamped to sensitivity range.  
- **6-Stop Turbo Gradient**: Smooth color transitions across 5 zones—deep purple (extreme low) → blue → cyan → lime → orange → dark red (extreme high) using color.from_gradient() for liquid visual effect.  
- **Adaptive Kalman Filter UDT**: Object-oriented implementation with `method update()` that adjusts process noise (Q) and measurement noise (R) dynamically based on ATR volatility ratio (fast/slow ATR).  
- **Signal Sensitivity Control**: Adjustable threshold (0.5-5.0, default 2.0) determines Z-score range—higher = fewer signals (stronger extremes only), generates scalp entry triangles at 90% of threshold.  
- **Color Smoothing**: EMA smoothing (1-10 bars, default 3) applied to normalized heat index prevents color flickering while preserving responsiveness.  
- **Bar Opacity Control**: 0-90% adjustable transparency for gradient overlay, optional baseline plot with tri-color coding (gray=neutral, red/cyan=extreme).  
- Optimized for 1m/5m scalping—14-bar volatility lookback, configurable Q/R parameters for AKF response tuning, alerts on oversold/overbought crossovers.  

---

## PQ_MAVG

**TL;DR**  
**Fancy as Fuck Moving Averages**:overlays up to 18 customizable MAs across 4 timeframes (Intraday/Daily/Weekly/Monthly) with optional cloud fills, ATR-based SuperTrend signals, and 14 MA algorithm choices per plot.

**Description**  
- **18 Moving Average Slots**: 5 Intraday + 5 Daily + 4 Weekly + 4 Monthly—each with independent visibility toggle, length (1-5000), source, and color.  
- **14 Algorithm Types**: SMA, EMA, RMA, VWMA, WMA, HMA (Hull), TEMA (Triple EMA), ZLEMA (Zero-Lag), DEMA (Double EMA), KAMA (Kaufman Adaptive), TRIMA (Triangular), JMA (Jurik), ALMA (Arnaud Legoux), FRAMA (Fractal Adaptive), VWEMA, GMA, MDA, EVWMA—all with algorithm-specific parameter controls (KAMA fast/slow periods, JMA phase, ALMA offset/sigma, FRAMA fast fractal).  
- **Cloud Fills**: Optional per-timeframe clouds (Intraday/Daily/Weekly) fill space between first two MAs with up/down color coding—instantly shows MA crossovers and hierarchy.  
- **SuperTrend Signals**: ATR-based (10-period default) long/short entries with 3.0x multiplier, optional "Change ATR Calculation Method" toggle, plots triangle markers (buy=▲ below bars, sell=▼ above bars) with lime/red coloring.  
- **Multi-Timeframe Display**: "Show Weekly on Daily" toggle with orange overlay, separate parameter groups for each timeframe class.  
- **Per-MA Customization**: Every MA has dedicated inputs for Fast/Slow KAMA periods, JMA phase (-100 to +100), ALMA offset (0-1) + sigma, FRAMA fast fractal (0.1-2.0).  
- **Visual Options**: 1-4px line width, adjustable cloud transparency (35% default fills), up/down plot colors (lime/red), signal text colors, plot fill color (blue 12.5% transparency).  
- **Signal Highlighting**: Optional bar coloring based on SuperTrend direction with long fill (lime 35%) and short fill (red 35%).  
- Imports TradingView/ta/8 library, supports max_labels_count=500, max_lines_count=200 for extensive overlay capability.  
- Ideal for multi-timeframe trend confirmation—align Intraday/Daily/Weekly MAs for confluence-based entries.

---

## PQ_QUANT

**TL;DR**  
**WünderQuant**: Quantitative analysis library (exported functions) implementing matrix-based Hurst Exponent R/S analysis, Shannon Entropy, and Weighted Least Squares regression for advanced statistical modeling.

**Description**  
- **Hurst Exponent R/S Analysis**: `get_hurst_exponent(src, length)` measures fractal dimension and long-term memory in time series using Rescaled Range method.  
  - H ≈ 0.5: Random walk (Brownian motion, efficient market)  
  - H > 0.5: Persistent/trending (momentum-favorable markets)  
  - H < 0.5: Anti-persistent/mean-reverting (reversal-favorable markets)  
  - Returns clamped float [0, 1] with `interpret_hurst()` helper (Strong Trend/Trending/Random Walk/Mean-Reverting/Strong Mean-Reversion).  
- **Shannon Entropy**: `get_shannon_entropy(src, length, bins)` calculates information-theoretic disorder normalized to [0, 1].  
  - Entropy = 0: Perfect order (all values in one bin, strong trending)  
  - Entropy = 1: Maximum disorder (uniform distribution, chaotic/random market)  
  - Uses dynamic binning, histogram population, and log₂ calculations for proper Shannon formula H = -Σ p_i · log₂(p_i).  
  - Returns normalized entropy with `interpret_entropy()` helper (Chaotic/Random/Mixed/Trending/Ordered).  
- **Weighted Least Squares**: `get_wls_slope(x_array, y_array, weights_array)` performs matrix-based WLS regression using Normal Equation β = (X^T W X)^(-1) X^T W y.  
  - Returns slope coefficient β₁ from design matrix [1, x] with diagonal weight matrix.  
  - Also provides `get_wls_coefficients()` for full [intercept, slope] tuple.  
  - Includes ridge regularization (1e-8) for numerical stability and weight normalization to prevent overflow.  
- **Matrix Implementation**: All calculations use Pine Script v6 `matrix.*` functions (transpose, mult, inv, get, set) for optimal performance and numerical accuracy.  
- **R/S Methodology**: Divides series into blocks at multiple sub-period sizes (8, 16, 32, 64, 128 bars), calculates cumulative deviations + range + standard deviation for each block, computes log-log regression on averaged R/S vs n.  
- **Library Module**: Designed as @version=6 library with exported functions—includes demonstration plots (Hurst, Entropy, reference lines) and top-right info table with regime labels.  
- Supports configurable lookback periods, number of bins for entropy, and provides test WLS validation (y = 2x + 1 example).  

---

## PQ_RNG

**TL;DR**  
**Swoosh Range Detector**: with comprehensive Smart Money Concepts—identifies consolidation zones, tracks internal/swing structure (BOS/CHoCH), plots order blocks with volatility filtering, detects equal highs/lows, fair value gaps, premium/discount zones, and MTF pivot levels.

**Description**  
- **Range Detection**: Computes minimum range length (10 bars) and width threshold (0.8 × ATR) to identify consolidation zones—draws unbroken/broken range boxes with color coding.  
- **Predictive Range Projection**: Uses 250-bar lookback + 3.0x factor on custom timeframe to forecast future support/resistance bands.  
- **Smart Money Concepts Suite**: 
  - **Swing Structure**: Displays market structure breaks (BOS) and changes of character (CHoCH) based on swing highs/lows with configurable leg detection.  
  - **Internal Structure**: Real-time internal BOS/CHoCH with confluence filter to reduce noise—separate color/label settings from swing structure.  
  - **Order Blocks**: Tracks up to 20 internal + 20 swing order blocks with UDT storage (`orderBlock` type stores barHigh, barLow, barTime, bias).  
  - **Order Block Filtering**: Choice of ATR (14-period) or Cumulative Mean Range to filter high-volatility false blocks—configurable mitigation levels (close vs high/low).  
  - **Equal Highs/Lows**: 3-bar confirmation with 0.1 threshold sensitivity—plots EQH/EQL labels and horizontal lines at detected levels.  
  - **Fair Value Gaps**: Auto-threshold FVG detection with configurable timeframe, 5-bar extension, separate bullish/bearish coloring (green/red with 70% opacity).  
  - **Premium/Discount Zones**: Overlays Fibonacci-based zones from swing structure—premium (bearish bias), equilibrium (neutral), discount (bullish bias).  
  - **MTF Levels**: Optional Daily/Weekly/Monthly high/low levels with customizable line styles (solid/dashed/dotted) and colors.  
- **UDT Architecture**: Uses 10+ custom types (alerts, trailingExtremes, fairValueGap, trend, equalDisplay, pivot, orderBlock) for organized state management.  
- **Mode Selection**: Historical (shows all past structure) vs Present (only recent/active structure)—reduces visual clutter on higher timeframes.  
- **Styling Options**: Colored vs Monochrome themes, adjustable label sizes (tiny/small/normal), separate color controls for all bullish/bearish elements.  
- **Alert System**: Fires alerts on internal/swing BOS/CHoCH, order block formations, EQH/EQL detection, and FVG creation—all with separate bool flags in `alerts` UDT.  
- Includes trend-colored candles, strong/weak high/low highlighting (10-bar lookback for swing points), and confluence filtering for internal structure.

---

## PQ_RSI

**TL;DR**  
**TurboRSI**: Blends a fast True Strength Index (TSI) oscillator, a multi-length RSI ensemble with stochastic smoothing, a candle heatmap based on regression deviation, plus overbought/oversold bands, signal lines, and an optional stats table.

**Description**  
- Computes TSI with user-defined short/long periods + signal EMA; scales linearly and logarithmically to highlight extremes.  
- Aggregates RSI over a configurable range to produce smoothed bullish/bearish channel bounds.  
- Calculates a regression line over a specified length; normalizes price deviation to a gradient for candle heat-mapping.  
- Overlays overbought/oversold zones, plotted channels, and reference lines at 80/50/20.  
- Optional table displays key metrics (% overbought/oversold, etc.).  

---

## PQ_SMIEO

**TL;DR**  
A straightforward True Strength Index (TSI) + EMA signal plot with the TSI–signal difference shown as a histogram for quick momentum assessment.

**Description**  
- Calculates TSI on your chosen source with customizable short/long periods.  
- Derives an EMA-based signal line.  
- Plots TSI & signal as translucent circles; oscillator (TSI–signal) as a histogram.  
- Helps you spot overbought/oversold conditions and momentum shifts at a glance.  

---

## PQ_VWAP

**TL;DR**  
**Milfed VWAP Bands**: UDT-based regime detection (Hurst exponent + volatility), asset-specific multiplier maps, per-timeframe manual parameters (1m/5m optimized), and optional signal filtering based on market regime.

**Description**  
- **Dual-Mode Operation**: Auto Mode scales length/multiplier from Daily base values using timeframe factors, Manual Mode uses per-timeframe presets (1m: 50/2.5, 5m: 30/2.2, 15m: 20/2.0, 1H: 14/2.0, 4H: 10/1.8, D: 8/1.5, W+: 5/1.2).  
- **RegimeState UDT**: Encapsulates Hurst exponent (trending vs mean-reverting), volatility (ATR-normalized %), trend duration tracking, and signal filter flags for adaptive strategy gating.  
- **Hurst Exponent Calculation**: Choice of Variance Ratio (more stable, default) or Rescaled Range methods—calculates H from log-return variance ratios to classify regime above/below configurable threshold (0.5 default).  
- **Signal Filter**: Optional regime-based gating disables trend-following signals in mean-reverting markets (H < threshold) or vice versa—prevents counter-regime false signals.  
- **Asset Volatility Map**: O(1) lookups for 60+ assets (BTC: 3.5x, ETH: 3.2x, SPX: 1.5x, EURUSD: 1.0x, etc.) automatically adjust band width for asset-specific volatility characteristics.  
- **Anchor UDT**: Pivot-anchored VWAPs with `method update()` for cumulative price*volume tracking—auto-resets on swing highs/lows for responsive institutional level detection.  
- **VWAP Calculation**: Standard cumulative volume-weighted average with variance-based band multiplier—aggregates from session start or custom anchor points.  
- **Regime Dashboard**: Displays current Hurst value, regime classification (📈 Trend++ / 📊 Revert+), volatility %, and filter status in top-right info label.  
- Manual parameter overrides available: custom volatility factor, Hurst threshold adjustment, regime detection toggle, lookback period (10-100 bars).  
- Optimized for 1m/5m scalping with tighter bands and faster response—also supports Daily/Weekly/Monthly with reduced sensitivity.

---

## Final Note

These indicators took off immediately—over **30** installs in the first day alone—until they were quietly banned. (Let’s just say their effectiveness spoke for itself, yolo)  

![Trading View Authority](ban.png)