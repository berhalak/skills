---
name: code-smells-analyzer
description:
  Analyze code for code smells from the Fowler / Beck / Kerievsky catalog (including the Industrial Logic
  Smells-to-Refactorings reference), then refactor them away one by one using named refactorings from Fowler's
  *Refactoring* and Kerievsky's *Refactoring to Patterns*. Each smell is fixed in isolation with a
  test→refactor→test→commit→push cycle. Never change tests.
---

# Code Smells Analyzer & Refactorer

Systematically scan code for smells, prioritize them, and remove them one at a time using named refactorings from the
catalog. Every refactoring follows the cycle: **test → refactor → test → fix → test → commit & push → next smell.**

**NEVER change tests.** Tests are the safety net. If a test fails after refactoring, fix the production code, not the
test. If there is a need to change a test, focus only on that first before doing any refactoring, and get confirmation
from the user that it's okay to change tests.

References used throughout this skill:

- **F** — Fowler, *Refactoring: Improving the Design of Existing Code*
- **K** — Kerievsky, *Refactoring to Patterns*
- **IL** — Industrial Logic, *Smells to Refactorings Quick Reference*

## Phase 1: Scan & Catalog

Before touching any code, perform a quick smell audit, finding the best shortest smells to fix first.

### The Smell Catalog

Scan for each of the following. For each smell found, record: file, line range, smell name, severity (high/medium/low),
and which refactoring(s) to apply.

#### Smells Inside Methods/Functions

| Smell                      | What to look for                                                              | Severity |
| -------------------------- | ----------------------------------------------------------------------------- | -------- |
| **Long Method**            | Method body >10 lines, does more than one thing, hard to name in one sentence | HIGH     |
| **Long Parameter List**    | >4 parameters — signals a missing object                                      | HIGH     |
| **Conditional Complexity** | Deep if/else/switch nesting (>2 levels) or sprawling conditional logic        | HIGH     |
| **Switch Statement**       | The same switch / if-else-if chain duplicated across the system               | HIGH     |
| **Temporary Field**        | Field/variable set only in some code paths, `undefined`/`null` in others      | MEDIUM   |
| **Dead Code**              | Unreachable branches, unused variables, commented-out code                    | LOW      |
| **Speculative Generality** | Abstractions, interfaces, params "for the future" that nothing uses today     | MEDIUM   |
| **Magic Numbers/Strings**  | Literal values without a named constant explaining intent                     | LOW      |

#### Smells Between Classes/Modules

| Smell                          | What to look for                                                                            | Severity |
| ------------------------------ | ------------------------------------------------------------------------------------------- | -------- |
| **Feature Envy**               | Method reaches into another object's data more than its own                                 | HIGH     |
| **Data Clumps**                | Same group of 3+ variables passed together across multiple call sites                       | HIGH     |
| **Primitive Obsession**        | Using `string`/`number`/array where a dedicated type/class would add safety                 | MEDIUM   |
| **Inappropriate Intimacy**     | Two classes/modules accessing each other's private internals                                | HIGH     |
| **Indecent Exposure**          | Methods/classes are public that should be hidden — clients see things they shouldn't        | MEDIUM   |
| **Message Chains**             | `a.b().c().d()` — long chains coupling caller to deep structure                             | MEDIUM   |
| **Middle Man**                 | Class/function that only delegates to another, adding no value                              | LOW      |
| **Shotgun Surgery**            | One logical change requires editing 5+ files                                                | HIGH     |
| **Divergent Change**           | One file changes for multiple unrelated reasons                                             | HIGH     |
| **Solution Sprawl**            | One responsibility is scattered across many classes; rushed feature work left it strewn out | HIGH     |
| **Incomplete Library Class**   | Library class is missing methods you need; you've worked around it instead of extending it  | LOW      |

#### Smells in Hierarchy/Polymorphism

| Smell                                             | What to look for                                                       | Severity |
| ------------------------------------------------- | ---------------------------------------------------------------------- | -------- |
| **Switch on Type**                                | `if/switch` branching on a type string/enum/tag to choose behavior     | HIGH     |
| **Refused Bequest**                               | Subclass overrides parent methods to no-op or throws "not supported"   | MEDIUM   |
| **Parallel Inheritance**                          | Adding a class in one hierarchy forces adding one in another           | HIGH     |
| **Alternative Classes with Different Interfaces** | Two classes do similar things but expose different interfaces          | MEDIUM   |

#### Design / Cross-cutting Smells

| Smell                               | What to look for                                                                            | Severity |
| ----------------------------------- | ------------------------------------------------------------------------------------------- | -------- |
| **God Class / Large Class**         | Class >200 lines, knows too much, does too much                                             | HIGH     |
| **Freeloader / Lazy Class**         | Class does too little to justify its existence (<10 lines of real behavior)                 | LOW      |
| **Data Class**                      | Class with only fields and getters/setters, zero behavior                                   | MEDIUM   |
| **Duplicated Code**                 | Same logic in 2+ places (even if slightly different shape)                                  | HIGH     |
| **Oddball / Inconsistent Solution** | A problem is solved one way in most places and a different way in one or two outliers       | MEDIUM   |
| **Combinatorial Explosion**         | Many code paths do the same thing for different combinations of data/behavior               | HIGH     |
| **Comments as Deodorant**           | Comment explaining what confusing code does, instead of making code clear                   | LOW      |

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
   - Refactoring: [refactoring name(s) to apply, with F/K source citations]
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
  3. REFACTOR — apply the named refactoring (see recipes below). Make the SMALLEST
                possible change. Do NOT change behavior. Do NOT change tests.
  4. TEST    — run tests again. Must be GREEN.
                If RED: fix the PRODUCTION code (not tests!) to make tests pass again.
                        Then run tests again. Must be GREEN before continuing.
  5. COMMIT  — create a commit with message: "refactor: [refactoring name] — [smell removed] in [file]"
  6. PUSH    — push to remote
  7. NEXT    — move to the next smell
```

### Critical Rules

- **ONE smell per cycle.** Never batch multiple refactorings into one commit.
- **NEVER change tests.** If tests break, the refactoring was wrong or incomplete — fix the production code.
- **NEVER change behavior.** Refactoring preserves behavior. If you need to change behavior, that's a separate commit.
- **SMALLEST change wins.** If a refactoring can be broken into smaller steps, break it down.
- **Tests must be green between every step within a refactoring.** Not just at the end.

## Phase 3: Refactoring Recipes

Recipes are grouped by category. Each cites its source. Always run tests after every step — even within a single recipe.

### Method-level refactorings

#### Extract Method [F 110]

**Removes:** Long Method, Comments as Deodorant, Duplicated Code (within a class)

```
1. Identify a section that does one thing (comments are hints)
2. Create a new method named after WHAT it does (not HOW)
3. Move the section into the new method
4. Pass needed variables as parameters, return computed results
5. Replace original section with call to new method
6. Run tests
```

#### Compose Method [K 123]

**Removes:** Long Method, Conditional Complexity (when the method's intent is buried in detail)

```
1. Pick a long method
2. Extract every distinct step into its own well-named method
3. Reduce the original method to a short list of calls — each at the same level of abstraction
4. The body should now read like a paragraph summarizing what the method does
5. Run tests after every extraction
```

#### Inline Method [F 117]

**Removes:** Middle Man, Lazy Class (sometimes)

```
1. Find the method that just delegates
2. Replace all calls with the method's body
3. Remove the method
4. Run tests
```

#### Extract Variable / Introduce Explaining Variable

**Removes:** Magic Numbers/Strings, complex expressions

```
1. Identify the complex expression or magic literal
2. Create a well-named const/variable with its value
3. Replace the expression/literal with the variable
4. Run tests
```

#### Replace Temp with Query [F 120]

**Removes:** Temporary Field, Long Method (preparation step)

```
1. Find the temp variable and its assignment
2. Extract the right-hand side into a method/getter
3. Replace all reads of the temp with calls to the new method
4. Remove the temp declaration
5. Run tests
```

#### Replace Method with Method Object [F 135]

**Removes:** Long Method that resists Extract Method because of tangled local state

```
1. Create a new class named after the method
2. Move the method's local variables into fields of the new class
3. Move the method body into a `compute()` (or similarly named) method
4. Replace the original call site with: new MethodObject(args).compute()
5. Now use Extract Method freely on the new class — no parameter-passing overhead
6. Run tests after each step
```

#### Substitute Algorithm [F 139]

**Removes:** Duplicated Code (subtle), overly complex implementation, Oddball Solution

```
1. Ensure tests cover the existing algorithm's behavior thoroughly
2. Replace the body of the method with a clearer / simpler algorithm
3. Run tests — they enforce behavioral equivalence
```

#### Decompose Conditional [F 238]

**Removes:** Conditional Complexity (a single complex if/else block)

```
1. Extract the condition into a well-named predicate method
2. Extract the then-branch body into its own method
3. Extract the else-branch body into its own method
4. The conditional now reads as: if (isFoo()) doFoo(); else doBar();
5. Run tests after each extraction
```

#### Introduce Assertion [F 267]

**Removes:** Implicit assumptions buried in code (often a precursor to other refactorings)

```
1. Identify a condition the code assumes is always true
2. Add an explicit assert/throw at the start of the method
3. Run tests — assertions must not fire under normal use
```

#### Rename Method [F 273]

**Removes:** Misleading names, Comments as Deodorant, parts of Alternative Classes with Different Interfaces

```
1. Pick a clearer name that reflects what the method actually does
2. Update the declaration
3. Update every call site (use IDE rename for safety)
4. Run tests
```

### Parameter / signature refactorings

#### Introduce Parameter Object [F 295]

**Removes:** Long Parameter List, Data Clumps

```
1. Create a class/type/interface for the parameter group
2. Add the new object as a parameter alongside old params
3. Move ONE old param's usage to read from new object — test
4. Repeat for each param — test after each
5. Remove old individual params from signature
6. Run tests
```

#### Preserve Whole Object [F 288]

**Removes:** Long Parameter List when params are all extracted from one object

```
1. Identify a call site passing several values that all came from one object
2. Change the method signature to accept the whole object instead
3. Inside the method, replace param uses with `obj.field` reads
4. Run tests
```

#### Replace Parameter with Method [F 292]

**Removes:** Long Parameter List when a param can be derived inside the callee

```
1. Verify the callee has access to compute the value itself
2. Have the callee call the relevant method instead of taking the param
3. Remove the param from the signature and from all call sites
4. Run tests
```

#### Replace Parameter with Explicit Methods [F 285]

**Removes:** Switch Statement on a parameter that picks behavior

```
1. For each branch of the switch on the parameter, create a dedicated method
2. Update callers to call the specific method for their case
3. Remove the original parameterized method
4. Run tests
```

#### Remove Parameter [F 277]

**Removes:** Speculative Generality

```
1. Verify the parameter is unused in the body (and across overrides)
2. Remove from the signature
3. Remove from every call site
4. Run tests
```

#### Move Accumulation to Collecting Parameter [K 313]

**Removes:** Long Method that builds up a result by mutating local state across many extracted calls

```
1. Introduce a collecting parameter (e.g., a builder, list, or stream)
2. Have each extracted helper write into it
3. The original method becomes: create accumulator → call helpers → return accumulator
4. Run tests after each helper migrates
```

#### Move Accumulation to Visitor [K 320]

**Removes:** Long Method that walks a structure accumulating results; also Shotgun Surgery across types

```
1. Define a Visitor interface with a method per node type
2. Move accumulation logic into a concrete visitor
3. Add `accept(visitor)` to each node type
4. Replace the original walking method with `structure.accept(new MyVisitor())`
5. Run tests after each node-type migrates
```

### Conditional / type-code refactorings

#### Replace Conditional with Polymorphism [F 255]

**Removes:** Switch on Type, Conditional Complexity

```
1. Identify the conditional and the type value it switches on
2. Create a base class/interface with default behavior
3. For EACH branch, create a subclass/implementation
4. Create a factory: type-value → instance
5. Replace conditional with polymorphic method call
6. Delete the conditional
7. Run tests after EACH step
```

**Functional/module variant (handler map):**

```
1. Create a record/map: type-value → handler function
2. Each handler has the same signature
3. Replace conditional with: handlers[type](args)
4. Run tests
```

#### Replace Conditional Logic with Strategy [K 129]

**Removes:** Conditional Complexity where each branch is a different *algorithm* (not a different type)

```
1. Define a Strategy interface representing the variable algorithm
2. Implement one Strategy class per branch
3. Inject the chosen Strategy into the context object
4. Replace the conditional with `strategy.execute(args)`
5. Run tests after each branch migrates
```

#### Replace State-Altering Conditionals with State [K 166]

**Removes:** Conditional Complexity where conditions check object state and mutate it

```
1. Identify the state values the conditionals key off
2. Define a State interface; create one State subclass per state value
3. Move the state-specific behavior (and transitions) into each subclass
4. The context delegates to its current State and gets a new one back on transitions
5. Run tests after each state migrates
```

#### Replace Conditional Dispatcher with Command [K 191]

**Removes:** Switch Statement that dispatches actions, Conditional Complexity

```
1. Define a Command interface with `execute()`
2. Create one Command class per branch
3. Build a registry: key → Command instance
4. Dispatcher becomes: `commands[key].execute()`
5. Run tests after each branch migrates
```

#### Replace Type Code with Class [F 218, K 286]

**Removes:** Primitive Obsession (a numeric/string code masquerading as a type)

```
1. Create a small class wrapping the type code value
2. Provide named constants/factory methods for the valid values
3. Replace the primitive field with the new class
4. Update users gradually; run tests after each
```

#### Replace Type Code with Subclasses [F 223]

**Removes:** Switch on Type when the code never changes after construction

```
1. Make the class with the type code abstract (or split it)
2. Create one subclass per type-code value
3. Move type-specific behavior into the subclasses
4. Replace `new Foo(TYPE_X)` with `new FooX()`
5. Run tests after each subclass appears
```

#### Replace Type Code with State/Strategy [F 227]

**Removes:** Switch on Type when the type code can change at runtime

```
1. Create a State/Strategy hierarchy for the type code's values
2. Replace the field's primitive value with a reference to the State/Strategy
3. Delegate type-specific behavior to it
4. Run tests
```

#### Replace Implicit Language with Interpreter [K 269]

**Removes:** Combinatorial Explosion when code expresses a mini-language via ad-hoc combinations

```
1. Identify the implicit "grammar" buried in conditionals/strings
2. Define an Expression hierarchy representing the grammar
3. Build interpreters that evaluate Expressions
4. Replace ad-hoc code with `expression.interpret(context)`
5. Run tests after each grammar element migrates
```

#### Replace Implicit Tree with Composite [K 178]

**Removes:** Primitive Obsession (nested arrays/maps used as a tree)

```
1. Define a Component interface with operations needed across the tree
2. Create Leaf and Composite implementations
3. Replace the implicit tree (nested primitive structures) with Composite instances
4. Run tests after each level migrates
```

#### Replace One/Many Distinctions with Composite [K 224]

**Removes:** Duplicated Code where single-item and many-item paths diverge

```
1. Apply the Composite pattern so a "many" object has the same interface as a "one"
2. Delete the duplicated single-item handling — callers use Composite uniformly
3. Run tests
```

#### Form Template Method [F 345, K 205]

**Removes:** Duplicated Code across sibling subclasses with similar overall flow

```
1. Identify the common skeleton across siblings
2. Pull the skeleton up to the parent as a template method
3. Replace varying steps with abstract / overridable hook methods
4. Subclasses override only the hooks
5. Run tests after each sibling migrates
```

#### Move Embellishment to Decorator [K 144]

**Removes:** Conditional Complexity where conditionals add optional behavior to a core flow

```
1. Identify the optional embellishment (logging, caching, validation, etc.)
2. Create a Decorator class wrapping the core component
3. Move embellishment code into the Decorator
4. Compose decorators at the construction site instead of branching at runtime
5. Run tests
```

#### Introduce Null Object [F 260, K 301]

**Removes:** Conditional Complexity caused by null checks; some Switch Statements

```
1. Create a NullXxx class implementing the same interface as Xxx with do-nothing behavior
2. Return NullXxx instead of null from factories/getters
3. Delete null-check conditionals at call sites
4. Run tests
```

### Class / structure refactorings

#### Move Method [F 142]

**Removes:** Feature Envy, Inappropriate Intimacy, Solution Sprawl

```
1. Identify which class/module the method "envies"
2. Copy the method to the target class/module
3. Adjust the method to use target's own data (no more reaching across)
4. Update callers to use the new location
5. Remove the old method
6. Run tests
```

#### Move Field [F 146]

**Removes:** Feature Envy on data, Inappropriate Intimacy, Solution Sprawl

```
1. Identify the field's true home (the class that uses it most)
2. Add the field on the target class with proper accessors
3. Update writes and reads on the source to delegate to target
4. Remove the field from the source
5. Run tests after each call site migrates
```

#### Extract Class [F 149]

**Removes:** God Class, Data Clumps, Divergent Change, Solution Sprawl, Temporary Field

```
1. Identify the cluster of fields + methods that belong together
2. Create a new class/module with ONLY those fields
3. Move ONE method at a time — test after each
4. Update the original to delegate to the new class
5. Run tests
```

#### Extract Subclass [F 330]

**Removes:** Indecent Exposure, Conditional Complexity tied to a subset of instances

```
1. Identify features used only by a subset of instances
2. Create a subclass; move those features into it
3. Update factories/constructors to instantiate the subclass when appropriate
4. Run tests after each feature migrates
```

#### Extract Interface [F 341]

**Removes:** Indecent Exposure, Alternative Classes with Different Interfaces (preparatory step)

```
1. Identify the subset of methods clients actually need
2. Define an interface containing exactly those methods
3. Have the class implement it
4. Change client code to depend on the interface
5. Run tests
```

#### Extract Composite [K 214]

**Removes:** Duplicated Code in sibling classes that all hold a similar collection / sub-tree

```
1. Identify the duplicated composite-handling code in siblings
2. Pull it up into a shared Composite parent class
3. Siblings inherit; duplication disappears
4. Run tests
```

#### Inline Class [F 154]

**Removes:** Lazy Class / Freeloader, Middle Man, Speculative Generality (over-extracted classes)

```
1. Move all features of the lazy class into the class that uses it
2. Update all callers
3. Delete the empty class
4. Run tests
```

#### Inline Singleton [K 114]

**Removes:** Indecent Exposure / unnecessary global state

```
1. Move the Singleton's behavior into the class that actually needs it
2. Replace `Singleton.instance().x()` with direct calls
3. Delete the Singleton class
4. Run tests
```

#### Encapsulate Field [F 206]

**Removes:** Inappropriate Intimacy, Indecent Exposure, Data Class

```
1. Make the field private
2. Add getter (and setter if needed)
3. Update all direct accesses to use getter/setter
4. Run tests
```

#### Encapsulate Collection [F 208]

**Removes:** Inappropriate Intimacy on collections, Data Class

```
1. Make the collection field private
2. Provide add/remove methods on the owner class
3. Return a copy or read-only view from the getter, never the raw collection
4. Update callers to use the new methods
5. Run tests
```

#### Hide Delegate [F 157]

**Removes:** Message Chains

```
1. On the server class (the one being chained through), create a delegating method
2. Replace client's chain call with the new method
3. Run tests
```

#### Remove Middle Man [F 160]

**Removes:** Middle Man (the *opposite* of Hide Delegate when delegation has gone too far)

```
1. Identify a class that mostly just forwards to a delegate
2. Have clients talk to the delegate directly
3. Delete the empty forwarding methods (or the whole middle man if nothing's left)
4. Run tests
```

#### Pull Up Method [F 322]

**Removes:** Duplicated Code across sibling subclasses

```
1. Verify the method has identical (or unifiable) bodies in siblings
2. Move it up to the common parent
3. Delete the sibling copies
4. Run tests
```

#### Pull Up Field [F 320]

**Removes:** Duplicated Code on data across siblings

```
1. Verify the field is used the same way in each sibling
2. Move it up to the common parent
3. Delete the sibling copies
4. Run tests
```

#### Push Down Method [F 322]

**Removes:** Refused Bequest

```
1. Identify a parent method only some subclasses actually use
2. Move it down into just those subclasses
3. Run tests
```

#### Push Down Field [F 329]

**Removes:** Refused Bequest on data

```
1. Identify a parent field only some subclasses actually use
2. Move it down into just those subclasses
3. Run tests
```

#### Collapse Hierarchy [F 344]

**Removes:** Refused Bequest, Parallel Inheritance (after other moves), Speculative Generality

```
1. Identify superclass and subclass that are too similar
2. Move all features into one class
3. Remove the empty class
4. Run tests
```

#### Replace Inheritance with Delegation [F 352]

**Removes:** Refused Bequest, Inappropriate Intimacy via inheritance

```
1. Create a field on the subclass referring to an instance of the (former) superclass
2. Make the subclass no longer extend the superclass
3. Forward (only) the methods you actually need to the delegate
4. Update callers if needed
5. Run tests after each method migrates
```

#### Replace Delegation with Inheritance [F 355]

**Removes:** Middle Man when nearly every method is forwarded

```
1. Verify nearly all of the delegate's interface is being forwarded
2. Make the class extend the delegate's class
3. Delete the forwarding methods
4. Run tests
```

#### Change Bidirectional Association to Unidirectional [F 200]

**Removes:** Inappropriate Intimacy via mutual references

```
1. Identify which side actually needs the reference
2. Remove the back-pointer field
3. Update callers that traversed the back-pointer to use the surviving direction
4. Run tests
```

### Data refactorings

#### Replace Data Value with Object [F 175]

**Removes:** Primitive Obsession, Indecent Exposure on raw data

```
1. Create a small class wrapping the value (e.g., Money around amount+currency)
2. Replace the field's type with the new class
3. Delegate operations to the new class
4. Run tests after each call site migrates
```

#### Replace Array with Object [F 186]

**Removes:** Primitive Obsession (using a positional array as a record)

```
1. Create a class with named fields matching the array positions
2. Replace array reads (`a[0]`, `a[1]`) with named getters (`obj.name`, `obj.age`)
3. Run tests after each access migrates
```

#### Replace Magic Literal with Named Constant

**Removes:** Magic Numbers/Strings

```
1. Create a named constant with the literal's value
2. Replace all occurrences of the literal
3. Run tests
```

### Construction / library refactorings

#### Chain Constructors [K 340]

**Removes:** Duplicated Code across overloaded constructors

```
1. Find the most general constructor
2. Have other constructors delegate to it (e.g., `this(...)`)
3. Remove the duplicated initialization from the delegators
4. Run tests
```

#### Encapsulate Classes with Factory [K 80]

**Removes:** Indecent Exposure of constructors, Solution Sprawl in object creation

```
1. Make the constructors of the family non-public
2. Provide static factory methods (or a Factory class) returning the abstract type
3. Update callers to use the factory
4. Run tests
```

#### Move Creation Knowledge to Factory [K 68]

**Removes:** Solution Sprawl, Shotgun Surgery on construction logic

```
1. Find creation logic scattered across callers (defaults, validation, wiring)
2. Move it into a Factory method/class
3. Callers now call one Factory entry point
4. Run tests after each caller migrates
```

#### Introduce Polymorphic Creation with Factory Method [K 88]

**Removes:** Switch Statement on type during construction, Conditional Complexity in creation

```
1. Define an abstract Factory Method on the parent
2. Override it in each subclass to return the appropriate concrete instance
3. Replace conditional creation with a call to the factory method
4. Run tests
```

#### Encapsulate Composite with Builder [K 96]

**Removes:** Primitive Obsession, Long Parameter List during composite tree construction

```
1. Define a Builder with a fluent / step-based API
2. Builder assembles the composite under the hood
3. Replace verbose construction code with builder usage
4. Run tests
```

### Library / interface gap refactorings

#### Introduce Foreign Method [F 162]

**Removes:** Incomplete Library Class (when you can't change the library, but only need a method or two)

```
1. In your code (not the library), write a function that takes the library instance as its first arg
2. Use this foreign method everywhere you'd have wanted the missing method
3. Run tests
```

#### Introduce Local Extension [F 164]

**Removes:** Incomplete Library Class (when several methods are missing)

```
1. Create a subclass or wrapper around the library class
2. Add the missing methods there
3. Have your code use the extension instead of the raw library type
4. Run tests after each call site migrates
```

#### Unify Interfaces with Adapter [K 247]

**Removes:** Alternative Classes with Different Interfaces, Oddball Solution

```
1. Identify the target interface (the one most callers use)
2. Wrap the odd class in an Adapter that implements the target interface
3. Update callers to depend on the target interface
4. Run tests
```

### Cleanup refactorings

#### Remove Dead Code

**Removes:** Dead Code, Speculative Generality

```
1. Verify the code is truly unreachable (grep for callers, check all code paths, dynamic dispatch, reflection)
2. Delete it
3. Run tests
```

## Decision Flowchart

When you find a smell, pick the refactoring:

```
Smell found
  ├─ Switch/if on type (stable)? → Replace Type Code with Subclasses [F 223]
  ├─ Switch/if on type (changes at runtime)? → Replace Type Code with State/Strategy [F 227]
  ├─ Switch/if dispatching actions? → Replace Conditional Dispatcher with Command [K 191]
  ├─ Switch/if picking algorithms? → Replace Conditional Logic with Strategy [K 129]
  ├─ Switch/if mutating state? → Replace State-Altering Conditionals with State [K 166]
  ├─ Other Switch on Type? → Replace Conditional with Polymorphism [F 255]
  ├─ Single complex if/else block? → Decompose Conditional [F 238]
  ├─ Null-check pollution? → Introduce Null Object [F 260, K 301]
  ├─ Optional behavior added via flags? → Move Embellishment to Decorator [K 144]
  ├─ Combinatorial Explosion? → Replace Implicit Language with Interpreter [K 269]
  ├─ Duplicated code (same class)? → Extract Method [F 110]
  ├─ Duplicated code (sibling methods)? → Pull Up Method [F 322]
  ├─ Duplicated code (sibling fields)? → Pull Up Field [F 320]
  ├─ Duplicated code (sibling skeletons)? → Form Template Method [F 345, K 205]
  ├─ Duplicated code (across classes)? → Extract Class [F 149]
  ├─ Duplicated code (one-vs-many paths)? → Replace One/Many Distinctions with Composite [K 224]
  ├─ Duplicated code (overloaded constructors)? → Chain Constructors [K 340]
  ├─ Duplicated code (subtle, different shape)? → Substitute Algorithm [F 139]
  ├─ Long method? → Extract Method [F 110] then Compose Method [K 123]
  ├─ Long method, tangled local state? → Replace Method with Method Object [F 135]
  ├─ Long method, building a result? → Move Accumulation to Collecting Parameter [K 313]
  ├─ Long method, walking a structure? → Move Accumulation to Visitor [K 320]
  ├─ Long parameter list? → Introduce Parameter Object [F 295] or Preserve Whole Object [F 288]
  ├─ Param derivable inside callee? → Replace Parameter with Method [F 292]
  ├─ Param picks behavior? → Replace Parameter with Explicit Methods [F 285]
  ├─ Unused param? → Remove Parameter [F 277]
  ├─ Data clump? → Introduce Parameter Object [F 295] or Extract Class [F 149]
  ├─ Primitive Obsession (single value)? → Replace Data Value with Object [F 175]
  ├─ Primitive Obsession (array as record)? → Replace Array with Object [F 186]
  ├─ Primitive Obsession (nested structures)? → Replace Implicit Tree with Composite [K 178]
  ├─ Primitive Obsession (type code)? → Replace Type Code with Class [F 218, K 286]
  ├─ Feature envy (method)? → Move Method [F 142]
  ├─ Feature envy (field)? → Move Field [F 146]
  ├─ God class? → Extract Class [F 149], then Extract Subclass [F 330] / Extract Interface [F 341] as needed
  ├─ Data class (no behavior)? → Move behavior IN via Move Method [F 142]
  ├─ Middle man? → Remove Middle Man [F 160] or Inline Class [F 154]
  ├─ Middle man (almost everything forwarded)? → Replace Delegation with Inheritance [F 355]
  ├─ Lazy class / Freeloader? → Inline Class [F 154] (or Inline Singleton [K 114])
  ├─ Message chain? → Hide Delegate [F 157]
  ├─ Inappropriate Intimacy? → Encapsulate Field [F 206] + Move Method [F 142]
  ├─ Inappropriate Intimacy (mutual refs)? → Change Bidirectional Association to Unidirectional [F 200]
  ├─ Inappropriate Intimacy (via inheritance)? → Replace Inheritance with Delegation [F 352]
  ├─ Indecent Exposure? → Encapsulate Field [F 206] / Encapsulate Classes with Factory [K 80] / Extract Interface [F 341]
  ├─ Magic literal? → Replace Magic Literal with Named Constant
  ├─ Dead code? → Remove Dead Code
  ├─ Speculative generality? → Remove Dead Code, Remove Parameter [F 277], Collapse Hierarchy [F 344], Inline Class [F 154]
  ├─ Temporary field? → Replace Temp with Query [F 120] or Extract Class [F 149]
  ├─ Refused bequest (method)? → Push Down Method [F 322]
  ├─ Refused bequest (field)? → Push Down Field [F 329]
  ├─ Refused bequest (severe)? → Replace Inheritance with Delegation [F 352] or Collapse Hierarchy [F 344]
  ├─ Parallel inheritance? → Move methods/fields to collapse one hierarchy
  ├─ Alternative Classes with Different Interfaces? → Rename Method [F 273], Extract Interface [F 341], Unify Interfaces with Adapter [K 247]
  ├─ Oddball / Inconsistent Solution? → Substitute Algorithm [F 139] or Unify Interfaces with Adapter [K 247]
  ├─ Solution Sprawl? → Move Method/Field [F 142, F 146], Move Creation Knowledge to Factory [K 68], Inline Class [F 154]
  ├─ Incomplete Library Class (one method)? → Introduce Foreign Method [F 162]
  ├─ Incomplete Library Class (many methods)? → Introduce Local Extension [F 164]
  ├─ Shotgun surgery? → Inline scattered pieces → re-Extract around axis of change; consider Move Accumulation to Visitor [K 320]
  ├─ Divergent change? → Extract Class [F 149] (split responsibilities)
  ├─ Misleading name? → Rename Method [F 273]
  ├─ Hidden assumption? → Introduce Assertion [F 267]
  └─ Comments as deodorant? → Extract Method [F 110] (name replaces comment), or Rename Method [F 273]
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
