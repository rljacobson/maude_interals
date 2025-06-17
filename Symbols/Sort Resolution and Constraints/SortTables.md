# Understanding the SortTable Class in Maude

## Purpose of SortTable

`SortTable` is an abstract class that manages the sort declarations associated with symbols in the Maude system. Its primary responsibilities include:

1. **Managing Operator Sort Declarations**: It maintains a collection of operator declarations that define the domain and range sorts for operators.
2. **Sort Computation**: It provides mechanisms to determine the sort of a term based on the sorts of its subterms.
3. **Sort Checking**: It enables verification that terms are well-formed with respect to the sort structure.

## Key Functionality

From the implementation in `sortTable.cc` and the interface in `sortTable.hh`, we can see that `SortTable`:

### 1. Maintains Sort Information

```cpp
void addOpDeclaration(const Vector<Sort*>& domainAndRange, bool constructorFlag)
```

This method allows adding operation declarations with their domain sorts (arguments) and range sort (result), along with information about whether they are constructors.

### 2. Builds Sort Diagrams

```cpp
void buildSortDiagram()
```

This constructs internal data structures that represent the relationships between sorts, which are used for efficient sort computation during term processing.

### 3. Compute Sort Functions

```cpp
void computeSortFunctionBdds(const SortBdds& sortBdds, Vector<Bdd>& sortFunctionBdds) const
```

This uses Binary Decision Diagrams (BDDs) to efficiently represent and compute sort functions.

### Term Sort Resolution

```c++
virtual void fillInSortInfo(Term* subject) = 0;
```

Defines a mechanism to compute and assign the correct sort to a term based on its structure and the sorts of its subterms.

### 4. Handle Sort Subsumption

```cpp
bool domainSubsumes(int subsumer, int victim) const
```

This determines when one sort declaration is more general than another, which is crucial for overloaded operators.

## Relationship to Symbol Class

As a base class of `Symbol`, `SortTable` provides each symbol with:

1. The ability to determine the sorts of terms constructed with that symbol
2. Information about whether the symbol is a constructor or not
3. Mechanisms to handle overloaded declarations of the same operator with different sorts

## Practical Example

When you define an operator in Maude like:

```maude
op _+_ : Nat Nat -> Nat .
op _+_ : Int Int -> Int .
```

The `SortTable` functionality in the corresponding `Symbol` instance:

- Stores both declarations
- Determines which declaration applies based on argument sorts
- Computes the resulting sort of any term using the `+` operator

Maude's order-sorted type system allows operators to be overloaded and ensures type safety during rewriting.

# Conclusion

`SortTable` provides the machinery for managing sort declarations, computing result sorts, and enforcing sort constraints. As a base class of `Symbol`, it gives symbols the ability to handle the sort-related aspects of term construction and manipulation, which implements Maude's order-sorted equational logic system.
