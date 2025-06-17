# Understanding `SortConstraintTable` in Maude

## Purpose of `SortConstraintTable`

`SortConstraintTable` is an abstract class that manages sort constraints associated with symbols in Maude. Its primary responsibilities include:

1. **Managing Sort Constraints**: It stores and organizes sort constraints that apply to a particular symbol.
2. **Enforcing Sort Constraints**: It applies these constraints to terms during rewriting to ensure they have the correct sort.
3. **Constraining Terms to Smaller Sorts**: It provides mechanisms to force terms to have more specific sorts when applicable.

## Key Functionality

From the implementation in `sortConstraintTable.cc` and the interface in `sortConstraintTable.hh`, we can see that `SortConstraintTable` provides:

### 1. Sort Constraint Management

```cpp
void offerSortConstraint(SortConstraint* sortConstraint);
virtual bool acceptSortConstraint(SortConstraint* sortConstraint) = 0;
```

These methods allow adding sort constraints to the table, with derived classes determining which constraints are applicable.

### 2. Sort Constraint Application

```cpp
void constrainToSmallerSort(DagNode* subject, RewritingContext& context);
void constrainToExactSort(DagNode* subject, RewritingContext& context);
```

These methods apply sort constraints to terms, potentially forcing them to have more specific sorts.

### 3. Sort Constraint Ordering

```cpp
void orderSortConstraints();
```

This method orders sort constraints by sort specificity, with constraints for smaller sorts (larger indices) coming first.

### 4. Sort Constraint Compilation

```cpp
virtual void compileSortConstraints();
```

This method compiles sort constraints for efficient application during rewriting.

## Differences Between SortTable and SortConstraintTable

While both classes deal with sorts, they serve different purposes:

### SortTable
1. **Declaration-focused**: Manages operator declarations with their domain and range sorts.
2. **Static Sort Computation**: Determines what sort a term will have based on its structure and the sorts of its subterms.
3. **Compile-time Analysis**: Builds sort diagrams and performs sort analysis during module compilation.
4. **Type Checking**: Ensures type correctness during term construction.

### SortConstraintTable
1. **Constraint-focused**: Manages sort constraints that can dynamically change a term's sort.
2. **Dynamic Sort Refinement**: Can force terms to have more specific sorts during rewriting.
3. **Runtime Application**: Applies constraints during term rewriting rather than just during construction.
4. **Membership Axioms**: Implements Maude's membership axioms (`mb` and `cmb` statements).

## Example of Usage

When a term like `f(a, b)` is rewritten:

1. `SortTable` initially determines the sort of the term based on the declarations of `f` and the sorts of `a` and `b`.
2. `SortConstraintTable` then checks if any sort constraints apply to this term.
3. If a constraint matches and its conditions are satisfied, the term's sort is refined to a more specific sort.

For example, if we have:

```
mb f(X:Nat, Y:Nat) : NzNat if X =/= 0 .
```

Then `SortConstraintTable` would refine the sort of `f(1, 2)` from `Nat` to `NzNat` during rewriting.

## Implementation Details

The implementation shows how sort constraints are applied:

```cpp
void SortConstraintTable::constrainToSmallerSort2(DagNode* subject, RewritingContext& context)
{
  // Try sort constraints, smallest sort first
  for (int i = 0; i < nrConstraints; i++) {
    SortConstraint* sc = sortConstraints[i];
    Sort* s = sc->getSort();
    if (leq(currentSortIndex, s))
      break;
    if (leq(s, currentSortIndex)) {
      // Try to match and apply the constraint
      if (sc->getLhsAutomaton()->match(subject, context, sp, 0)) {
        // If matched and condition satisfied, change the sort
        currentSortIndex = s->index();
        subject->setSortIndex(currentSortIndex);
        goto retry;
      }
    }
  }
}
```

This shows how `SortConstraintTable` dynamically refines sorts during rewriting, which is fundamentally different from the static sort computation performed by `SortTable`.

## Conclusion

`SortConstraintTable` is a component in Maude's membership equational logic. While `SortTable` handles the basic sort structure and static sort computation, `SortConstraintTable` provides the dynamic sort refinement mechanism needed to implement membership axioms.

# Understanding `SortTable` and `SortConstraintTable` in Maude

Differences between `SortTable` and `SortConstraintTable`.

## SortTable

`SortTable` manages statically determined sort information for symbols. This information is established during compilation and remains fixed during execution:

1. **Purpose**: Determines what sort a term will have based on the sorts of its subterms
2. **When it's used**: During compilation and term normalization
3. **Key functionality**:
   - Stores operator declarations (domain and range sorts)
   - Builds and maintains a "sort diagram" - a state machine that computes resulting sorts
   - Handles sort computation during term normalization

From the code, we can see this static nature:

```cpp
void SortTable::buildSortDiagram() {
  // Builds a static decision structure for determining sorts
  // This happens once during compilation
}
```

## SortConstraintTable

`SortConstraintTable` manages dynamically applied sort constraints that can change a term's sort during program execution:

1. **Purpose**: Applies membership axioms to potentially refine a term's sort at runtime
2. **When it's used**: During term rewriting at runtime
3. **Key functionality**:
   - Stores and manages sort constraints (membership axioms)
   - Applies constraints to potentially refine a term's sort
   - Handles dynamic sort changes during rewriting

The dynamic nature is evident in this method:

```cpp
void SortConstraintTable::constrainToSmallerSort2(DagNode* subject, RewritingContext& context) {
  // This can dynamically change a term's sort during execution
  // ...
  currentSortIndex = s->index();
  subject->setSortIndex(currentSortIndex);
  // ...
}
```

## The Relationship

The relationship between these classes reflects Maude's two-level type system:

1. **Static Level (SortTable)**: Determines the most general sort a term can have based on its structure and operator declarations
2. **Dynamic Level (SortConstraintTable)**: Allows for runtime refinement of sorts through membership axioms

This design enables Maude to have both efficient static type checking and the flexibility of runtime sort refinement through membership axioms.

A good analogy might be:

- `SortTable` is like a compiler's type checker that determines basic types
- `SortConstraintTable` is like a runtime type system that can refine types based on values and conditions

