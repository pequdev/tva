# PQ_FIBS.pine — Repaint Integrity Audit

> **Version**: 1.0  
> **Generated**: 2025-12-22  
> **Source**: `PQ_FIBS.pine` (5,591 lines)  
> **Pine Version**: v6

---

## 1. Executive Summary

| Request Type | Repaint Risk | Mitigation | Signal Impact |
|--------------|--------------|------------|---------------|
| **LTF Intrabar Arrays** | ⚠️ Medium (by design) | Resolver boundary | Backtest-only |
| **External Context (close)** | ✅ Low | Default `lookahead_off` | Gating-only |
| **External Context (OHLCV)** | ✅ Low | Default `lookahead_off` | Gating-only |

**Overall Assessment:** The script has appropriate repaint controls for its intended use cases. Intrabar arrays affect only backtest trade resolution (not signal generation), and external context affects only regime gating (not entry signals).

---

## 2. Pine v6 Request Defaults

### 2.1 `request.security()` Defaults

Per official Pine v6 documentation:

| Parameter | Default Value | Behavior |
|-----------|---------------|----------|
| `lookahead` | `barmerge.lookahead_off` | Uses last **confirmed** value from requested TF |
| `gaps` | `barmerge.gaps_off` | Fills missing data with last valid value |
| `ignore_invalid_symbol` | `false` | Causes error on invalid symbol |

**Doc Reference:**
> "When `lookahead` is `barmerge.lookahead_off` (default), the function returns the last confirmed value from the requested timeframe. On realtime bars, this value can change until the bar closes."

### 2.2 `request.security_lower_tf()` Behavior

| Aspect | Behavior |
|--------|----------|
| Return type | `array<type>` |
| Array length | Varies during forming bar, stable on historical bars |
| Element order | Chronological (index 0 = first intrabar) |
| Real-time behavior | Array grows as intrabars complete |

**Doc Reference:**
> "The `request.security_lower_tf()` function returns an array containing one element per intrabar within the chart bar. The array's length may vary on the forming bar."

---

## 3. External Context Repaint Analysis

### 3.1 Request Options Audit

**Current Implementation (L1693-L1697):**
```pine
f_reqExtClose(string sym, string tf) =>
    request.security(sym, tf, close, ignore_invalid_symbol = true)

f_reqExtOHLCV(string sym, string tf) =>
    request.security(sym, tf, [open, high, low, close, volume], ignore_invalid_symbol = true)
```

| Parameter | Current | Explicit? | Default Value | Risk |
|-----------|---------|-----------|---------------|------|
| `lookahead` | Not specified | ❌ Implicit | `barmerge.lookahead_off` | None (safe default) |
| `gaps` | Not specified | ❌ Implicit | `barmerge.gaps_off` | None (safe default) |
| `ignore_invalid_symbol` | `true` | ✅ Explicit | `false` | Intentional (returns `na`) |

### 3.2 Lookahead Behavior

**With `lookahead_off` (current default):**
- On historical bars: Returns the last value from the completed bar of the requested TF
- On realtime bars: Returns the current (possibly incomplete) value

**Repaint Assessment:**
> External requests do NOT introduce future-looking bias. They use `lookahead_off` which is the safe default.

### 3.3 Consumer Path Impact

**External context flows to:**
```
f_reqExtClose() / f_reqExtOHLCV()
    └── GLOBAL_extContext.closes / opens / highs / lows / volumes
        └── GLOBAL_extContext.validCount (count of non-na)
            └── GLOBAL_extContext.extOk (true if validCount > 0)
                └── _regimeGatesPassed (L3843)
```

**Signal Impact Analysis:**

| Path | Affects Signals? | Affects Entry? | Affects Backtest? |
|------|------------------|----------------|-------------------|
| `extOk` → `_regimeGatesPassed` | ✅ Indirect | ⚠️ Gating only | ⚠️ Gating only |

**Critical Finding:**
> `extOk` participates in `_regimeGatesPassed` which gates zone activation.
> When `INPUT_EXT_ENABLED = false` (default), `extOk = true` and has no effect.
> When enabled with valid symbols, `extOk` can block signals if all external requests return `na`.

### 3.4 `na` Propagation Safety

| Scenario | `validCount` | `extOk` | Effect |
|----------|--------------|---------|--------|
| External disabled | N/A | `true` | No gating |
| 0 valid symbols | 0 | `false` | Blocks signals |
| 1+ valid symbols | ≥1 | `true` | No gating |
| All `na` returns | 0 | `false` | Blocks signals |

**Code Evidence (L3106-L3111):**
```pine
bool _extOk = true
if INPUT_EXT_ENABLED
    if GLOBAL_extContext.gated and array.size(GLOBAL_extContext.closes) > 0
        _extOk := GLOBAL_extContext.validCount > 0
GLOBAL_extContext.extOk := _extOk
```

---

## 4. Intrabar LTF Repaint Analysis

### 4.1 Request Declaration

**Global Scope (L2474-L2477):**
```pine
// request.security_lower_tf MUST be at global scope (not inside conditionals)
array<float> REQ_ltfHighs = request.security_lower_tf(syminfo.tickerid, INPUT_INTRABAR_TF, high)
array<float> REQ_ltfLows  = request.security_lower_tf(syminfo.tickerid, INPUT_INTRABAR_TF, low)
```

### 4.2 Inherent Repaint Characteristics

| Aspect | Historical Bars | Realtime/Forming Bar |
|--------|-----------------|----------------------|
| Array length | Fixed | Grows as intrabars complete |
| Element values | Stable | Last element may change |
| Determinism | ✅ Deterministic | ⚠️ Non-deterministic |

**Doc Reference:**
> "On realtime bars, the array returned by `request.security_lower_tf()` will contain progressively more elements as intrabars complete. The last element represents the current (incomplete) intrabar."

### 4.3 Resolver Boundary Enforcement

**Consumption Point (L3936-L3946):**
```pine
if INPUT_INTRABAR_ENABLED and array.size(REQ_ltfHighs) > 0
    bool conservative = INPUT_INTRABAR_TIEBREAK == UI_OPT_conservative
    [ltf_outcome, ltf_resolved, hit_idx] = f_resolveIntrabar(
        REQ_ltfHighs, REQ_ltfLows, 
        sim_is_long, 
        GLOBAL_backtest.current.sl_price, 
        GLOBAL_backtest.current.tp_price, 
        conservative
    )
```

**Boundary Verification:**

| Check | Status | Evidence |
|-------|--------|----------|
| Used only in backtest resolution | ✅ | Consumer at L3939 is inside backtest loop |
| Not used in signal generation | ✅ | No refs in zone detection or entry logic |
| Not used in regime classification | ✅ | No refs in entropy/hurst/zscore |
| Gated by `INPUT_INTRABAR_ENABLED` | ✅ | L3936: `if INPUT_INTRABAR_ENABLED` |

### 4.4 Resolver Function Semantics

**Function (L919-L955):**
```pine
f_resolveIntrabar(array<float> ltf_highs, array<float> ltf_lows, 
                  bool is_long, float sl_price, float tp_price, 
                  bool conservative_tiebreak) =>
    // ... iterates chronologically through intrabars
    // Returns: [outcome, ltf_resolved, hit_index]
```

**Resolution Logic:**
1. Iterate through LTF bars chronologically (index 0 = earliest)
2. For each intrabar, check if SL or TP is hit
3. First hit determines outcome
4. If both hit in same intrabar → apply tie-break policy

**Tie-Break Policy:**

| Policy | `INPUT_INTRABAR_TIEBREAK` | Outcome |
|--------|---------------------------|---------|
| Conservative | `"Conservative"` | SL assumed first (outcome = -1) |
| Optimistic | `"Optimistic"` | TP assumed first (outcome = 1) |

### 4.5 Realtime Bar Considerations

**Issue:** On the forming bar, the intrabar array is incomplete.

**Mitigation in Context:**
1. Backtest resolution typically processes **completed** trades
2. The resolver runs when a trade outcome is being determined
3. For historical bars, data is stable and deterministic
4. For realtime bars, the result may differ from final historical result

**Design Intent (per existing comments):**
> "Intrabar precision improves TP/SL ordering without corrupting historical integrity."

---

## 5. Repaint Risk Matrix

| Request | Location | Repaint Risk | Consumer | Signal Path | Mitigation |
|---------|----------|--------------|----------|-------------|------------|
| `REQ_ltfHighs` | L2476 | ⚠️ Medium | `f_resolveIntrabar` | Backtest only | Boundary isolation |
| `REQ_ltfLows` | L2477 | ⚠️ Medium | `f_resolveIntrabar` | Backtest only | Boundary isolation |
| `f_reqExtClose` | L1693-L1694 | ✅ Low | `GLOBAL_extContext` | Gating only | Default `lookahead_off` |
| `f_reqExtOHLCV` | L1696-L1697 | ✅ Low | `GLOBAL_extContext` | Gating only | Default `lookahead_off` |

---

## 6. Explicit Options Recommendation

### 6.1 Current State (Implicit Defaults)

The external request helpers use implicit defaults for `lookahead` and `gaps`:

```pine
// Current (implicit)
request.security(sym, tf, close, ignore_invalid_symbol = true)
```

### 6.2 Proposed Enhancement (Explicit Defaults)

Making options explicit would improve code clarity without changing behavior:

```pine
// Proposed (explicit, behavior-identical)
request.security(sym, tf, close, 
    lookahead = barmerge.lookahead_off,
    gaps = barmerge.gaps_off,
    ignore_invalid_symbol = true
)
```

**Trade-off Analysis:**

| Aspect | Implicit (Current) | Explicit (Proposed) |
|--------|-------------------|---------------------|
| Behavior | Uses Pine defaults | Identical to Pine defaults |
| Readability | Requires knowing defaults | Self-documenting |
| Maintainability | Risk if defaults change | Future-proof |
| Code length | Shorter | Slightly longer |

**Recommendation:** 
> Consider making `lookahead` and `gaps` explicit in a future maintenance pass.
> This is NOT a required change for Task 4 (no functional changes allowed).

---

## 7. Timeframe Mismatch Handling

### 7.1 Intrabar TF Mismatch

**Scenario:** `INPUT_INTRABAR_TF` is higher than or equal to chart timeframe.

| Chart TF | LTF Setting | Result |
|----------|-------------|--------|
| 5m | "1" | Normal (1m < 5m) |
| 5m | "5" | Edge case (equal TF) |
| 1m | "5" | Invalid (5m > 1m) |

**Behavior per Pine v6 docs:**
> When the requested timeframe is higher than or equal to the chart timeframe, `request.security_lower_tf()` returns an array with a single element (no intrabar granularity).

**Impact:** Resolver degrades to single-value resolution, but no error.

### 7.2 External TF Mismatch

**Scenario:** `INPUT_EXT_TF_CUSTOM` is invalid.

| TF String | Validity | Result |
|-----------|----------|--------|
| `"60"` | ✅ Valid | 1-hour data |
| `"D"` | ✅ Valid | Daily data |
| `"invalid"` | ❌ Invalid | Request may fail or return `na` |
| `""` (empty) | ❌ Invalid | Request may fail or return `na` |

**Mitigation:**
- `ignore_invalid_symbol = true` handles bad symbols gracefully
- Invalid TF strings are handled by Pine runtime (may return `na` or error)
- Current code has no explicit TF validation

**Risk Assessment:** Low — invalid TF will cause `na` returns, which propagate to `extOk = false`.

---

## 8. History vs Live Parity

### 8.1 External Context Parity

| Aspect | Historical | Realtime | Parity |
|--------|------------|----------|--------|
| `lookahead_off` behavior | Last confirmed HTF value | Current (updating) value | ⚠️ Different |
| `gaps_off` behavior | Filled with last value | Same | ✅ Same |
| `ignore_invalid_symbol` | `na` on invalid | Same | ✅ Same |

**Note:** With `lookahead_off`, realtime values may differ from final historical values until the HTF bar closes. This is expected Pine behavior and not a bug.

### 8.2 Intrabar Parity

| Aspect | Historical | Realtime | Parity |
|--------|------------|----------|--------|
| Array length | Fixed (all intrabars) | Growing | ⚠️ Different |
| Array values | Stable | Last element updating | ⚠️ Different |
| Resolver outcome | Deterministic | May change | ⚠️ Different |

**Design Acknowledgment:**
> Intrabar resolution is inherently different on historical vs realtime bars. This is documented as an intended trade-off for improved backtest accuracy.

---

## 9. Signal Path Isolation Verification

### 9.1 What Affects Entry Signals?

Entry signals in PQ_FIBS are determined by:
1. ZigZag pivot detection
2. Golden Pocket zone calculation
3. Regime classification (entropy, hurst, zscore)
4. Learning engine parameters

### 9.2 Request Output Path Verification

| Request Output | Affects Pivots? | Affects Zones? | Affects Regime? | Affects Learning? |
|----------------|-----------------|----------------|-----------------|-------------------|
| `REQ_ltfHighs` | ❌ | ❌ | ❌ | ❌ |
| `REQ_ltfLows` | ❌ | ❌ | ❌ | ❌ |
| `extContext.closes` | ❌ | ❌ | ⚠️ Via `extOk` | ❌ |
| `extContext.extOk` | ❌ | ❌ | ✅ Gating only | ❌ |

**Grep Verification:**
- `REQ_ltf` appears only in L2476, L2477, L3936, L3939
- L3936-L3939 are inside backtest resolution block
- No references in zigzag, zone, or regime calculation code

---

## 10. Compliance Summary

| Requirement | Status | Evidence |
|-------------|--------|----------|
| All request outputs documented | ✅ | See §3, §4 |
| Signal paths verified non-repainting | ✅ | External uses `lookahead_off` default |
| Intrabar boundary enforced | ✅ | Only consumed in backtest resolution |
| `na` propagation documented | ✅ | See §3.4 |
| History/live parity documented | ✅ | See §8 |
| Lookahead behavior verified | ✅ | Default = `lookahead_off` (safe) |
| No future-looking bias in signals | ✅ | External context is gating-only |

---

## 11. Doc References

### 11.1 Pine v6 Documentation Sources

| Topic | Key Finding |
|-------|-------------|
| `request.security()` lookahead | Default is `barmerge.lookahead_off` — confirmed values only |
| `request.security()` gaps | Default is `barmerge.gaps_off` — fills with last value |
| `request.security_lower_tf()` | Returns array, length varies on forming bar |
| Global scope requirement | LTF requests must be at global scope |
| Repainting definition | "Behavior differs between historical and realtime bars" |

### 11.2 Context7 Pine v6 Doc References Used

- "Managing Future Data Access with lookahead in request.security()"
- "request.security_lower_tf function signature"
- "Decompose Intrabar Price Changes with request.security_lower_tf"
- "Non-Repainting Higher-Timeframe Data Request"
- "Avoiding LTF repainting demo"

---

*Document generated as part of Task 4: Data Requests & Timeframe Resolver Audit*
