# PQ_FIBS — Task Plan (Audit → Finalisation → Consolidation)
Cross-referenced with the attached **Technical Audit and Optimization Protocol** (constraints + science-driven method) and the current `PQ_FIBS.pine` implementation (line anchors approximate).

---

## Task 1 — Codebase Index + Dependency Flow Map (Baseline Truth)
**Why (protocol):** Everything else depends on a deterministic “Inputs → Requests → Stats → Regime → Signals → Execution → Rendering” graph.  
**Current code anchors:** `indicator()` (L8), major type cluster (L1164–L2350), intrabar requests (L2472), debug overlays (L5415+).

**Deliverables**
- Section index + dependency flow map for *this* file (no assumptions).
- “Upstream immunity” check: verify rendering/logging never mutates gating/signal state.
- A Functional Matrix v1 (modules, inputs/outputs, exec freq, complexity, failure modes).

**Exit criteria**
- Every major subsystem has: anchor, inputs, outputs, downstream consumers, gating condition(s).

---

## Task 2 — Architecture & Runtime Constraints Audit (Hard Limits)
**Why (protocol):** v6 limits define the feasible design space (requests, arrays, loops, objects).  
**Current code anchors:** runtime header + script config (L1–L12), perf controller (L495), DS/theme block (L11+).

**Deliverables**
- Verify script settings are aligned with intended runtime:
  - `dynamic_requests=true`, `max_*_count`, `max_bars_back`, `calc_bars_count`.
- Audit global constants and “static vs state” separation:
  - confirm heavy initializations happen once (`var`) and not per bar.
- Compile hygiene scan: type strictness, implicit casts, unused vars.

**Exit criteria**
- A “Hard Limits Checklist” mapped to code (what enforces it, where it can break).

---

## Task 3 — Inputs & UX: Validation + Consolidation (No Feature Changes)
**Why (protocol):** Robust inputs prevent deep-stack defensive coding and runtime surprises.  
**Current code anchors:** large input clusters (e.g., external context inputs L759+, learning controls L36842, intrabar precision inputs L38527+).

**Deliverables**
- Input inventory grouped by intent (UX groups → internal subsystems).
- Validate ranges + invariants (lengths > 0, windows bounded, max symbols sane).
- Identify duplicated inputs / derived constants and propose consolidation (without new UX).

**Exit criteria**
- Input→Subsystem mapping + a “safe defaults” note for performance-critical toggles (debug, external requests).

---

## Task 4 — Data Requests & Timeframe Resolver: Budget + Repaint Integrity
**Why (protocol):** Requests are the #1 cost + repaint risk. Must obey 40 contexts and lookahead safety.  
**Current code anchors:**
- Intrabar arrays: `request.security_lower_tf` global scope (L2472–L2479)
- External context requests: `f_reqExtClose`, `f_reqExtOHLCV` (L91650–L92050)
- External context inputs/gating (L759+), `ExternalContextState` (L2155)

**Deliverables**
- Request inventory: every `request.*` call, context definition, gating, and dedupe strategy.
- Budget model:
  - Worst-case unique contexts = `externalSymbolsCount * fieldsModeContexts + intrabarContexts`.
- Repaint audit:
  - `lookahead` usage (explicit or default), offsets, and whether any request-return is used in signals vs UI-only.

**Exit criteria**
- “Request Context Ledger” + PASS/FAIL for: budget, gating, repaint safety, intrabar resolver isolation.

---

## Task 5 — Boolean Semantics + Lazy Evaluation Wiring (v6 Correctness)
**Why (protocol):** v6 strict bool + short-circuiting enables big savings and prevents `na` logic drift.  
**Current code anchors:** lazy eval header (L1038), regime gates (`RegimeState` L2131), gating helpers (e.g., `f_readyForRegime`, `perf_shouldComputeHeavy` in function list).

**Deliverables**
- Boolean audit:
  - every `var bool` init must be explicit `false`
  - remove any “na-as-false” patterns (or guard them with `not na()` + explicit fallback).
- Lazy-eval ordering audit:
  - ensure expensive calls are RHS of `and/or` behind cheap guards.
- Produce a “Signal Gate Tree” (one-page) describing evaluation order and dependencies.

**Exit criteria**
- No boolean can become `na` at runtime.
- Heavy computations are provably gated by cheap predicates on most bars.

---

## Task 6 — Stats Engines Audit: Complexity + Consolidation (No New Math)
**Why (protocol):** Loops + heavy math trip 500ms-per-bar limit; built-ins outperform custom loops.  
**Current code anchors:**
- `SymbolicEntropyState` (L1164) + entropy functions (`f_shannonEntropy`, `f_rollingEntropy`)
- `DyadicHurstState` (L1450) + hurst pipeline (`f_computeRS`, `f_rollingHurst`)
- Z-score/momentum areas (inputs around L43775+)

**Deliverables**
- Hotspot map:
  - list all `for` loops (count, bounds, frequency), classify O(1)/O(window)/O(N²).
- Verify “science-driven” formulas match protocol (entropy stable form, dyadic R/S for hurst).
- Consolidation candidates:
  - shared intermediate series (e.g., log returns, ATR-normalization) computed once per bar and reused.

**Exit criteria**
- Each engine has an explicit complexity profile and a gating condition.
- Redundant recomputation eliminated (behavior-preserving).

---

## Task 7 — Memory & Buffers: Persistence, Bounds, Negative Indexing
**Why (protocol):** `var` persistence + bounded buffers prevent GC churn and 100k array blowups.  
**Current code anchors:** circular buffer helpers (L996), buffer fields in UDTs (entropy/hurst/learning types), `BUF_*` globals (e.g., `BUF_fibLevels`).

**Deliverables**
- Array lifecycle audit:
  - identify any per-bar `array.new` in hot paths vs init blocks.
- Boundedness proof:
  - confirm all rolling structures enforce max window sizes (push+shift or circular index).
- Replace “size()-1” last-access patterns with v6 negative indexing where safe/readable.

**Exit criteria**
- No unbounded growth paths remain.
- Arrays are typed explicitly and persist via `var` / controlled initialization.

---

## Task 8 — Learning/Preview + Backtest Engine: Determinism & Finalisation
**Why (protocol):** Persistent state must not reset accidentally; intrabar logic must not bleed into historical.  
**Current code anchors:** learning inputs (L35491+), learning data structures (L2480+), backtest/intrabar resolver `f_resolveIntrabar` (near L? shown early), trade types (`BacktestEngine`, `SetupState`, etc.).

**Deliverables**
- State transition audit:
  - “spawn → manage → close → learn” ordering (ensure no double-processing).
- Intrabar resolver integrity:
  - confirm it uses LTF arrays strictly as resolver input and doesn’t introduce repaint into gating.
- Consolidate “execution state” representation:
  - ensure one canonical source of truth for signals used by plots/alerts/execution (no drift).

**Exit criteria**
- Deterministic state machine documented and verified (no hidden side-effects).
- Learning updates occur exactly once per relevant event.

---

## Task 9 — Rendering & Objects: Polyline/Line/Label Budget + Manual GC
**Why (protocol):** Drawing objects can freeze UI or hit hard caps; GC must be explicit for sustained use.  
**Current code anchors:** ZigZag PolylinePlus notes (L843+), `ZigZagVisualBuffer` (L2035), polyline delete usage, debug overlays (L5415+).

**Deliverables**
- Object ledger:
  - count creation sites (`line.new`, `label.new`, `polyline.new`) and their gating.
- GC verification:
  - confirm deletion buffers exist and are enforced before hitting caps.
- Rendering frequency audit:
  - heavy redraw only on meaningful events (`barstate.islast`, confirmed pivots, etc.) where consistent with current behavior.

**Exit criteria**
- Object creation is bounded + gated; manual GC cannot be bypassed in normal flow.

---

## Task 10 — Logging/Docs Consolidation + “Functional Matrix Final Pack”
**Why (protocol):** String churn is a silent killer; docs must support maintainability and AI-assisted refactors.  
**Current code anchors:** UI_TEXT blocks (L11+), debug overlay sections (L5415+), `array.join` usage.

**Deliverables**
- Logging gate audit:
  - ensure debug OFF path does not build strings/tables/labels.
- Replace any loop-time concatenation in debug paths with `array<string>` + `array.join`.
- Finalisation deliverables pack:
  - Functional Matrix vFinal
  - Request Context Ledger vFinal
  - Loop/Hotspot Report vFinal
  - DoD checklist for the entire “finalisation + consolidation” phase

**Exit criteria**
- No doc-block “@field” style remains if semantic naming already covers it.
- Debug/logging is strictly opt-in and cheap when off.
- Final pack is ready to become the next set of specs.

---

## Dependency Order (recommended)
1 → 2 → 4 → 5 → 6 → 7 → 8 → 9 → 10 (with Task 3 parallel to 2/4)

If you want, I can also extract a **one-page “current-state snapshot”** from the script (requests count, loop count, object creation sites, array init sites) to pin down exactly where Tasks 4/6/7/9 should focus first.