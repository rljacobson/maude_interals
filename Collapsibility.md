# Term Collapsibility in Maude

## Definition of Collapsibility

In Maude, a term is considered "collapsible" when it can be reduced to a simpler form due to the presence of identity elements in algebraic theories. This is particularly important in theories with associative and/or commutative operators.

## Types of Collapse

There are two main types of collapse:

### 1. Collapse to Identity

This occurs when a term can be reduced to the identity element of an operation. For example:

```
X * e = X
```

Where `e` is the identity element for the operation `*`. The code shows this in `AU_Symbol.cc` and `ACU_Symbol.cc` where identity elements are handled.

### 2. Collapse to Subterm

This happens when a term can be reduced to one of its subterms. For example, in the `AU_Term.cc` implementation:

```cpp
void analyseCollapses2()
```

This method analyzes potential collapses based on identity elements and sets flags like `collapseToOurSymbol` and `matchOurIdentity`.

## Implementation Details

Collapsibility is handled in a few ways:

1. **Tracking Collapse Potential**:
   - In `FreeTerm.hh`, there's a field `uniqueCollapseSubtermIndex` that tracks which subterm can uniquely collapse.
   - In `ACU_Term.hh`, the `Pair` structure contains flags like `collapseToOurSymbol` and `matchOurIdentity`.

2. **Collapse During Normalization**:
   - In `AU_Term.cc` and `ACU_Term.cc`, the `normalize()` method handles collapsing terms during normalization.

3. **Collapse in Word Problems**:
   - In `wordLevel-collapseCase.cc`, there's specific handling for cases where "collapse to empty" is allowed.

## Example from the Code

From `wordLevel-collapseCase.cc`:

```cpp
if (constraintMap[j].canTakeEmpty())
{
  if (makeEmptyAssignment(j))  // may introduce null equations
    changed = true;
}
```

This shows a case where a variable can "collapse" to the empty word, which is a form of identity element in word problems.

## Practical Significance

Collapsibility is used for:

1. **Efficiency**: Identifying collapsible terms allows for more efficient rewriting.
2. **Correctness**: Proper handling of collapse cases ensures that term rewriting follows the algebraic laws correctly.
3. **Termination**: Understanding when terms can collapse helps in determining if rewriting processes will terminate.
4. **Unification and Matching**: Collapse cases must be considered during pattern matching and unification to ensure all possible matches are found.

