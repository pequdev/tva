# Pine Script v6 Naming & Coding Standards

> **Version**: 1.0  
> **Last Updated**: 2024-12-22  
> **Applies To**: `PQ_FIBS.pine` and all PQ_* indicator modules

---

## Overview

This document defines the **semantic naming system** and **comment hygiene standards** for the PQ indicator suite. The goal is to make code self-documenting through consistent, meaningful identifiers—eliminating the need for verbose doc-block annotations (`@type`, `@field`, `@description`).

---

## 1. Prefix Taxonomy

All identifiers use a **prefix** to indicate scope, lifetime, and purpose.

| Prefix    | Scope / Lifetime                         | Example                              |
|-----------|------------------------------------------|--------------------------------------|
| `GLOBAL_` | Top-level state used across sections     | `GLOBAL_zigzag`, `GLOBAL_regime`     |
| `INPUT_`  | User inputs (`input.*` results)          | `INPUT_SHOW_ZIGZAG`, `INPUT_LEARNING_ENABLED` |
| `STATIC_` | Compile-time constants (immutable)       | `STATIC_LEARN_minSamples`, `STATIC_KEY_default` |
| `STATE_`  | Persistent `var`/`varip` state           | `STATE_tfDevThresholds`, `STATE_tfDepths` |
| `TMP_`    | Ephemeral per-bar temporaries            | `TMP_iMidPivot_hist`, `TMP_pEndBase_hist` |
| `BUF_`    | Arrays / circular buffers                | `BUF_entropyHistory`, `BUF_returns` |
| `REQ_`    | Request engine / HTF-LTF contexts        | `REQ_htf`, `REQ_htfAuto`, `REQ_htfUserParsed` |
| `SIG_`    | Signal booleans / regime gates           | `SIG_isEntryValid`, `SIG_shouldTrade` |
| `UI_`     | UI tokens, theme fields, rendering-only  | `UI_COLOR_green`, `UI_TEXT_long`, `UI_GROUP_pivot` |
| `DBG_`    | Debug flags and counters                 | `DBG_counterEntropyRuns`, `DBG_counterLearningRuns` |

### Usage Guidelines

1. **Always use the appropriate prefix** — no unprefixed global variables
2. **Prefix indicates behavior** — readers immediately know if a variable persists, is user-configurable, or is temporary
3. **Consistency over brevity** — `GLOBAL_cachedPivots` not `gCachedPivots`

---

## 2. Naming Conventions

### 2.1 Case Style

- **After prefix**: Use `lowerCamelCase`
- **Complete identifier**: `PREFIX_lowerCamelCase`

```pine
// ✓ Correct
GLOBAL_indexMidPivot
INPUT_LEARNING_ENABLED
STATIC_CONF_wrWeight
STATE_tfDevThresholds

// ✗ Incorrect
GLOBAL_IndexMidPivot    // Wrong: PascalCase after prefix
global_index_mid_pivot  // Wrong: snake_case
```

### 2.2 Boolean Naming

Booleans **must read as predicates** using these patterns:

| Pattern         | Usage                        | Example                           |
|-----------------|------------------------------|-----------------------------------|
| `*_is*`         | State check                  | `isHealthy`, `isExpired`, `isInZone` |
| `*_has*`        | Presence check               | `hasSignal`, `hasDivergenceEdge`  |
| `*_should*`     | Decision/action gate         | `shouldTrade`, `shouldRecalculate` |
| `*_can*`        | Capability check             | `canSpawn`, `canExecute`          |
| `*_needs*`      | Requirement flag             | `needsRecalculation`              |

```pine
// ✓ Correct
bool isPermtestSignificant = false
bool needsRecalculation = false
bool useTakeProfitAggressive = false

// ✗ Incorrect  
bool significant = false      // Ambiguous
bool recalc = false           // Verb, not predicate
bool aggressive_tp = false    // Not a predicate
```

### 2.3 Units in Names

Include units when meaningful for clarity:

| Suffix          | Meaning                      | Example                           |
|-----------------|------------------------------|-----------------------------------|
| `*Percentile`   | 0-100 percentile rank        | `atrPercentile`                   |
| `*Percent`      | Percentage value             | `bufferPercent`                   |
| `*Ratio`        | Dimensionless ratio          | `sharpeRatio`, `sortinoRatio`     |
| `*Bars`         | Bar count                    | `averageBarsHeld`, `barsSinceCreation` |
| `*Price`        | Price level                  | `stopLossPrice`, `takeProfitPrice` |
| `*Multiplier`   | Scaling factor               | `stopLossMultiplier`              |
| `*Rate`         | Rate/velocity                | `decayRate`, `winRate`            |
| `*Edge`         | Win rate differential        | `divergenceEdge`, `entropyEdge`   |

### 2.4 Abbreviations

**Allowed** (industry standard):
- `ATR` — Average True Range
- `RSI` — Relative Strength Index
- `HTF` — Higher Time Frame
- `LTF` — Lower Time Frame
- `TP` — Take Profit
- `SL` — Stop Loss
- `EV` — Expected Value
- `WR` — Win Rate
- `MAE` — Maximum Adverse Excursion
- `MFE` — Maximum Favorable Excursion
- `PF` — Profit Factor

**Avoid** (spell out):
- ~~`idx`~~ → `index`
- ~~`cnt`~~ → `count`
- ~~`buf`~~ → `buffer` (except in `BUF_` prefix)
- ~~`conf`~~ → `confidence` or `configuration`
- ~~`mult`~~ → `multiplier`
- ~~`aggr`~~ → `aggressive`
- ~~`cons`~~ → `conservative`

---

## 3. Type Definitions

### 3.1 Type Naming

Types use `PascalCase` without prefixes:

```pine
type ZigZagEngine
type LearningMetrics
type AlertState
type CachedPivots
```

### 3.2 Field Naming

Fields use `lowerCamelCase` **without prefixes** (the type name provides context):

```pine
type LearningMetrics
    float winRate = 0.55
    float stopLossMultiplier = 1.5
    float averageMaxAdverseExcursion = 0.0
    float divergenceEdge = 0.0
    bool  needsRecalculation = false
    bool  isHealthy = true
```

### 3.3 Field Name Consistency

When accessing type fields, use the **full field name** — never abbreviate:

```pine
// ✓ Correct
GLOBAL_learning.winRateHighVolatility
GLOBAL_learning.averageMaxAdverseExcursion
GLOBAL_alertState.stopLossPrice
GLOBAL_alertState.takeProfitConservative

// ✗ Incorrect (abbreviated)
GLOBAL_learning.wrHighVol        // Use winRateHighVolatility
GLOBAL_learning.avgMae           // Use averageMaxAdverseExcursion
GLOBAL_alertState.slPrice        // Use stopLossPrice
GLOBAL_alertState.tpCons         // Use takeProfitConservative
```

---

## 4. Comment Policy

### 4.1 What to Keep

Comments should **only** explain:

| Category                  | Example                                              |
|---------------------------|------------------------------------------------------|
| Algorithm steps           | `// Step 2: Normalize entropy to [0,1] range`        |
| Math abstractions         | `// Shannon entropy: H = -Σ p(x) log₂ p(x)`          |
| Invariants                | `// Invariant: buffer size never exceeds 500`        |
| Performance budgets       | `// Budget: max 40 dynamic requests per script`      |
| Non-repaint guarantees    | `// Pine v6: request.security with barmerge.lookahead.off` |
| Object limits             | `// max_lines_count=500, max_labels_count=500`       |
| Pine v6 semantics         | `// Pine v6: bool requires explicit true/false, no truthy` |

### 4.2 What to Delete

Remove these comment patterns:

```pine
// ✗ Delete: Doc-block annotations
// @type Description of type
// @field fieldName - Description
// @description Function description
// @param paramName Description
// @returns Description

// ✗ Delete: Obvious restatements
int count = 0  // Counter variable    ← redundant
bool isLong = true  // True if long   ← redundant

// ✗ Delete: Ad-hoc notes
// TODO: fix this later
// HACK: workaround for issue #123
// NOTE: John added this
```

### 4.3 Section Headers

Use consistent section header format:

```pine
// ═══════════════════════════════════════════════════════════════════════════════════════════════════════════════════
// SECTION NAME (ALL CAPS)
// ═══════════════════════════════════════════════════════════════════════════════════════════════════════════════════
```

### 4.4 Type Headers

For complex types, use a single compact header:

```pine
// ── Type: LearningMetrics (updated on trade completion; UI-free)
type LearningMetrics
    float winRate = 0.55
    // ... fields
```

---

## 5. Functional Groups

Code is organized into 10 semantic groups:

| #  | Group                    | Description                                           |
|----|--------------------------|-------------------------------------------------------|
| 1  | Architecture & Flow      | Core execution structure, indicator setup             |
| 2  | Inputs & UX              | All `input.*` declarations, UI groups                 |
| 3  | Data Requests & TFs      | `request.security`, timeframe handling                |
| 4  | Regime/Signals Core      | Entropy, Hurst, Z-score regime detection              |
| 5  | Stats Engines            | Learning, backtesting, Kelly, EV calculations         |
| 6  | Memory & Buffers         | Arrays, circular buffers, history tracking            |
| 7  | Execution & Performance  | Budget gates, lazy evaluation, optimization           |
| 8  | Rendering & Objects      | Lines, labels, boxes, polylines, tables               |
| 9  | Logging & Debug          | Debug counters, diagnostic outputs                    |
| 10 | Noise / Leftovers        | Deprecated code, cleanup candidates                   |

---

## 6. Current Codebase Statistics

As of 2024-12-22 (`PQ_FIBS.pine`):

| Metric                    | Count   |
|---------------------------|---------|
| Total Lines               | 5,590   |
| Type Definitions          | 20      |
| `GLOBAL_` references      | 741     |
| `INPUT_` references       | 410     |
| `STATIC_` references      | 554     |
| `STATE_` references       | 54      |
| `UI_` references          | 745     |
| `REQ_` references         | 11      |
| `DBG_` references         | 19      |
| `TMP_` references         | 24      |
| `BUF_` references         | 29      |

### Types Defined

```
SymbolicEntropyState    DyadicHurstState       LevelLog
PivotContext            ZigZagEngine           ZigZagVisualBuffer
RegimeState             ExternalContextState   LearningMetrics
PositionVisual          ProjectionState        FibLevel
SetupRecord             CircularBufferSetupRecord
BacktestTrade           BacktestStats          BacktestEngine
SetupState              AlertState             CachedPivots
```

---

## 7. Common Mistakes & Fixes

### 7.1 Field Name Mismatches

| Wrong (Abbreviated)              | Correct (Full Name)                    |
|----------------------------------|----------------------------------------|
| `GLOBAL_learning.divEdge`        | `GLOBAL_learning.divergenceEdge`       |
| `GLOBAL_learning.slMult`         | `GLOBAL_learning.stopLossMultiplier`   |
| `GLOBAL_learning.avgBars`        | `GLOBAL_learning.averageBarsHeld`      |
| `GLOBAL_learning.wrLong`         | `GLOBAL_learning.winRateLong`          |
| `GLOBAL_learning.avgMae`         | `GLOBAL_learning.averageMaxAdverseExcursion` |
| `GLOBAL_alertState.slHit`        | `GLOBAL_alertState.stopLossHit`        |
| `GLOBAL_alertState.tpPrice`      | `GLOBAL_alertState.takeProfitPrice`    |
| `GLOBAL_alertState.inZone`       | `GLOBAL_alertState.isInZone`           |
| `GLOBAL_alertState.expired`      | `GLOBAL_alertState.isExpired`          |

### 7.2 Missing Prefixes

```pine
// ✗ Wrong: unprefixed global
htf = timeframe.period == '1' ? '240' : 'D'

// ✓ Correct: with REQ_ prefix
REQ_htf = timeframe.period == '1' ? '240' : 'D'
```

### 7.3 Type Field Inconsistency

```pine
// ✗ Wrong: mismatched field names in type vs usage
type CachedPivots
    int GLOBAL_indexMidPivot2 = na    // Prefix inside type
    
GLOBAL_cachedPivots.iMidPivot2        // Different name in usage

// ✓ Correct: consistent naming
type CachedPivots
    int iMidPivot2 = na               // No prefix inside type
    
GLOBAL_cachedPivots.iMidPivot2        // Same name in usage
```

---

## 8. Quality Gates

Before merging any code changes, verify:

- [ ] **No `@type/@field/@description` blocks remain**
- [ ] **All variables have appropriate prefixes**
- [ ] **Type field names match their usage sites exactly**
- [ ] **Boolean names read as predicates**
- [ ] **No abbreviated field names in type accesses**
- [ ] **Script compiles without errors**
- [ ] **Logic behavior unchanged** (only naming/comments differ)

---

## 9. Tooling Commands

### Verify No Errors
```bash
# Check for Pine Script errors (use TradingView compiler)
```

### Search for Mismatched Fields
```bash
# Find abbreviated field references
grep -E 'GLOBAL_learning\.(slMult|avgBars|wrLong|divEdge)' PQ_FIBS.pine
grep -E 'GLOBAL_alertState\.(slHit|tpHit|inZone|expired)' PQ_FIBS.pine
```

### Bulk Field Rename (sed)
```bash
sed -i '' \
  -e 's/GLOBAL_alertState\.slHit/GLOBAL_alertState.stopLossHit/g' \
  -e 's/GLOBAL_alertState\.tpHit/GLOBAL_alertState.takeProfitHit/g' \
  PQ_FIBS.pine
```

### Count Prefix Usage
```bash
grep -c "GLOBAL_" PQ_FIBS.pine
grep -c "INPUT_" PQ_FIBS.pine
grep -c "STATIC_" PQ_FIBS.pine
```

---

## 10. Version History

| Version | Date       | Changes                                              |
|---------|------------|------------------------------------------------------|
| 1.0     | 2024-12-22 | Initial naming standards documentation               |

---

## References

- [Pine Script v6 Language Reference](https://www.tradingview.com/pine-script-reference/v6/)
- [PQ_FIBS Performance Guide](./PQ_FIBS_PERFORMANCE_GUIDE.md)
- [Pine Script v6 Coding Requirements](./PINESCRIPT_V6_CODING_REQUIREMENTS.md)
