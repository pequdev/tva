# Pine Script v6 Coding Requirements & Reference

**Version:** Pine Script v6  
**Last Updated:** December 21, 2025  
**Purpose:** Reference guide for AI assistants and developers working on Pine Script v6 indicators/strategies

---

## 🎯 Critical Rules - READ FIRST

### 1. Version Declaration
```pine
//@version=6
indicator(title = 'My Indicator', shorttitle = 'IND', overlay = true)
```
- **MUST** be the first line
- No version = defaults to v1 (incompatible syntax)

### 2. Indentation
- Use **4 SPACES** (not tabs)
- Consistent indentation is enforced by compiler

### 3. Line Continuation
- NO backslash `\` for line breaks
- Implicit continuation inside `()`, `[]`, or after operators

---

## 📊 Type System

### Qualifiers Hierarchy (Weakest → Strongest)
```
const < input < simple < series
```

#### `const` - Compile-time constants
```pine
const float GOLDEN_RATIO = 0.618
const string PREFIX = "SETUP_"
```
- Available at compile time
- Cannot change during execution
- **Required** for exported library constants

#### `input` - User input values
```pine
int lengthInput = input.int(20, "Length")
```
- Set via Settings/Inputs tab
- Reloads entire script when changed

#### `simple` - Runtime constants
```pine
simple int barCount = 100
```
- Calculated once on first bar
- Does NOT change across bars

#### `series` - Dynamic values
```pine
series float price = close  // Can change every bar
```
- Default for most calculations
- Can vary bar-to-bar

### Type Casting
```pine
// Automatic: int → float
float result = 10 + 1.5  // OK, int 10 becomes 10.0

// Manual: float → int (truncates, doesn't round)
int rounded = int(10.9)  // Returns 10, NOT 11
int rounded = int(math.round(10.9))  // Returns 11

// Explicit na casting
float myVar = na  // Error - ambiguous type
float myVar = na(float)  // OK
```

---

## 🏗️ User-Defined Types (UDTs)

### Declaration Syntax
```pine
// @type MyType - Description
// @field fieldName Description of field
type MyType
    int fieldName = 0
    float price = na
    bool active = false
```

### Creating Instances
```pine
// Constructor (auto-generated)
MyType obj = MyType.new(fieldName = 10, price = 100.0, active = true)

// Copying
MyType copy = obj.copy()  // Shallow copy
```

### Accessing Fields
```pine
// Get field
float p = obj.price

// Set field
obj.price := 101.0

// Compound assignment
obj.fieldName += 1
```

### Methods
```pine
// Method definition (note: 'method' keyword + typed first param)
// @function Description
// @param this (MyType) Instance
// @param value (float) New value
// @returns (float) Updated value
method updatePrice(MyType this, float value) =>
    this.price := value
    this.price

// Method call (dot notation)
float newPrice = obj.updatePrice(200.0)
```

---

## 🔢 Collections

### Arrays
```pine
// Modern syntax (recommended)
array<float> prices = array.new<float>(10, 0.0)

// Legacy syntax (deprecated, avoid)
float[] prices = array.new_float(10, 0.0)

// Common operations
prices.push(close)
float last = prices.pop()
float val = prices.get(5)
prices.set(5, 100.0)
int size = prices.size()
prices.clear()

// Methods chain
float avg = prices.copy().fill(1.0, 0.0, 0, 50).avg()
```

### Matrices
```pine
matrix<float> m = matrix.new<float>(3, 3, 0.0)
m.set(0, 1, 99.0)
float val = m.get(0, 1)
```

### Maps
```pine
// Key type MUST be a value type (int, float, bool, string)
// Value type can be any type
map<string, float> config = map.new<string, float>()

config.put("threshold", 0.618)
float val = config.get("threshold")
bool exists = config.contains("threshold")
config.remove("threshold")

// Iterate
for [key, value] in config
    label.new(bar_index, high, str.format("{0}: {1}", key, value))
```

---

## 🔄 Control Flow

### If-Then-Else
```pine
// Statement form (no return value)
if condition
    plot(close)
else if otherCondition
    plot(open)
else
    plot(hl2)

// Expression form (returns value)
float result = if condition
    close
else
    open

// Ternary operator
float result = condition ? close : open
```

### Switch
```pine
// Expression form
string direction = switch
    close > open => "UP"
    close < open => "DOWN"
    => "FLAT"  // Default case

// Statement form with matching
switch maType
    maChoice.sma =>
        plot(ta.sma(close, 20))
    maChoice.ema =>
        plot(ta.ema(close, 20))
```

### Loops
```pine
// For loop (index-based)
for i = 0 to 10
    // code

// For-in loop (collection iteration)
for element in myArray
    total += element

// For-in with index
for [i, element] in myArray
    label.new(i, element, str.tostring(element))

// While loop
int i = 0
while i < 10
    i += 1
```

---

## 🚫 Common Mistakes & Gotchas

### ❌ WRONG
```pine
// 1. Using series values in simple/const context
simple int len = input.int(20)  // Error if treated as series

// 2. Reassigning const variables
const float VAL = 1.0
VAL := 2.0  // ERROR

// 3. Using na directly in comparisons
if myVar == na  // ERROR - use na() function
    
// 4. Modifying loop variable
for i = 0 to 10
    i := i + 1  // ERROR - loop var is read-only

// 5. Declaring functions inside conditionals
if condition
    myFunc() => close * 2  // ERROR

// 6. Using var without understanding persistence
var float x = 0.0
x := x + 1  // x grows forever (persists across bars)

// 7. History operator on first bar
float prev = close[1]  // Returns na on first bar

// 8. Wrong method parameter type
method myMethod(array<float> this, float val) =>  // Wrong
// Should be:
method myMethod(float this, float val) =>  // Methods extend types
```

### ✅ CORRECT
```pine
// 1. Proper qualifier usage
int len = input.int(20)  // Auto-infers 'input int'

// 2. Check before modifying
const float VAL = 1.0  // Cannot reassign

// 3. Proper na checking
if na(myVar)
    myVar := 0.0

// 4. Use loop variable as read-only
for i = 0 to 10
    int adjusted = i + 1

// 5. Declare functions in global scope
myFunc() => close * 2
if condition
    plot(myFunc())

// 6. Reset var when needed
var float x = 0.0
if condition
    x := 0.0  // Reset explicitly

// 7. Check bar_index or use nz()
float prev = bar_index > 0 ? close[1] : close

// 8. Correct method signature
method myMethod(array<float> this, float val) =>
    this.push(val)
```

---

## 📦 Variable Declaration Modes

### Default (Reinitializes every bar)
```pine
int counter = 0  // Resets to 0 on every bar
counter += 1     // Always 1
```

### `var` (Persists across bars)
```pine
var int counter = 0  // Initialized only on bar_index == 0
counter += 1         // Grows: 1, 2, 3, 4...
```

### `varip` (Persists including intrabar)
```pine
varip int ticks = 0  // Survives real-time ticks
ticks += 1           // Counts every update, not just bar close
```

---

## 🎨 Plotting & Drawing

### Plot Functions
```pine
// Basic plot (returns plot ID)
plot1 = plot(close, "Close", color.blue)

// Fill between plots
plot2 = plot(open, "Open", color.red)
fill(plot1, plot2, color.new(color.purple, 80))

// Conditional plotting
plot(condition ? close : na, "Conditional")

// hline (horizontal line)
hline1 = hline(0, "Zero", color.gray)
```

### Drawing Objects
```pine
// Line
line myLine = line.new(x1 = bar_index[10], y1 = close[10], 
                       x2 = bar_index, y2 = close, 
                       color = color.blue, width = 2)
line.set_xy2(myLine, bar_index, high)
line.delete(myLine)

// Label
label myLabel = label.new(x = bar_index, y = high, 
                          text = "HI", style = label.style_label_down)
label.set_text(myLabel, "UPDATED")

// Box
box myBox = box.new(left = bar_index[5], top = high[5],
                    right = bar_index, bottom = low,
                    border_color = color.blue, bgcolor = color.new(color.blue, 90))

// Table (persistent across bars)
var table t = table.new(position.top_right, 2, 2)
table.cell(t, 0, 0, "Win Rate", text_color = color.white)
table.cell(t, 1, 0, "75%", text_color = color.green)
```

---

## 🔁 Request Functions

### request.security()
```pine
// Get data from another symbol/timeframe
float dailyClose = request.security(syminfo.tickerid, "D", close)

// Request tuple (multiple values)
[o, h, l, c] = request.security(syminfo.tickerid, "D", [open, high, low, close])

// Avoid repainting with lookahead
float dailyClose = request.security(syminfo.tickerid, "D", close, 
                                     lookahead = barmerge.lookahead_off)
```

---

## 🧮 Math & Statistics

### Common Functions
```pine
float absolute = math.abs(-10)      // 10
float maximum = math.max(10, 20)    // 20
float minimum = math.min(10, 20)    // 10
float rounded = math.round(10.6)    // 11
float ceiled = math.ceil(10.1)      // 11
float floored = math.floor(10.9)    // 10
float power = math.pow(2, 3)        // 8
float root = math.sqrt(16)          // 4
float logged = math.log(10)         // ln(10)
float logged10 = math.log10(100)    // 2

// Random
float rand = math.random(0, 100)    // Pseudo-random [0, 100)

// Tick rounding
float tickRounded = math.round_to_mintick(close)
```

### Technical Analysis
```pine
// Moving averages
float sma = ta.sma(close, 20)
float ema = ta.ema(close, 20)
float wma = ta.wma(close, 20)
float vwma = ta.vwma(close, 20)

// Momentum
float rsi = ta.rsi(close, 14)
float roc = ta.roc(close, 10)
float mom = ta.mom(close, 10)

// Volatility
float atr = ta.atr(14)
float stdev = ta.stdev(close, 20)

// Support/Resistance
float highest = ta.highest(high, 20)
float lowest = ta.lowest(low, 20)

// Pivot detection
float pivotHigh = ta.pivothigh(high, 5, 5)  // Returns price or na
float pivotLow = ta.pivotlow(low, 5, 5)

// Crossovers
bool crossUp = ta.crossover(close, sma)
bool crossDown = ta.crossunder(close, sma)
bool cross = ta.cross(close, sma)

// Change detection
bool changed = ta.change(close) != 0
float barsSince = ta.barssince(crossUp)

// Value when condition
float valueWhen = ta.valuewhen(crossUp, close, 0)  // 0 = most recent
```

---

## 🔧 String Formatting

```pine
// Basic concatenation
string msg = "Price: " + str.tostring(close)

// Format with placeholders
string formatted = str.format("O:{0} H:{1} L:{2} C:{3}", open, high, low, close)

// Number formatting
string fixed = str.format("{0,number,#.##}", 3.14159)  // "3.14"

// Multiline (escape sequences)
string multi = "Line 1\nLine 2\nLine 3"
string tabbed = "Col1\tCol2\tCol3"
```

---

## 🔐 Scope & Visibility

### Global Scope
- Variables, UDTs, functions declared outside any block
- Accessible everywhere in script

### Local Scope
- Variables inside functions, methods, if/for/while blocks
- NOT accessible outside their block

### Function/Method Scope Rules
```pine
// ❌ WRONG - Cannot modify global var from function
var int globalCounter = 0

myFunc() =>
    globalCounter += 1  // ERROR

// ✅ CORRECT - Use UDTs for mutable state
type Counter
    int value = 0

method increment(Counter this) =>
    this.value += 1

var Counter counter = Counter.new()
counter.increment()  // OK
```

---

## 📡 Alerts

### Alert Conditions
```pine
// Simple alert
alertcondition(ta.crossover(close, ta.sma(close, 20)), 
               title = "Golden Cross",
               message = "Price crossed above SMA")

// Alert function (more flexible)
if ta.crossover(close, ta.sma(close, 20))
    alert("Golden Cross at " + str.tostring(close), alert.freq_once_per_bar)
```

### Alert Frequencies
- `alert.freq_all` - Every time condition true
- `alert.freq_once_per_bar` - Once per bar (recommended)
- `alert.freq_once_per_bar_close` - Only on bar close

---

## 🎓 Best Practices

### 1. Use Type Annotations
```pine
// ❌ Implicit (hard to debug)
myVar = close

// ✅ Explicit (clear intent)
float myVar = close
```

### 2. Document with Annotations
```pine
// @function Calculate Fibonacci level
// @param start (float) Starting price
// @param end (float) Ending price
// @param ratio (float) Fibonacci ratio (e.g., 0.618)
// @returns (float) Price at Fibonacci level
calcFibLevel(float start, float end, float ratio) =>
    start + (end - start) * ratio
```

### 3. Use Constants for Magic Numbers
```pine
// ❌ Magic number
if rsi > 70
    signal := -1

// ✅ Named constant
const int RSI_OVERBOUGHT = 70
if rsi > RSI_OVERBOUGHT
    signal := -1
```

### 4. Prefer UDTs Over Parallel Arrays
```pine
// ❌ Hard to maintain
var array<int> timestamps = array.new<int>()
var array<float> prices = array.new<float>()

// ✅ Self-documenting
type PricePoint
    int timestamp
    float price

var array<PricePoint> priceHistory = array.new<PricePoint>()
```

### 5. Handle NA Values
```pine
// Always check for na on history references
float prev = na(close[1]) ? close : close[1]

// Or use nz() to replace na with default
float prev = nz(close[1], close)
```

### 6. Limit Drawing Objects
```pine
// Drawings have limits (~500 lines, ~500 labels, ~500 boxes)
// Delete old objects or use var to reuse
var line myLine = na
if not na(myLine)
    line.delete(myLine)
myLine := line.new(...)
```

---

## 📚 Official Documentation

- **Main Docs:** https://www.tradingview.com/pine-script-docs/
- **Reference Manual:** https://www.tradingview.com/pine-script-reference/v6/
- **Type System:** https://www.tradingview.com/pine-script-docs/language/type-system/
- **Methods:** https://www.tradingview.com/pine-script-docs/language/methods/
- **User Manual:** https://www.tradingview.com/pine-script-docs/primer/first-steps/

---

## 🐛 Debugging Tips

### 1. Use Pine Logs
```pine
// Inspect variables
log.info("close: {0}, volume: {1}", close, volume)

// Conditional logging
if condition
    log.warning("Condition triggered at bar {0}", bar_index)
```

### 2. Plot for Visualization
```pine
// Quick debug plot
plot(myVar, "Debug", color.yellow, display = display.data_window)
```

### 3. Labels for Text
```pine
// On-chart debugging
if barstate.islast
    label.new(bar_index, high, 
              str.format("Total setups: {0}\nWin rate: {1}%", 
                         setupCount, winRate * 100))
```

### 4. Check Compilation Panel
- Red squiggle = syntax error
- Yellow = warning (may still run)
- Hover for error message

---

## ⚠️ Performance Considerations

### 1. Minimize History References
```pine
// ❌ Slow - repeated lookback
for i = 0 to 100
    sum += close[i]

// ✅ Faster - use ta functions
sum = ta.sma(close, 100) * 100
```

### 2. Avoid Nested Loops
```pine
// ❌ O(n²) - very slow
for i = 0 to 100
    for j = 0 to 100
        // ...

// ✅ O(n) - use built-in functions or single pass
```

### 3. Use Maps for Lookups
```pine
// ❌ O(n) array search
float val = na
for [k, v] in keyValuePairs
    if k == searchKey
        val := v

// ✅ O(1) map lookup
float val = myMap.get(searchKey)
```

### 4. Limit Request.security() Calls
```pine
// Each call counts toward 40 request limit
// Batch requests using tuples
[d_open, d_high, d_low, d_close] = 
    request.security(syminfo.tickerid, "D", [open, high, low, close])
```

---

## 🎯 Project-Specific Patterns (PQ_FIBS)

### State Flag Pattern (Bitwise Simulation)
```pine
// Since Pine has no bitwise operators, use arithmetic
int FLAG_IS_LONG = 1
int FLAG_ACTIVE = 2
int FLAG_TP_HIT = 4

// Set flag
state := state + FLAG_ACTIVE  // Only if not already set

// Check flag
bool isActive = (state % (FLAG_ACTIVE * 2)) >= FLAG_ACTIVE

// Clear flag
state := state - (state % (FLAG_ACTIVE * 2) >= FLAG_ACTIVE ? FLAG_ACTIVE : 0)
```

### Regime State Pattern
```pine
// Consolidate all regime detection in one UDT
type RegimeState
    float entropy
    float hurst
    bool momentumOk
    bool entropyOk

method checkAll(RegimeState this) =>
    this.momentumOk and this.entropyOk
```

### Learning Metrics Pattern
```pine
// Track learning state and trigger recalculation
type LearningMetrics
    float winRate
    bool needsRecalc
    array<float> returns

method recordTrade(LearningMetrics this, float returnVal) =>
    this.returns.push(returnVal)
    this.needsRecalc := true

method recalculate(LearningMetrics this) =>
    if this.needsRecalc
        this.winRate := this.returns.avg()
        this.needsRecalc := false
```

---

## 🚨 Version Differences (v5 → v6)

### Removed in v6
- `var` in function parameters (use UDTs instead)
- `security()` - use `request.security()`
- `study()` - use `indicator()`
- `color.new()` with positional args - must use named: `color.new(color.blue, 50)`

### New in v6
- Enum types with `enum` keyword
- Enhanced method overloading
- Improved type inference
- `switch` expressions
- Better array/map iteration

---

## ✅ Pre-Flight Checklist

Before committing Pine Script code:

- [ ] Version declaration present (`//@version=6`)
- [ ] All UDTs documented with `@type`, `@field`
- [ ] All functions documented with `@function`, `@param`, `@returns`
- [ ] No magic numbers (use constants)
- [ ] NA values handled (use `na()`, `nz()`, or history checks)
- [ ] Type annotations on ambiguous declarations
- [ ] No series values in simple/const contexts
- [ ] Methods have proper typed first parameter
- [ ] Drawing object limits respected (delete old objects)
- [ ] Alert messages are descriptive
- [ ] Performance: no nested loops, minimal history refs

---

**End of Reference Document**

*For questions on syntax, always consult the official documentation first.*  
*When in doubt, test in Pine Editor - it provides excellent error messages.*
