# Meta-Analysis: Pine Script v6 Implementation Errors & Lessons Learned

## Summary
During the logarithmic returns feature implementation in `PQ_FIBS.pine`, multiple critical errors were made that violated Pine Script v6 standards, proper software engineering practices, and the project's coding patterns. This document catalogs these failures for reference in future work.

---

## 1. Forward Reference Violations (CRITICAL)

### Error
Functions `f_logReturn()`, `f_getPeriodsPerYear()`, and `f_annualizedVol()` were placed at lines 159-207, but they referenced constants `LOG_PERIODS_DAILY`, `LOG_PERIODS_1M`, etc. that were declared 80+ lines LATER at lines 288-303.

### Pine Script v6 Rule Violated
**All identifiers must be declared BEFORE use.** Pine Script v6 has no hoisting. The parser reads top-to-bottom.

### Root Cause
- Failed to verify constant locations before writing functions
- Did not follow the existing script pattern where constants precede functions
- Rushed implementation without checking declaration order

### Correct Pattern
```pine
// 1. CONSTANTS (lines 40-65)
int LOG_PERIODS_DAILY = 252
float LOG_PERIODS_1M = 252.0 * 6.5 * 60

// 2. FUNCTIONS that use those constants (lines 170+)
f_getPeriodsPerYear() =>
    float result = float(LOG_PERIODS_DAILY)  // ✅ Valid: constant declared above
```

---

## 2. Variable Scope Errors

### Error
Used `sample_count` at line 3232 within a loop, but the variable was declared later at line 3368 in a completely different scope block. Later used `sample_count` at line 3458 in an `else` block where it wasn't available.

### Pine Script v6 Rule Violated
**Variables are block-scoped.** A variable declared inside an `if` block at level N is only visible within that block and its children, not in sibling `else` blocks or parent scopes.

### Root Cause
- Did not trace variable declarations before referencing them
- Assumed variable hoisting (JavaScript/C behavior) instead of checking Pine Script v6 scoping rules
- Failed to verify the actual variable `samples_to_check` was already defined for the loop

### Correct Pattern
```pine
if INPUT_LEARNING_ENABLED
    int sample_count = GLOBAL_setupHistory.size()  // Declared at level 1
    if sample_count >= 10                          // ✅ Visible here (level 2)
        learn_text := "..."
    else                                           // ✅ Visible here (sibling level 2)
        learn_text := str.tostring(sample_count)
```

---

## 3. Failure to Follow Existing Code Patterns

### Error
Added new global variables and functions without:
- Checking existing naming conventions (`GLOBAL_*` prefix pattern)
- Verifying where similar globals were declared (lines 1700-1750)
- Following the established constant grouping structure
- Matching documentation style used throughout the script

### Pine Script v6 Standards Violated
The script follows strict patterns:
- **Constants**: `UPPERCASE_WITH_UNDERSCORES`
- **Globals**: `GLOBAL_camelCase`
- **Functions**: `f_functionName()`
- **UDT Fields**: `camelCase`
- **Type annotations**: Always explicit (`float`, `int`, `bool`, `string`)

### Examples of Pattern Violations
❌ **Wrong**: Placing log constants randomly at line 288 (middle of unrelated section)  
✅ **Right**: Grouped with related constants at lines 47-62 (LOGARITHMIC RETURNS CONSTANTS section)

❌ **Wrong**: Incomplete documentation tags  
✅ **Right**: Every function has `@function`, `@param`, `@returns`; every UDT field has `@field`

---

## 4. Insufficient Pre-Implementation Analysis

### Error
Began writing code without:
1. Reading the entire scope where edits would occur
2. Checking what variables were already available (`samples_to_check` vs `sample_count`)
3. Verifying the declaration order of dependencies
4. Understanding the control flow structure (if/else nesting)

### Correct Workflow (MUST FOLLOW)
```
1. READ entire function/block where changes will occur (read_file with ±30 lines context)
2. IDENTIFY all variables in scope (grep_search for variable names)
3. VERIFY declaration order (grep_search with line numbers)
4. PLAN the edit with exact line ranges
5. EXECUTE the edit with 5+ lines of context
6. VALIDATE with get_errors
```

### What I Did (WRONG)
```
1. Assumed variables existed
2. Wrote code without checking scope
3. Made edits without sufficient context
4. Only checked errors AFTER breaking the code
```

---

## 5. Type System Negligence

### Error
Did not verify or document:
- Return types for new functions (Pine Script v6 is strongly typed)
- Parameter types in function signatures
- Array element types (`array<float>` for `GLOBAL_ltfHighs`)
- UDT field types in `LearningMetrics`

### Pine Script v6 Type System Rules
- **Every variable must have explicit or inferable type**
- **Function parameters require type annotations**: `f_func(float x, int y)`
- **Return types inferred from last expression but should be documented**
- **Arrays are typed**: `array<float>`, `array<int>`, etc.
- **UDT fields require explicit types**

### Correct Example
```pine
// @function Compute log return
// @param price_now (float) Current price
// @param price_prev (float) Previous price  
// @returns (float) Log return or 0.0 if invalid
f_logReturn(float price_now, float price_prev) =>
    float result = 0.0  // ✅ Explicit type
    if not na(price_now) and not na(price_prev) and price_prev > 0 and price_now > 0
        result := math.log(price_now / price_prev)
    result  // ✅ Returns float
```

---

## 6. Mixing Language Paradigms

### Error
Applied mental models from JavaScript/C/Python to Pine Script v6:
- Expected variable hoisting (JavaScript)
- Expected block scope to leak to parent (older C behavior)
- Expected implicit type coercion (JavaScript)

### Pine Script v6 Is NOT JavaScript
| Feature | JavaScript | Pine Script v6 |
|---------|-----------|----------------|
| Hoisting | ✅ `var` hoisted | ❌ No hoisting |
| Block scope | ✅ `let`/`const` | ✅ All variables |
| Type coercion | ✅ Implicit | ❌ Explicit only |
| Semicolons | Optional | Not used |
| Return keyword | Required | Implicit (last expr) |

### Pine Script v6 Is NOT C
- No pointers, no manual memory management
- No preprocessor directives
- No bitwise operators (must simulate with arithmetic)
- No `break`/`continue` in loops
- No `switch` statements

---

## 7. Insufficient Validation

### Error
- Did not run `get_errors` immediately after each edit
- Did not verify variable scope before using variables
- Did not check if constants existed before referencing them
- Did not validate against Pine Script v6 syntax before committing code

### Correct Validation Workflow
```
After EVERY edit:
1. get_errors → Check syntax
2. grep_search → Verify all references exist
3. read_file → Confirm context is correct
4. get_errors again → Final validation
```

---

## 8. Spec Compliance Failures

### Error
- Did not verify each FR (Functional Requirement) was correctly implemented
- Did not check that new code integrated with existing Learning Engine logic
- Did not validate that log-return calculations matched the mathematical spec
- Did not confirm Sharpe ratio formula: `avgLogReturn / logReturnStdev`

### Spec-Driven Development (MUST FOLLOW)
```
For each FR:
1. READ the spec requirement
2. FIND the relevant code section (semantic_search)
3. VERIFY existing patterns (grep_search, read_file)
4. IMPLEMENT following existing patterns
5. VALIDATE against spec (check formula, check integration)
6. PROVE completion (grep_search for all components)
```

---

## 9. Documentation Anti-Patterns

### Error
- Added code without proper Pine Script v6 doc comments
- Mixed documentation styles (some functions with `@param`, others without)
- Verbose inline comments instead of self-documenting code
- Did not maintain the existing documentation standard

### Pine Script v6 Documentation Standard
```pine
// @function Brief one-line description
// @param param_name Description of parameter
// @param another_param Description of another parameter
// @returns Description of return value
f_functionName(type param_name, type another_param) =>
    // Implementation
    result

// @type Brief description of UDT purpose
// @field field_name Description of this field
// @field another_field Description of another field
type MyUDT
    float field_name = 0.0
    int another_field = 0
```

---

## 10. Accountability & Process Failures

### Error
- Did not maintain a mental model of the entire codebase structure
- Made changes without understanding surrounding context
- Did not verify assumptions before coding
- Responded to errors reactively instead of preventing them

### Mandatory Pre-Coding Checklist
```
Before writing ANY code:
☐ Have I read the relevant section with ±50 lines context?
☐ Have I verified all variable declarations are in scope?
☐ Have I checked that all constants/functions I reference exist ABOVE my code?
☐ Have I confirmed the types of all parameters and return values?
☐ Have I checked existing patterns in the codebase?
☐ Have I planned the exact line ranges for the edit?
☐ Do I have 5+ lines of context for replace_string_in_file?
☐ Have I verified this follows Pine Script v6 syntax (not JS/C/Python)?
```

---

## Recovery Actions Taken

1. ✅ Moved `LOG_PERIODS_*` constants from lines 288-303 to lines 51-60 (before functions)
2. ✅ Fixed `sample_count` variable scope by using `samples_to_check` at line 3232
3. ✅ Verified all identifiers are declared before use with `grep_search`
4. ✅ Confirmed syntax with `get_errors` showing no errors
5. ✅ Validated declaration order: constants (51-60) → functions (175-217) → usage (1729)

---

## Commitment to Future Work

### I WILL
- ✅ Read entire scope before editing (minimum ±30 lines context)
- ✅ Verify all variable declarations with `grep_search` before using them
- ✅ Check constant/function declaration order before writing new code
- ✅ Run `get_errors` after EVERY edit
- ✅ Follow Pine Script v6 standards (not JavaScript/C/Python mental models)
- ✅ Validate against spec requirements before claiming completion
- ✅ Maintain existing code patterns (naming, structure, documentation)

### I WILL NOT
- ❌ Assume variables exist without verifying with `grep_search`
- ❌ Place code without checking where similar code is located
- ❌ Reference identifiers without confirming they're declared above
- ❌ Mix language paradigms (JavaScript hoisting, C pointers, Python duck typing)
- ❌ Write code without reading surrounding context first
- ❌ Skip validation steps to "save time"
- ❌ Make assumptions about scope, types, or declaration order

---

## Reference: Pine Script v6 Core Principles

1. **Top-to-bottom execution**: Script executes once per bar, top to bottom
2. **No hoisting**: All identifiers must be declared before use
3. **Block scope**: Variables declared in `if`/`for` blocks are local to that block
4. **Strong typing**: Every variable has a type (float, int, bool, string, arrays, UDTs)
5. **Immutability by default**: Use `:=` for reassignment of `var` variables
6. **Implicit return**: Last expression in function is the return value
7. **No semicolons**: Line breaks delimit statements
8. **Arithmetic bit operations**: No native bitwise operators, simulate with math

---

## Usage Instructions

**Copy this document and paste it at the start of EVERY Pine Script v6 task:**

```
REMINDER: Before ANY edits, review "Meta-Analysis: Pine Script v6 Implementation Errors"
- Check declaration order (constants → functions → usage)
- Verify variable scope with grep_search
- Read ±50 lines context before editing
- Run get_errors after EVERY change
- Follow Pine Script v6 rules, NOT JavaScript/C/Python
```

---

*This meta-analysis documents failures during the logarithmic returns feature implementation (21 Dec 2025) for `PQ_FIBS.pine` to prevent recurrence in future Pine Script v6 development.*
