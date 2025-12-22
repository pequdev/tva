# PQ_FIBS.pine — Request Context Ledger

> **Version**: 1.0  
> **Generated**: 2025-12-22  
> **Source**: `PQ_FIBS.pine` (5,591 lines)  
> **Pine Version**: v6

---

## 1. Executive Summary

| Metric | Value |
|--------|-------|
| **Total Request Call Sites** | 4 (2 helper functions + 2 global intrabar arrays) |
| **Base Context Count (LTF)** | 2 (fixed, always present) |
| **Dynamic Context Count (External)** | 0–20 (depending on `INPUT_EXT_MAX_SYMBOLS` and fields mode) |
| **Maximum Possible Contexts** | 22 (2 LTF + 20 external) |
| **Default Context Count** | 2 (external OFF by default) |
| **Budget Cap** | 40 (`STATIC_EXT_maxContexts`) |

---

## 2. Request Call Site Inventory

### 2.1 Intrabar LTF Requests (Global Scope)

| ID | Call Site | Location | Function |
|----|-----------|----------|----------|
| **LTF-1** | `request.security_lower_tf(syminfo.tickerid, INPUT_INTRABAR_TF, high)` | L2476 | Returns `array<float>` of intrabar highs |
| **LTF-2** | `request.security_lower_tf(syminfo.tickerid, INPUT_INTRABAR_TF, low)` | L2477 | Returns `array<float>` of intrabar lows |

**Global Variables:**
```pine
array<float> REQ_ltfHighs = request.security_lower_tf(syminfo.tickerid, INPUT_INTRABAR_TF, high)
array<float> REQ_ltfLows  = request.security_lower_tf(syminfo.tickerid, INPUT_INTRABAR_TF, low)
```

**Scope Constraint (per Pine v6 docs):**
> `request.security_lower_tf()` MUST be declared at global scope. Cannot be inside conditionals or functions.

### 2.2 External Context Requests (Helper Functions)

| ID | Call Site | Location | Function |
|----|-----------|----------|----------|
| **EXT-1** | `f_reqExtClose(sym, tf)` | L1693-L1694 | Single-value close request |
| **EXT-2** | `f_reqExtOHLCV(sym, tf)` | L1696-L1697 | Tuple OHLCV request |

**Helper Implementations:**
```pine
f_reqExtClose(string sym, string tf) =>
    request.security(sym, tf, close, ignore_invalid_symbol = true)

f_reqExtOHLCV(string sym, string tf) =>
    request.security(sym, tf, [open, high, low, close, volume], ignore_invalid_symbol = true)
```

---

## 3. Request Options Analysis

### 3.1 Intrabar Requests (LTF-1, LTF-2)

| Parameter | Value | Source | Notes |
|-----------|-------|--------|-------|
| `symbol` | `syminfo.tickerid` | Fixed | Current chart symbol |
| `timeframe` | `INPUT_INTRABAR_TF` | Input (L723) | Options: "1", "5", "15" |
| `expression` | `high` / `low` | Fixed | Price series |
| `ignore_invalid_symbol` | Default (`false`) | Implicit | Not specified |
| `currency` | Default (none) | Implicit | Not specified |
| `ignore_invalid_timeframe` | Default (`false`) | Implicit | Not specified |
| `calc_bars_count` | Default (all) | Implicit | Not specified |

**Doc Reference (Pine v6):**
> `request.security_lower_tf()` returns an array with one element per intrabar within the chart bar. Array size varies on the forming bar and stabilizes on historical bars.

### 3.2 External Requests (EXT-1, EXT-2)

| Parameter | Value | Source | Notes |
|-----------|-------|--------|-------|
| `symbol` | Dynamic (CSV parsed) | `INPUT_EXT_SYMBOLS` | Max 20 via `STATIC_EXT_maxSymbolsCap` |
| `timeframe` | Dynamic | `f_getExtTimeframe()` | CURRENT or CUSTOM |
| `expression` | `close` or `[open,high,low,close,volume]` | `INPUT_EXT_FIELDS` | Single vs tuple |
| `ignore_invalid_symbol` | `true` | Explicit (L1694, L1697) | Returns `na` for invalid symbols |
| `lookahead` | Default (`barmerge.lookahead_off`) | Implicit | **Audit Note: Not explicit** |
| `gaps` | Default (`barmerge.gaps_off`) | Implicit | **Audit Note: Not explicit** |

**Doc Reference (Pine v6):**
> Default `lookahead` is `barmerge.lookahead_off` — uses last confirmed value from requested timeframe.
> Default `gaps` is `barmerge.gaps_off` — fills missing data with last valid value.

---

## 4. Unique Context Key Model

### 4.1 What Constitutes a Unique Context?

Per Pine v6 documentation, a unique request context is determined by:

```
UniqueContextKey = (symbol, timeframe, expression_shape, request_options)
```

| Component | Description |
|-----------|-------------|
| `symbol` | The ticker ID being requested |
| `timeframe` | The timeframe string |
| `expression_shape` | The structure of the expression (e.g., `close` vs `[o,h,l,c,v]`) |
| `request_options` | Combination of `lookahead`, `gaps`, `ignore_invalid_symbol`, etc. |

### 4.2 Context Count Per Request Type

| Request Type | Contexts Per Call | Notes |
|--------------|-------------------|-------|
| `request.security_lower_tf(..., high)` | 1 | Fixed symbol, fixed expression |
| `request.security_lower_tf(..., low)` | 1 | Fixed symbol, fixed expression |
| `request.security(sym, tf, close, ...)` | 1 per symbol | Dynamic symbol, close expression |
| `request.security(sym, tf, [o,h,l,c,v], ...)` | 1 per symbol | Dynamic symbol, tuple expression |

**Critical Insight:**
> If `INPUT_EXT_FIELDS` is "CLOSE_ONLY", the script uses `f_reqExtClose()`.
> If `INPUT_EXT_FIELDS` is "OHLCV", the script uses `f_reqExtOHLCV()`.
> Only ONE mode is active per execution — contexts are not additive across modes.

---

## 5. Consumer Traceability

### 5.1 Intrabar Arrays (REQ_ltfHighs, REQ_ltfLows)

| Consumer | Location | Purpose | Signal Impact |
|----------|----------|---------|---------------|
| `f_resolveIntrabar()` | L919-L955 | Chronological TP/SL resolution | **BACKTEST ONLY** |
| Backtest loop | L3936-L3939 | Called when `INPUT_INTRABAR_ENABLED` and `sl_hit and tp_hit` | Determines win/loss ordering |

**Call Chain:**
```
REQ_ltfHighs/REQ_ltfLows (global)
    └── L3936: if INPUT_INTRABAR_ENABLED and array.size(REQ_ltfHighs) > 0
        └── L3939: f_resolveIntrabar(REQ_ltfHighs, REQ_ltfLows, ...)
            └── Returns: [outcome, ltf_resolved, hit_idx]
                └── L3942: trade_won := ltf_outcome == 1
```

**Resolver Boundary:**
> Intrabar arrays affect ONLY the backtest trade resolution (win/loss determination).
> They do NOT influence: signal generation, regime classification, entry zone detection.

### 5.2 External Context Arrays

| Consumer | Location | Purpose | Signal Impact |
|----------|----------|---------|---------------|
| `GLOBAL_extContext.extOk` | L3843 | Regime gating composite | **INDIRECT — Gating only** |
| Debug overlay | L5486-L5487 | Diagnostic display | None |

**Call Chain:**
```
f_reqExtClose() / f_reqExtOHLCV() (helpers)
    └── L3089-L3103: External request loop
        └── GLOBAL_extContext.closes.push(), .opens.push(), etc.
            └── GLOBAL_extContext.validCount += 1 if not na()
                └── L3110: _extOk := validCount > 0
                    └── L3843: _regimeGatesPassed = ... and GLOBAL_extContext.extOk
```

**Critical Finding:**
> `extOk` participates in `_regimeGatesPassed` at L3843.
> When `INPUT_EXT_ENABLED = false` (default), `extOk = true` — no gating effect.
> When enabled, `extOk = validCount > 0` — affects whether regime gates pass.

---

## 6. Request Gating Conditions

### 6.1 Intrabar Requests (LTF-1, LTF-2)

| Condition | Gating Effect |
|-----------|---------------|
| Global scope | Always executed (Pine constraint) |
| `INPUT_INTRABAR_ENABLED` | Gates **consumption**, not request execution |

**Important:** The LTF requests always execute at global scope. Only their consumption in `f_resolveIntrabar()` is gated.

### 6.2 External Requests (EXT-1, EXT-2)

| Gate Layer | Condition | Location |
|------------|-----------|----------|
| **Layer 1: Master Toggle** | `INPUT_EXT_ENABLED == true` | L760, L3063 |
| **Layer 2: Regime Gating** | `f_computeExtGate(INPUT_EXT_GATE_MODE, ...)` | L1684-L1691, L3063 |
| **Layer 3: Budget Check** | `not GLOBAL_extContext.budgetExceeded` | L3067 |
| **Layer 4: Symbol Count** | `_symCount > 0` (parsed from CSV) | L3069-L3075 |

**Combined Gate (L3067):**
```pine
if _extGated and not GLOBAL_extContext.budgetExceeded
    // Execute external requests
```

---

## 7. Timeframe Resolution

### 7.1 Resolver Function

| Function | Location | Logic |
|----------|----------|-------|
| `f_getExtTimeframe(tf_mode, tf_custom)` | L1681-L1682 | Returns effective TF string |

```pine
f_getExtTimeframe(string tf_mode, string tf_custom) =>
    tf_mode == STATIC_EXT_tfCurrent ? timeframe.period : tf_custom
```

### 7.2 Timeframe Inputs

| Input | Default | Options | Location |
|-------|---------|---------|----------|
| `INPUT_EXT_TF_MODE` | `"CURRENT"` | CURRENT, CUSTOM | L761 |
| `INPUT_EXT_TF_CUSTOM` | `"60"` | User string | L762 |
| `INPUT_INTRABAR_TF` | `"5"` | 1, 5, 15 | L723 |

### 7.3 TF Validation Status

| TF Source | Validation | Notes |
|-----------|------------|-------|
| `INPUT_EXT_TF_CUSTOM` | ⚠️ No validation | User can enter invalid TF strings |
| `INPUT_INTRABAR_TF` | ✅ Options-constrained | Only "1", "5", "15" allowed |

**Risk:** Invalid `INPUT_EXT_TF_CUSTOM` may cause `request.security` to fail silently or return `na`.

---

## 8. Request Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        GLOBAL SCOPE (L2474-L2477)                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  REQ_ltfHighs = request.security_lower_tf(..., high)        │  │
│  │  REQ_ltfLows  = request.security_lower_tf(..., low)         │  │
│  │                                                              │  │
│  │  [2 contexts, always executed]                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    CONDITIONAL SCOPE (L3067-L3103)                  │
│                                                                     │
│  if _extGated and not budgetExceeded                               │
│    for i = 0 to _symCount - 1                                       │
│      if INPUT_EXT_FIELDS == OHLCV:                                  │
│        f_reqExtOHLCV(sym, tf)     [1 context per sym]              │
│      else:                                                          │
│        f_reqExtClose(sym, tf)     [1 context per sym]              │
│                                                                     │
│  [0-20 contexts, conditionally executed]                            │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        CONSUMPTION PATHS                            │
│                                                                     │
│  REQ_ltfHighs/Lows → f_resolveIntrabar() → Backtest win/loss       │
│                                                                     │
│  GLOBAL_extContext.closes → extOk → _regimeGatesPassed             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 9. Summary Table

| Request ID | Location | Symbol | Timeframe | Expression | Contexts | Gated By |
|------------|----------|--------|-----------|------------|----------|----------|
| LTF-1 | L2476 | `syminfo.tickerid` | `INPUT_INTRABAR_TF` | `high` | 1 | None (global) |
| LTF-2 | L2477 | `syminfo.tickerid` | `INPUT_INTRABAR_TF` | `low` | 1 | None (global) |
| EXT-1 | L1693-L1694 | Dynamic CSV | Dynamic TF | `close` | 0-20 | 4-layer gate |
| EXT-2 | L1696-L1697 | Dynamic CSV | Dynamic TF | `[o,h,l,c,v]` | 0-20 | 4-layer gate |

---

*Document generated as part of Task 4: Data Requests & Timeframe Resolver Audit*
