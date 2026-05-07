---
name: code-smells-analyzer
description:
  Analyze code for code smells from Martin Fowler / Kent Beck catalog, then refactor them away one by one using Martin
  Fowler's named refactorings. Each smell is fixed in isolation with a test→refactor→test→commit→push cycle. Never
  change tests.
---

# Code Smells Analyzer & Refactorer

Systematically scan code for smells, prioritize them, and remove them one at a time using Martin Fowler's catalog of
refactorings. Every refactoring follows the cycle: **test → refactor → test → fix → test → commit & push → next smell.**

**NEVER change tests.** Tests are the safety net. If a test fails after refactoring, fix the production code, not the
test. If there is a need to change a test, focus only on that first before doing any refactoring, and get confirmation
from the user that it's okay to change tests.

## Phase 1: Scan & Catalog

Before touching any code, perform a quick smell audit, finding the best shortest smells to fix first.

### The Smell Catalog

Scan for of the following. For each smell found, record: file, line range, smell name, severity (high/medium/low), and
which Fowler refactoring to apply.

#### Smells Inside Methods/Functions

| Smell                      | What to look for                                                              | Severity |
| -------------------------- | ----------------------------------------------------------------------------- | -------- |
| **Long Method**            | Method body >10 lines, does more than one thing, hard to name in one sentence | HIGH     |
| **Long Parameter List**    | >4 parameters — signals a missing object                                      | HIGH     |
| **Nested Conditionals**    | Deep if/else/switch nesting (>2 levels) — hidden polymorphism                 | HIGH     |
| **Temporary Field**        | Field/variable set only in some code paths, `undefined`/`null` in others      | MEDIUM   |
| **Dead Code**              | Unreachable branches, unused variables, commented-out code                    | LOW      |
| **Speculative Generality** | Abstractions, interfaces, params "for the future" that nothing uses today     | MEDIUM   |
| **Magic Numbers/Strings**  | Literal values without a named constant explaining intent                     | LOW      |

#### Smells Between Classes/Modules

| Smell                      | What to look for                                                      | Severity |
| -------------------------- | --------------------------------------------------------------------- | -------- |
| **Feature Envy**           | Method reaches into another object's data more than its own           | HIGH     |
| **Data Clumps**            | Same group of 3+ variables passed together across multiple call sites | HIGH     |
| **Primitive Obsession**    | Using `string`/`number` where a dedicated type/class would add safety | MEDIUM   |
| **Inappropriate Intimacy** | Two classes/modules accessing each other's private internals          | HIGH     |
| **Message Chains**         | `a.b().c().d()` — long chains coupling caller to deep structure       | MEDIUM   |
| **Middle Man**             | Class/function that only delegates to another, adding no value        | LOW      |
| **Shotgun Surgery**        | One logical change requires editing 5+ files                          | HIGH     |
| **Divergent Change**       | One file changes for multiple unrelated reasons                       | HIGH     |

#### Smells in Hierarchy/Polymorphism

| Smell                    | What to look for                                                     | Severity |
| ------------------------ | -------------------------------------------------------------------- | -------- |
| **Switch on Type**       | `if/switch` branching on a type string/enum/tag to choose behavior   | HIGH     |
| **Refused Bequest**      | Subclass overrides parent methods to no-op or throws "not supported" | MEDIUM   |
| **Parallel Inheritance** | Adding a class in one hierarchy forces adding one in another         | HIGH     |

#### Design Smells

| Smell                       | What to look for                                                            | Severity |
| --------------------------- | --------------------------------------------------------------------------- | -------- |
| **God Class / Large Class** | Class >200 lines, knows too much, does too much                             | HIGH     |
| **Lazy Class**              | Class does too little to justify its existence (<10 lines of real behavior) | LOW      |
| **Data Class**              | Class with only fields and getters, zero behavior                           | MEDIUM   |
| **Duplicated Code**         | Same logic in 2+ places (even if slightly different shape)                  | HIGH     |
| **Comments as Deodorant**   | Comment explaining what confusing code does, instead of making code clear   | LOW      |

### Scan Procedure

```
1. IDENTIFY target — user specifies file(s), directory, or module to analyze
2. READ all target files
3. For EACH file, check against EVERY smell in the catalog above
4. Record findings in a structured report (see format below)
5. SORT by severity (HIGH first), then by "blast radius" (how many places benefit)
6. Present report to user and ask which smell to fix first (or recommend the highest-priority one)
```

### Report Format

Present findings as:

```markdown
## Code Smell Report

### HIGH severity

1. **[Smell Name]** in `file.ts:25-80`
   - What: [one-sentence description of the smell]
   - Refactoring: [Martin Fowler refactoring name to apply]
   - Risk: [low/medium — how likely this refactoring is to break things]

### MEDIUM severity

...

### LOW severity

...

### Recommended order

1. [smell] in [file] — [why first]
2. [smell] in [file] — [why second] ...
```

## Phase 2: Refactor One Smell at a Time

### The Cycle (RIGID — follow exactly)

```
FOR EACH smell in priority order:
  1. ANNOUNCE — tell the user which smell you're fixing and which refactoring you'll use
  2. TEST    — run the full test suite (or relevant subset). Must be GREEN before starting.
                If tests fail, STOP. Fix existing failures first. Do not refactor on red.
  3. REFACTOR — apply the Fowler refactoring (see recipes below). Make the SMALLEST
                possible change. Do NOT change behavior. Do NOT change tests.
  4. TEST    — run tests again. Must be GREEN.
                If RED: fix the PRODUCTION code (not tests!) to make tests pass again.
                        Then run tests again. Must be GREEN before continuing.
  5. COMMIT  — create a commit with message: "refactor: [Fowler refactoring name] — [smell removed] in [file]"
  6. PUSH    — push to remote
  7. NEXT    — move to the next smell
```

### Critical Rules

- **ONE smell per cycle.** Never batch multiple refactorings into one commit.
- **NEVER change tests.** If tests break, the refactoring was wrong or incomplete — fix the production code.
- **NEVER change behavior.** Refactoring preserves behavior. If you need to change behavior, that's a separate commit.
- **SMALLEST change wins.** If a refactoring can be broken into smaller steps, break it down.
- **Tests must be green between every step within a refactoring.** Not just at the end.

## Phase 3: Martin Fowler's Refactoring Recipes

### Extract Method

**Removes:** Long Method, Comments as Deodorant

```
1. Identify a section that does one thing (comments are hints)
2. Create a new method named after WHAT it does (not HOW)
3. Move the section into the new method
4. Pass needed variables as parameters, return computed results
5. Replace original section with call to new method
6. Run tests
```

### Inline Method

**Removes:** Middle Man, Lazy Class (sometimes)

```
1. Find the method that just delegates
2. Replace all calls with the method's body
3. Remove the method
4. Run tests
```

### Extract Variable / Introduce Explaining Variable

**Removes:** Magic Numbers/Strings, complex expressions

```
1. Identify the complex expression or magic literal
2. Create a well-named const/variable with its value
3. Replace the expression/literal with the variable
4. Run tests
```

### Replace Temp with Query

**Removes:** Temporary Field

```
1. Find the temp variable and its assignment
2. Extract the right-hand side into a method/getter
3. Replace all reads of the temp with calls to the new method
4. Remove the temp declaration
5. Run tests
```

### Introduce Parameter Object

**Removes:** Long Parameter List, Data Clumps

```
1. Create a class/type/interface for the parameter group
2. Add the new object as a parameter alongside old params
3. Move ONE old param's usage to read from new object — test
4. Repeat for each param — test after each
5. Remove old individual params from signature
6. Run tests
```

### Replace Conditional with Polymorphism

**Removes:** Switch on Type, Nested Conditionals

```
1. Identify the conditional and the type value it switches on
2. Create a base class/interface with default behavior
3. For EACH branch, create a subclass/implementation
4. Create a factory: type-value → instance
5. Replace conditional with polymorphic method call
6. Delete the conditional
7. Run tests after EACH step
```

**Functional/module variant:**

```
1. Create a record/map: type-value → handler function
2. Each handler has the same signature
3. Replace conditional with: handlers[type](args)
4. Run tests
```

### Move Method / Move Function

**Removes:** Feature Envy

```
1. Identify which class/module the method "envies"
2. Copy the method to the target class/module
3. Adjust the method to use target's own data (no more reaching across)
4. Update callers to use the new location
5. Remove the old method
6. Run tests
```

### Extract Class

**Removes:** God Class, Data Clumps, Divergent Change

```
1. Identify the cluster of fields + methods that belong together
2. Create a new class/module with ONLY those fields
3. Move ONE method at a time — test after each
4. Update the original to delegate to the new class
5. Run tests
```

### Inline Class

**Removes:** Lazy Class, Middle Man

```
1. Move all features of the lazy class into the class that uses it
2. Update all callers
3. Delete the empty class
4. Run tests
```

### Encapsulate Field / Encapsulate Collection

**Removes:** Inappropriate Intimacy, Data Class

```
1. Make the field private
2. Add getter (and setter if needed)
3. Update all direct accesses to use getter/setter
4. For collections: return a copy or read-only view, never the raw collection
5. Run tests
```

### Hide Delegate

**Removes:** Message Chains

```
1. On the server class (the one being chained through), create a delegating method
2. Replace client's chain call with the new method
3. Run tests
```

### Replace Magic Literal with Named Constant

**Removes:** Magic Numbers/Strings

```
1. Create a named constant with the literal's value
2. Replace all occurrences of the literal
3. Run tests
```

### Remove Dead Code

**Removes:** Dead Code, Speculative Generality

```
1. Verify the code is truly unreachable (grep for callers, check all code paths)
2. Delete it
3. Run tests
```

### Collapse Hierarchy

**Removes:** Refused Bequest, Parallel Inheritance (after other refactorings)

```
1. Identify superclass and subclass that are too similar
2. Move all features into one class
3. Remove the empty class
4. Run tests
```

## Decision Flowchart

When you find a smell, pick the refactoring:

```
Smell found
  ├─ Switch/if on type? → Replace Conditional with Polymorphism
  ├─ Duplicated code? → Extract Method (if same class) or Extract Class (if across classes)
  ├─ Long method? → Extract Method
  ├─ Long param list? → Introduce Parameter Object
  ├─ Data clump? → Introduce Parameter Object or Extract Class
  ├─ Feature envy? → Move Method
  ├─ God class? → Extract Class
  ├─ Data class (no behavior)? → Move behavior IN (from envious callers) via Move Method
  ├─ Middle man? → Inline Class or Inline Method
  ├─ Lazy class? → Inline Class
  ├─ Message chain? → Hide Delegate
  ├─ Inappropriate intimacy? → Encapsulate Field + Move Method
  ├─ Magic literal? → Replace Magic Literal with Named Constant
  ├─ Dead code? → Remove Dead Code
  ├─ Speculative generality? → Remove Dead Code (delete unused abstractions)
  ├─ Temporary field? → Replace Temp with Query or Extract Class
  ├─ Refused bequest? → Collapse Hierarchy or Replace Inheritance with Delegation
  ├─ Parallel inheritance? → Move methods to collapse one hierarchy
  ├─ Shotgun surgery? → Inline scattered pieces → re-Extract around axis of change
  ├─ Divergent change? → Extract Class (split responsibilities)
  └─ Comments as deodorant? → Extract Method (name replaces comment)
```

## Red Flags — STOP immediately if:

| You're tempted to...                 | Instead...                           |
| ------------------------------------ | ------------------------------------ |
| Change a test to make it pass        | Fix the production code              |
| Fix two smells at once               | One smell per cycle                  |
| Rewrite from scratch                 | Refactor in tiny steps               |
| Add new behavior during refactoring  | Separate commit for behavior changes |
| Skip running tests "just this once"  | ALWAYS run tests. No exceptions.     |
| Refactor on a red test suite         | Fix tests first, then refactor       |
| Push without testing                 | Run tests before every push          |
| Combine multiple refactoring commits | One commit per refactoring           |
