# PQ_FIBS.pine — Budget Model & Gates

> **Version**: 1.0  
> **Generated**: 2025-12-22  
> **Source**: `PQ_FIBS.pine` (5,591 lines)  
> **Pine Version**: v6

---

## 1. Executive Summary

| Metric | Value | Location |
|--------|-------|----------|
| **Budget Cap** | 40 contexts | L558: `STATIC_EXT_maxContexts` |
| **Base Contexts (Fixed)** | 2 | LTF high + low |
| **Max External Symbols** | 20 | L557: `STATIC_EXT_maxSymbolsCap` |
| **Default External Symbols** | 5 | L559: `STATIC_EXT_defaultSymbols` |
| **Worst-Case Total** | 22 | 2 + 20 |
| **Budget Safety Margin** | 18 | 40 - 22 |

---

## 2. Budget Formula

### 2.1 Context Count Formula

```
TotalContexts = BaseContexts + ExternalContexts

Where:
  BaseContexts     = 2  (REQ_ltfHighs + REQ_ltfLows, always present)
  ExternalContexts = N  (N = min(parsed_symbols, INPUT_EXT_MAX_SYMBOLS))
                         when INPUT_EXT_ENABLED = true, else 0
```

### 2.2 Implementation in Code

**Budget Estimation (L3081-L3082):**
```pine
int _estimatedContexts = 2 + _symCount
if _estimatedContexts > STATIC_EXT_maxContexts
    GLOBAL_extContext.budgetExceeded := true
```

| Variable | Computation | Notes |
|----------|-------------|-------|
| `_symCount` | `array.size(_parsedSymbols)` | After parsing CSV and applying max cap |
| `_estimatedContexts` | `2 + _symCount` | Base (2) + external symbols |
| `STATIC_EXT_maxContexts` | `40` | Hard-coded budget cap |

---

## 3. Budget Constants

### 3.1 Constant Definitions (L554-L568)

| Constant | Value | Purpose | Location |
|----------|-------|---------|----------|
| `STATIC_EXT_maxSymbolsCap` | 20 | Hard cap on external symbols | L557 |
| `STATIC_EXT_maxContexts` | 40 | TradingView unique request limit | L558 |
| `STATIC_EXT_defaultSymbols` | 5 | Default max symbols | L559 |

### 3.2 Input Constraints

| Input | Default | Min | Max | Location |
|-------|---------|-----|-----|----------|
| `INPUT_EXT_MAX_SYMBOLS` | 5 | 1 | 20 | L764 |

**Input Definition (L764):**
```pine
INPUT_EXT_MAX_SYMBOLS = input.int(STATIC_EXT_defaultSymbols, '   ↳ Max Symbols', 
                                   minval=1, maxval=STATIC_EXT_maxSymbolsCap, ...)
```

---

## 4. Budget Enforcement Mechanism

### 4.1 Gate Hierarchy

```
Layer 1: INPUT_EXT_ENABLED (master toggle)
    │
    ▼
Layer 2: f_computeExtGate() (regime-based gating)
    │
    ▼
Layer 3: budgetExceeded check
    │
    ▼
Layer 4: Symbol parsing (bounded by INPUT_EXT_MAX_SYMBOLS)
    │
    ▼
[Execute Requests]
```

### 4.2 Code Flow (L3063-L3085)

```pine
// L3063: Compute gating condition
bool _extGated = INPUT_EXT_ENABLED and f_computeExtGate(INPUT_EXT_GATE_MODE, ...)

// L3067: Execute only if gated and within budget
if _extGated and not GLOBAL_extContext.budgetExceeded
    // L3069: Parse symbols with max cap
    array<string> _parsedSymbols = f_parseSymbols(INPUT_EXT_SYMBOLS, INPUT_EXT_MAX_SYMBOLS)
    int _symCount = array.size(_parsedSymbols)
    
    // L3078: Resolve timeframe
    string _extTf = f_getExtTimeframe(INPUT_EXT_TF_MODE, INPUT_EXT_TF_CUSTOM)
    
    // L3081-L3083: Budget check
    int _estimatedContexts = 2 + _symCount
    if _estimatedContexts > STATIC_EXT_maxContexts
        GLOBAL_extContext.budgetExceeded := true
    else
        // L3085: Record used contexts
        GLOBAL_extContext.usedContexts := _symCount
        // L3088-L3103: Execute requests in bounded loop
```

### 4.3 Symbol Parsing Safety

**Parser Function (L1672-L1679):**
```pine
f_parseSymbols(string csv, int max_count) =>
    array<string> result = array.new<string>(0)
    if str.length(str.trim(csv)) > 0
        array<string> parts = str.split(csv, ',')
        for i = 0 to math.min(array.size(parts) - 1, max_count - 1)
            string sym = str.trim(array.get(parts, i))
            if str.length(sym) > 0
                array.push(result, sym)
    result
```

**Safety Properties:**
| Property | Implementation | Status |
|----------|----------------|--------|
| Empty CSV handling | `str.length(str.trim(csv)) > 0` | ✅ |
| Whitespace trimming | `str.trim()` on each symbol | ✅ |
| Max count enforcement | `math.min(parts.size - 1, max_count - 1)` | ✅ |
| Empty entries skipped | `str.length(sym) > 0` check | ✅ |
| Duplicate handling | ⚠️ Not implemented | Potential context waste |

---

## 5. Worst-Case Analysis

### 5.1 Scenario Matrix

| Scenario | LTF Contexts | External Contexts | Total | Within Budget |
|----------|--------------|-------------------|-------|---------------|
| Default (external OFF) | 2 | 0 | 2 | ✅ (5% of cap) |
| External ON, 5 symbols | 2 | 5 | 7 | ✅ (17.5% of cap) |
| External ON, 10 symbols | 2 | 10 | 12 | ✅ (30% of cap) |
| External ON, 20 symbols | 2 | 20 | 22 | ✅ (55% of cap) |
| External ON, 38 symbols* | 2 | 38 | 40 | ✅ (100% of cap) |

*Note: `INPUT_EXT_MAX_SYMBOLS` is capped at 20, so 38 is not achievable via UI.

### 5.2 Fields Mode Impact

**Critical Finding:** The script uses EITHER `f_reqExtClose` OR `f_reqExtOHLCV`, never both simultaneously.

```pine
// L3091-L3103
if INPUT_EXT_FIELDS == STATIC_EXT_fieldsOHLCV
    [_o, _h, _l, _c, _v] = f_reqExtOHLCV(_sym, _extTf)  // Tuple expression
    ...
else
    float _c = f_reqExtClose(_sym, _extTf)              // Single expression
    ...
```

**Implication:** Mode switching doesn't accumulate contexts — only one expression shape is used per execution.

---

## 6. Budget State Tracking

### 6.1 ExternalContextState Fields

| Field | Type | Purpose | Location |
|-------|------|---------|----------|
| `budgetExceeded` | `bool` | Flag when estimated > cap | L2161 |
| `usedContexts` | `int` | Count of external contexts used | L2162 |

### 6.2 State Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│  Bar Start                                                  │
│  └── GLOBAL_extContext.reset() [L3059]                     │
│      ├── budgetExceeded = false (initialized)              │
│      └── usedContexts = 0 (initialized)                    │
├─────────────────────────────────────────────────────────────┤
│  Budget Check [L3081-L3083]                                 │
│  └── if _estimatedContexts > STATIC_EXT_maxContexts        │
│      └── budgetExceeded := true                            │
├─────────────────────────────────────────────────────────────┤
│  Request Execution [L3085]                                  │
│  └── if not budgetExceeded                                 │
│      └── usedContexts := _symCount                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. Budget Safety Properties

### 7.1 Guaranteed Properties

| Property | Enforcement | Evidence |
|----------|-------------|----------|
| **Max external symbols ≤ 20** | Input `maxval` constraint | L764: `maxval=STATIC_EXT_maxSymbolsCap` |
| **Budget check before execution** | Conditional guard | L3067: `not GLOBAL_extContext.budgetExceeded` |
| **Base contexts counted** | Formula includes `+ 2` | L3081: `_estimatedContexts = 2 + _symCount` |
| **No request loops without bounds** | Loop bounded by `_symCount` | L3088: `for i = 0 to _symCount - 1` |

### 7.2 Potential Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Duplicate symbols in CSV | Low | Wastes contexts but within cap |
| Invalid TF string | Low | `na` returns, no context explosion |
| Mode churn (CLOSE↔OHLCV) | None | Only one mode active per bar |

---

## 8. Request Gating Functions

### 8.1 Regime-Based Gating

**Function (L1684-L1691):**
```pine
f_computeExtGate(string gate_mode, int entropy_regime, float hurst, float hurst_meanrev) =>
    switch gate_mode
        STATIC_EXT_gateAlways => true
        STATIC_EXT_gateEntropy => entropy_regime == STATIC_ENTROPY_regimeDisordered
        STATIC_EXT_gateHurst => not na(hurst) and hurst < hurst_meanrev
        STATIC_EXT_gateCustom => true
        => true
```

| Gate Mode | Condition | Effect |
|-----------|-----------|--------|
| `ALWAYS` | Always true | Requests every bar |
| `ON_ENTROPY_DISORDER` | Entropy regime == disordered | Requests only in high entropy |
| `ON_LOW_HURST` | Hurst < mean-revert threshold | Requests only in mean-reverting regimes |
| `CUSTOM` | Always true (placeholder) | Same as ALWAYS |

### 8.2 Gate Input

| Input | Default | Options | Location |
|-------|---------|---------|----------|
| `INPUT_EXT_GATE_MODE` | `"ALWAYS"` | ALWAYS, ON_ENTROPY_DISORDER, ON_LOW_HURST, CUSTOM | L766 |

---

## 9. Budget Calculation Examples

### 9.1 Example: Default Configuration

```
INPUT_EXT_ENABLED = false (default)
INPUT_EXT_MAX_SYMBOLS = 5 (default, but irrelevant)

Calculation:
  BaseContexts = 2
  ExternalContexts = 0 (disabled)
  TotalContexts = 2

Result: 2 contexts used (5% of budget)
```

### 9.2 Example: Maximum Configuration

```
INPUT_EXT_ENABLED = true
INPUT_EXT_SYMBOLS = "A,B,C,D,E,F,G,H,I,J,K,L,M,N,O,P,Q,R,S,T" (20 symbols)
INPUT_EXT_MAX_SYMBOLS = 20

Calculation:
  BaseContexts = 2
  ExternalContexts = 20
  TotalContexts = 22

Result: 22 contexts used (55% of budget)
```

### 9.3 Example: Budget Exceeded (Hypothetical)

```
If STATIC_EXT_maxSymbolsCap were 50 and user entered 50 symbols:

Calculation:
  BaseContexts = 2
  ExternalContexts = 50
  _estimatedContexts = 52

Check: 52 > 40 (STATIC_EXT_maxContexts)
Action: budgetExceeded := true
Result: No external requests executed
```

**Note:** This is a purely hypothetical scenario that assumes `STATIC_EXT_maxSymbolsCap = 50` for illustration; under the actual current constraint (`STATIC_EXT_maxSymbolsCap = 20`), this situation cannot occur.

---

## 10. Compliance Summary

| Requirement | Status | Evidence |
|-------------|--------|----------|
| Budget formula explicit | ✅ | L3081: `2 + _symCount` |
| Worst-case documented | ✅ | Max 22 contexts (2 + 20) |
| Budget cap enforced | ✅ | L3082-L3083: comparison with `STATIC_EXT_maxContexts` |
| Gating prevents execution when exceeded | ✅ | L3067: `not GLOBAL_extContext.budgetExceeded` |
| Fields mode doesn't multiply contexts | ✅ | L3091: `if/else` exclusive branching |
| Base contexts (LTF) always counted | ✅ | L3081: `+ 2` in formula |

---

*Document generated as part of Task 4: Data Requests & Timeframe Resolver Audit*
