# `Module::minimumSubstitutionSize`

The `Module::minimumSubstitutionSize` field tracks the minimum number of variable slots that substitutions need to accommodate when working with equations and rules in a Maude module. (`module.hh:119`)

## Purpose and Functionality

The field serves as a coordination mechanism to ensure that variable indexing doesn't conflict between different parts of the system. When equations or rules are compiled, they notify the module of how many variable slots they need, and the module tracks the maximum requirement. (`module.hh:205-214`)

The field is initialized to 1 in the module constructor (`module.cc:57`) and gets updated through the `notifySubstitutionSize()` method when equations are compiled. (`ChangeLog:2028-2039`)

## Key Usage Patterns

**RewritingContext Construction**: When creating rewriting contexts, the minimum substitution size determines the initial size of the substitution vector: (`rewritingContext.hh:220-221`)

**Variable Indexing in Narrowing**: The field provides a starting offset for indexing variables in terms being narrowed, ensuring they don't conflict with variables from equations: `narrowingSearchState.cc:70`, `narrowingSearchState2.cc:103`, `variantNarrowingSearchState.cc:98`.

**Substitution Management**: During narrowing operations, the field helps determine which substitution slots need to be cleared after rule application: (`narrowingSearchState2.cc:355-357`)

**LaTeX Output**: The field is used when generating compound substitutions for LaTeX output, separating equation variables from narrowing variables: (`maudeLatexBuffer.cc:323`)

## Notes

This mechanism was introduced in commit [241f8e6f](https://github.com/rljacobson/Maude/commit/241f8e6f) in 2007 as part of changes to the substitution system. The field ensures proper variable separation between different phases of term rewriting and narrowing operations, preventing variable index conflicts that could lead to incorrect unification results.

## Why It's Needed: Variable Index Conflict Prevention

The fundamental problem this field solves is variable index collision between different phases of term processing. In Maude's unification and narrowing system, variables are assigned integer indices in substitution arrays. Without coordination, variables from equations could use the same indices as variables from terms being narrowed, causing incorrect bindings. ChangeLog:2028-2039

The changelog from commit [241f8e6f](https://github.com/rljacobson/Maude/commit/241f8e6f) shows this was introduced specifically to fix substitution system issues.

## When It's Written To: Equation Compilation

The field gets updated during equation compilation through `notifySubstitutionSize()`: (`module.hh:205-214`)

This method is called from `preEquation.cc (compileMatch)` as noted in the changelog. Each time an equation is compiled, it reports how many variable slots it needs, and the module tracks the maximum across all equations.

The field is initialized to 1 in the module constructor: (`module.cc:57`)

## When It's Read From: Multiple Critical Points

### 1. RewritingContext Construction

Every rewriting context uses this value to size its substitution array: (`rewritingContext.hh:220-221`)

This ensures the substitution has enough slots for all equation variables before any narrowing variables are added.

### 2. Narrowing Variable Indexing

Multiple narrowing classes use this as the starting offset for indexing variables in terms being narrowed: `narrowingSearchState2.cc:103`, `variantNarrowingSearchState.cc:98`, `variantUnificationProblem.cc:123`

### 3. Substitution Cleanup

After rule application, the system clears unused substitution slots to avoid confusing the unification algorithm: `narrowingSearchState.cc:180-186`

The loop clears slots from `r->getNrProtectedVariables()` up to `nrSlots` (which is `minimumSubstitutionSize`).

### 4. LaTeX Output Generation

Even the LaTeX generation system uses this for proper variable separation: `maudeLatexBuffer.cc:323`

This ensures that when generating compound substitutions for documentation, equation variables and narrowing variables are properly separated.

### 5. Meta-level Operations

The meta-programming interface also relies on this value: `metaNarrow.cc:175`

## The Complete Flow

1. **Module Creation**: `minimumSubstitutionSize` initialized to 1
2. **Equation Compilation**: Each equation calls `notifySubstitutionSize()` with its variable count, updating the module's minimum
3. **Runtime Operations**: All substitution-based operations (rewriting contexts, narrowing, unification) use this value to:
    - Size substitution arrays appropriately
    - Offset variable indices to prevent conflicts
    - Clean up unused slots after operations

This creates a clean separation where equation variables occupy slots 0 through `minimumSubstitutionSize-1`, and narrowing/target variables start from `minimumSubstitutionSize` onwards, preventing any index collisions that could corrupt unification results.

## Notes

The system is designed to be conservative - it tracks the maximum requirement across all equations in the module, ensuring sufficient space even when different equations have different variable counts. This prevents the need for dynamic resizing during critical unification operations.

## The Substitution Data Structure

The `Substitution` class uses a vector to store variable bindings and temporary construction values substitution.hh:63 . This vector is accessed through methods like `value(int index)` and `bind(int index, DagNode* value)` substitution.hh:114-128 .

## How minimumSubstitutionSize Functions as an Index

The `minimumSubstitutionSize` serves as a **starting offset** rather than a direct index. It marks the boundary between equation variables (slots 0 to `minimumSubstitutionSize-1`) and narrowing/target variables (starting from `minimumSubstitutionSize`).

### Substitution Construction

When creating substitutions, `minimumSubstitutionSize` determines the initial size . The substitution constructor takes this size parameter (`substitution.hh:67-73`) .

### Variable Indexing Pattern

Multiple narrowing operations use `minimumSubstitutionSize` as the starting slot for indexing variables:

- In `NarrowingSearchState`, variables are indexed starting from this offset (`narrowingSearchState.cc:70-75`)
- In `VariantNarrowingSearchState`, the same pattern is used (`variantNarrowingSearchState.cc:98-99`)
- In `VariantUnificationProblem`, it's explicitly documented as "first slot after variables reserved for preEquation" (`variantUnificationProblem.cc:123`)

### Accessing Values Through the Index

When accessing substitution values, the code uses `minimumSubstitutionSize` as an offset. For example, in LaTeX generation, narrowing variables are accessed at `firstTargetSlot + i` where `firstTargetSlot` is `minimumSubstitutionSize` (`maudeLatexBuffer.cc:323-327`) .

## Notes

The `minimumSubstitutionSize` isn't a single index but rather a partitioning mechanism for the substitution vector. It ensures that equation variables (indices 0 to `minimumSubstitutionSize-1`) don't conflict with variables from terms being narrowed (indices `minimumSubstitutionSize` and above). This prevents variable binding corruption during unification operations.