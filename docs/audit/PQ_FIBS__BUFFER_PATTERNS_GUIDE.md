# PQ_FIBS.pine — Buffer Patterns Guide

> **Task 7 Deliverable** | Generated: 2025-12-23  
> **Purpose**: Document canonical buffer patterns used in this repository for consistent maintenance

---

## Overview

This guide documents the three canonical buffer patterns used in PQ_FIBS.pine:

1. **Push-Bounded Pattern** — Rolling statistical windows
2. **Circular Buffer Pattern** — Fixed-capacity history with O(1) access
3. **GC Object Buffer Pattern** — Drawing object lifecycle management

All patterns share these properties:
- `var` persistence (allocated once)
- Explicit typing (`array<T>`)
- Hard capacity enforcement
- No per-bar allocation in hot paths

---

## Pattern 1: Push-Bounded (Rolling Window)

### Purpose
Maintain a sliding window of the most recent N values for statistical calculations.

### Implementation (L1000-1010)

```pine
// Float variant
method pushBounded(array<float> buf, float v, int window) =>
    if not na(v)
        buf.push(v)
        while buf.size() > window
            buf.shift()

// Int variant
method pushBounded(array<int> buf, int v, int window) =>
    if not na(v)
        buf.push(v)
        while buf.size() > window
            buf.shift()
```

### Usage Pattern

```pine
// Declaration: var for persistence, empty init
var array<float> return_buffer = array.new<float>(0)

// Update: pushBounded enforces cap
return_buffer.pushBounded(log_return, window)

// Access: size check or use helper methods
if return_buffer.isFull(window)
    float latest = return_buffer.lastVal()  // Negative indexing internally
    float prev = return_buffer.prev()
```

### Helper Methods (L1012-1040)

```pine
// Check if buffer has reached target size
method isFull(array<float> buf, int window) =>
    buf.size() >= window

// O(1) last element access via negative indexing
method lastVal(array<float> buf) =>
    buf.size() > 0 ? buf.get(-1) : na

// O(1) second-to-last element
method prev(array<float> buf) =>
    buf.size() > 1 ? buf.get(-2) : na

// Single-pass mean + stdev computation
method meanStdev(array<float> buf) =>
    int n = buf.size()
    // ... O(n) computation
    [mean_val, stdev_val]
```

### Characteristics

| Property | Value |
|----------|-------|
| Allocation | Once (`var`) |
| Update Cost | O(1) amortized |
| Access Cost | O(1) |
| Memory | Fixed after warm-up |
| Warm-up | `window` bars to fill |

### Where Used
- `f_rollingEntropy()` L1112
- `f_rollingHurst()` L1376
- `f_rollingZScore()` L1641
- `SymbolicEntropyState.update()` L1273
- `DyadicHurstState.update()` L1541

---

## Pattern 2: Circular Buffer (Fixed History)

### Purpose
Store fixed-capacity history with O(1) insert and O(1) lookback access. Oldest entries automatically overwritten when capacity reached.

### Implementation (L2555-2590)

```pine
type CircularBufferSetupRecord
    array<SetupRecord> data
    int head = 0
    int count = 0
    int capacity = 50

method init(CircularBufferSetupRecord this, int capacity) =>
    this.capacity := capacity
    this.head := 0
    this.count := 0
    this.data := array.new<SetupRecord>(capacity)  // Pre-allocate full capacity
    this

method push(CircularBufferSetupRecord this, SetupRecord value) =>
    array.set(this.data, this.head, value)  // Overwrite at head position
    this.head := (this.head + 1) % this.capacity  // Wrap around
    this.count := math.min(this.count + 1, this.capacity)
    this

method get(CircularBufferSetupRecord this, int lookback) =>
    if lookback < 0 or lookback >= this.count
        na
    else
        // Convert logical lookback to physical index
        int physicalIndex = (this.head - 1 - lookback + this.capacity) % this.capacity
        array.get(this.data, physicalIndex)
```

### Usage Pattern

```pine
// Declaration
var CircularBufferSetupRecord GLOBAL_setupHistory = CircularBufferSetupRecord.new()

// Initialization
if barstate.isfirst
    GLOBAL_setupHistory.init(INPUT_LEARNING_SAMPLES)

// Push new record (oldest automatically evicted if at capacity)
GLOBAL_setupHistory.push(newRecord)

// Access by lookback (0 = most recent)
SetupRecord latest = GLOBAL_setupHistory.get(0)
SetupRecord older = GLOBAL_setupHistory.get(5)

// Iterate through history
for i = 0 to GLOBAL_setupHistory.size() - 1
    SetupRecord rec = GLOBAL_setupHistory.get(i)
```

### Characteristics

| Property | Value |
|----------|-------|
| Allocation | Once (pre-allocated full capacity) |
| Push Cost | O(1) always |
| Access Cost | O(1) |
| Memory | Constant (capacity × element size) |
| Overwrites | Oldest entry when full |

### Key Difference from Push-Bounded
- **Push-Bounded**: Array grows to window size, then shifts (memory varies during warm-up)
- **Circular Buffer**: Array pre-allocated at full capacity, index wraps (constant memory)

### Where Used
- `GLOBAL_setupHistory` (SetupRecord learning buffer)

---

## Pattern 3: GC Object Buffer (Drawing Objects)

### Purpose
Manage TradingView drawing objects (lines, labels, polylines) with explicit garbage collection to prevent object limit violations.

### Implementation Variants

#### 3.1 Per-Bar Clear Pattern (L3429-3435)

```pine
// Declaration
var array<line> UI_linesBuffer = array.new<line>()
var array<label> UI_labelsBuffer = array.new<label>()

// GC: Delete all objects on new bar
if ta.change(time) != 0 and array.size(UI_linesBuffer) > 0
    for i = 1 to array.size(UI_linesBuffer) by 1
        line.delete(array.shift(UI_linesBuffer))

if ta.change(time) != 0 and array.size(UI_labelsBuffer) > 0
    for i = 1 to array.size(UI_labelsBuffer) by 1
        label.delete(array.shift(UI_labelsBuffer))

// Create: Add objects during bar
f_drawLineTZ(...) =>
    if _x1 - bar_index < 500
        array.push(UI_linesBuffer, line.new(...))
```

#### 3.2 Capped Rolling Pattern (L2068-2093)

```pine
// ZigZag polyline buffer with hard cap
method gcPolys(ZigZagVisualBuffer this) =>
    if not na(this.sealed)
        while array.size(this.sealed) > this.maxPolys
            polyline doomed = array.shift(this.sealed)
            polyline.delete(doomed)  // Delete oldest
    this

method flushIfFull(ZigZagVisualBuffer this) =>
    if array.size(this.points) >= this.maxPoints and not na(this.active)
        array.push(this.sealed, this.active)
        this.active := na
        this.gcPolys()
        chart.point lastPoint = array.get(this.points, -1)  // Use negative indexing
        array.clear(this.points)
        array.push(this.points, lastPoint)  // Preserve continuity
    this
```

### Characteristics

| Property | Per-Bar Clear | Capped Rolling |
|----------|--------------|----------------|
| Object Lifetime | 1 bar | Until evicted |
| GC Trigger | `ta.change(time)` | Capacity exceeded |
| Memory | Per-bar bounded | Fixed cap |
| Use Case | Ephemeral visuals | Persistent visuals |

### Critical Rules for Object Buffers

1. **Always delete before shift/clear**
   ```pine
   // CORRECT
   line.delete(array.shift(buffer))
   
   // WRONG - leaks objects
   array.shift(buffer)  // Object still exists, just orphaned
   ```

2. **Check for na before delete**
   ```pine
   if not na(this.active)
       polyline.delete(this.active)
   ```

3. **Guard creation with visibility checks**
   ```pine
   if _x1 - bar_index < 500  // Only create if visible
       array.push(buffer, line.new(...))
   ```

### Where Used
- `UI_linesBuffer` / `UI_labelsBuffer` — Fib level lines and labels
- `ZigZagVisualBuffer` — ZigZag polyline rendering

---

## Pattern Comparison Matrix

| Aspect | Push-Bounded | Circular Buffer | GC Object Buffer |
|--------|--------------|-----------------|------------------|
| **Purpose** | Rolling stats | Fixed history | Drawing objects |
| **Allocation** | `var array<T>()` | `var UDT` + pre-alloc | `var array<T>()` |
| **Push Method** | `pushBounded()` | `set()` at head | `push()` |
| **Cap Enforcement** | `while > window` | Modular index | `while > max` + delete |
| **Access Pattern** | `get(-1)`, `get(i)` | `get(lookback)` | Iteration for GC |
| **Memory Profile** | Grows to window | Constant | Bounded by cap |

---

## Negative Indexing Best Practices

Pine v6 supports negative indexing for O(1) last-element access:

```pine
// Preferred: Negative indexing
float last = array.get(buf, -1)
float secondLast = array.get(buf, -2)

// Avoid: Manual size calculation
float last = array.get(buf, array.size(buf) - 1)  // Verbose, error-prone
```

### Safety Requirements

```pine
// ALWAYS guard against empty arrays
if array.size(buf) > 0
    float last = array.get(buf, -1)  // Safe

// Or use helper methods that include guards
method lastVal(array<float> buf) =>
    buf.size() > 0 ? buf.get(-1) : na  // Returns na if empty
```

### Migration Candidates in Codebase

| Location | Current | Recommended |
|----------|---------|-------------|
| L2090 | `array.get(this.points, array.size(this.points) - 1)` | `array.get(this.points, -1)` |
| L2105 | `array.get(this.points, array.size(this.points) - 1)` | `array.get(this.points, -1)` |

---

## Anti-Patterns to Avoid

### ❌ Per-Bar Allocation in Hot Paths

```pine
// BAD: Creates new array every bar
for i = 0 to window - 1
    array<float> temp = array.new<float>(0)  // GC churn
```

```pine
// GOOD: Reuse persistent array
var array<float> temp = array.new<float>(0)
temp.clear()  // Reset, don't reallocate
```

### ❌ Unbounded Push

```pine
// BAD: No cap enforcement
array.push(history, newItem)  // Grows forever
```

```pine
// GOOD: Cap with eviction
if history.size() >= maxHistory
    history.shift()
history.push(newItem)
```

### ❌ Array.copy in Loops

```pine
// BAD: N copies for N simulations
for sim = 0 to num_sims - 1
    array<float> shuffled = array.copy(original)  // Memory churn
```

```pine
// BETTER: Shuffle in-place with reset
for sim = 0 to num_sims - 1
    // Reset to original order, then shuffle
    // (implementation depends on use case)
```

### ❌ Orphaned Drawing Objects

```pine
// BAD: Object leaked
array.clear(UI_linesBuffer)  // Objects still exist on chart
```

```pine
// GOOD: Delete then clear
while UI_linesBuffer.size() > 0
    line.delete(array.shift(UI_linesBuffer))
```

---

## Summary

| Pattern | Key Principle | When to Use |
|---------|---------------|-------------|
| **Push-Bounded** | `push()` + `while > cap` + `shift()` | Rolling statistical windows |
| **Circular Buffer** | Pre-allocate + modular index + `set()` | Fixed-size history buffers |
| **GC Object Buffer** | `delete()` before `shift()/clear()` | Drawing object management |

All patterns share:
- ✅ `var` persistence
- ✅ Explicit `array<T>` typing
- ✅ Hard capacity enforcement
- ✅ No per-bar allocation in hot paths
