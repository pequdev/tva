# PQ_FIBS Request Context Ledger — Final Version

**Version:** vFINAL  
**Date:** 23 December 2025  
**Source:** `PQ_FIBS.pine` (5,605 lines)  
**Branch:** `55-pq_fibs-33-audit---logging-docs-consolidation`  
**Tasks Consolidated:** 4, 8

---

## 1. Executive Summary

| Metric | Value |
|--------|-------|
| **Total Request Call Sites** | 4 (2 LTF + 2 External helpers) |
| **Base Context Count (LTF)** | 2 (fixed, always present) |
| **Dynamic Context Count (External)** | 0–20 |
| **Maximum Possible Contexts** | 22 |
| **Default Context Count** | 2 (external OFF by default) |
| **TradingView Limit** | 40 |
| **Safety Margin** | 45% (at max config) |

---

## 2. Request Call Site Inventory

### 2.1 Intrabar LTF Requests (Global Scope)

| ID | Location | Symbol | Timeframe | Expression | Contexts |
|----|----------|--------|-----------|------------|----------|
| **LTF-1** | L2476 | `syminfo.tickerid` | `INPUT_INTRABAR_TF` | `high` | 1 |
| **LTF-2** | L2477 | `syminfo.tickerid` | `INPUT_INTRABAR_TF` | `low` | 1 |

```pine
array<float> REQ_ltfHighs = request.security_lower_tf(syminfo.tickerid, INPUT_INTRABAR_TF, high)
array<float> REQ_ltfLows  = request.security_lower_tf(syminfo.tickerid, INPUT_INTRABAR_TF, low)
```

**Pine v6 Constraint:** `request.security_lower_tf()` MUST be at global scope.

### 2.2 External Context Requests (Helper Functions)

| ID | Location | Function | Expression | Contexts |
|----|----------|----------|------------|----------|
| **EXT-1** | L1693-L1694 | `f_reqExtClose(sym, tf)` | `close` | 1 per symbol |
| **EXT-2** | L1696-L1697 | `f_reqExtOHLCV(sym, tf)` | `[o,h,l,c,v]` | 1 per symbol |

```pine
f_reqExtClose(string sym, string tf) =>
    request.security(sym, tf, close, ignore_invalid_symbol = true)

f_reqExtOHLCV(string sym, string tf) =>
    request.security(sym, tf, [open, high, low, close, volume], ignore_invalid_symbol = true)
```

---

## 3. Request Options Matrix

### 3.1 LTF Requests (LTF-1, LTF-2)

| Parameter | Value | Source |
|-----------|-------|--------|
| `symbol` | `syminfo.tickerid` | Fixed |
| `timeframe` | `INPUT_INTRABAR_TF` | Input: "1", "5", "15" |
| `expression` | `high` / `low` | Fixed |
| `ignore_invalid_symbol` | `false` (default) | Implicit |
| `lookahead` | N/A | Not applicable to LTF |

### 3.2 External Requests (EXT-1, EXT-2)

| Parameter | Value | Source |
|-----------|-------|--------|
| `symbol` | Dynamic | CSV from `INPUT_EXT_SYMBOLS` |
| `timeframe` | Dynamic | `f_getExtTimeframe()` |
| `expression` | `close` or tuple | `INPUT_EXT_FIELDS` |
| `ignore_invalid_symbol` | `true` | Explicit |
| `lookahead` | `barmerge.lookahead_off` | Default (implicit) |
| `gaps` | `barmerge.gaps_off` | Default (implicit) |

---

## 4. Unique Context Key Model

```
UniqueContextKey = (symbol, timeframe, expression_shape, request_options)
```

### 4.1 Context Count Per Request Type

| Request | Symbol | TF | Expression | Options | Key |
|---------|--------|-----|------------|---------|-----|
| LTF-1 | current | INPUT_INTRABAR_TF | high | default | Unique |
| LTF-2 | current | INPUT_INTRABAR_TF | low | default | Unique |
| EXT-1 | sym[i] | extTF | close | ignore=true | Per-symbol |
| EXT-2 | sym[i] | extTF | [o,h,l,c,v] | ignore=true | Per-symbol |

**Key Insight:** External requests use either CLOSE_ONLY or OHLCV mode — never both simultaneously.

---

## 5. Gating Layers

### 5.1 LTF Request Gating

| Layer | Condition | Effect |
|-------|-----------|--------|
| Execution | Always | Global scope, cannot gate |
| Consumption | `INPUT_INTRABAR_ENABLED` | Gates `f_resolveIntrabar()` |

**Note:** LTF arrays always exist; only their consumption is gated.

### 5.2 External Request Gating (4-Layer)

| Layer | Condition | Location | Effect |
|-------|-----------|----------|--------|
| **1. Master** | `INPUT_EXT_ENABLED` | L760 | Feature toggle |
| **2. Regime** | `f_computeExtGate()` | L1684-1691 | Regime-aware |
| **3. Budget** | `not budgetExceeded` | L3067 | Cap protection |
| **4. Symbols** | `_symCount > 0` | L3069 | Valid CSV |

```pine
if _extGated and not GLOBAL_extContext.budgetExceeded
    for i = 0 to _symCount - 1
        // Execute requests
```

---

## 6. Consumer Traceability

### 6.1 LTF Arrays → Intrabar Resolver

```
REQ_ltfHighs/REQ_ltfLows (L2476-2477)
    └── L3936: if INPUT_INTRABAR_ENABLED and size > 0
        └── L3939: f_resolveIntrabar(...)
            └── Returns: [outcome, ltf_resolved, hit_idx]
                └── L3942: trade_won := outcome == 1
```

**Boundary:** Affects ONLY backtest trade resolution (win/loss ordering).  
**Does NOT affect:** Signal generation, regime, entry zones.

### 6.2 External Context → Regime Gating

```
f_reqExtClose/OHLCV (L1693-1697)
    └── L3089-L3103: External request loop
        └── GLOBAL_extContext.closes.push(...)
            └── validCount += 1 if not na
                └── L3110: _extOk := validCount > 0
                    └── L3843: _regimeGatesPassed = ... and extOk
```

**Boundary:** `extOk` participates in regime gating.  
**Default:** `INPUT_EXT_ENABLED = false` → `extOk = true` (no gating).

---

## 7. Context Budget Formula

### 7.1 Worst-Case Context Count

```
Contexts = LTF_base + EXT_count

Where:
  LTF_base = 2 (always)
  EXT_count = min(parsed_symbols, STATIC_EXT_maxSymbolsCap) when enabled
            = 0 when disabled

Max = 2 + 20 = 22 (well under 40 limit)
```

### 7.2 Budget Enforcement

| Constant | Value | Location |
|----------|-------|----------|
| `STATIC_EXT_maxContexts` | 40 | L1661 |
| `STATIC_EXT_maxSymbolsCap` | 20 | L1662 |

```pine
GLOBAL_extContext.budgetExceeded := usedContexts > STATIC_EXT_maxContexts
```

---

## 8. Request Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    GLOBAL SCOPE (L2476-2477)                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  REQ_ltfHighs = request.security_lower_tf(..., high)      │  │
│  │  REQ_ltfLows  = request.security_lower_tf(..., low)       │  │
│  │  [2 contexts — always executed]                           │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              CONDITIONAL SCOPE (L3067-L3103)                    │
│  if _extGated and not budgetExceeded:                          │
│    for i = 0 to _symCount - 1:                                  │
│      OHLCV mode: f_reqExtOHLCV(sym, tf)  [1 ctx/sym]          │
│      CLOSE mode: f_reqExtClose(sym, tf)  [1 ctx/sym]          │
│  [0-20 contexts — conditionally executed]                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CONSUMPTION PATHS                            │
│  REQ_ltfHighs/Lows → f_resolveIntrabar → Backtest win/loss     │
│  GLOBAL_extContext → extOk → _regimeGatesPassed                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. Timeframe Configuration

### 9.1 LTF Timeframe

| Input | Default | Options | Validation |
|-------|---------|---------|------------|
| `INPUT_INTRABAR_TF` | `"5"` | "1", "5", "15" | ✅ Options-constrained |

### 9.2 External Timeframe

| Input | Default | Options | Validation |
|-------|---------|---------|------------|
| `INPUT_EXT_TF_MODE` | `"CURRENT"` | CURRENT, CUSTOM | ✅ |
| `INPUT_EXT_TF_CUSTOM` | `"60"` | User string | ⚠️ No validation |

**Risk:** Invalid `INPUT_EXT_TF_CUSTOM` → `request.security` may return `na`.

---

## 10. Lookahead & Repaint Analysis

### 10.1 LTF Requests

| Aspect | Status | Notes |
|--------|--------|-------|
| Lookahead | N/A | LTF returns array, no lookahead |
| Repaint risk | ✅ None | Intrabar data is confirmed |

### 10.2 External Requests

| Aspect | Status | Notes |
|--------|--------|-------|
| Lookahead | `barmerge.lookahead_off` | Default, confirmed data |
| Gaps | `barmerge.gaps_off` | Forward-fills missing |
| Repaint risk | ✅ None | Historical bars stable |

---

## 11. Invariants

| Invariant | Enforcement |
|-----------|-------------|
| LTF always at global scope | Pine constraint |
| External gated by 4 layers | Conditional block L3067 |
| Budget ≤ 40 contexts | `budgetExceeded` check |
| Symbols ≤ 20 | `STATIC_EXT_maxSymbolsCap` |
| Invalid symbols → na | `ignore_invalid_symbol = true` |

---

## 12. Summary Table

| ID | Location | Symbol | TF | Expression | Gating | Contexts |
|----|----------|--------|-----|------------|--------|----------|
| LTF-1 | L2476 | current | INPUT_INTRABAR_TF | high | None | 1 |
| LTF-2 | L2477 | current | INPUT_INTRABAR_TF | low | None | 1 |
| EXT-1 | L1693-1694 | CSV | extTF | close | 4-layer | 0-20 |
| EXT-2 | L1696-1697 | CSV | extTF | [o,h,l,c,v] | 4-layer | 0-20 |
| **Total** | | | | | | **2-22** |

---

*Consolidated from Tasks 4 and 8. Supersedes prior request ledger versions.*
