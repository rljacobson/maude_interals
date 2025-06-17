# `SortConstraintTable`

The `SortConstraintTable` is an abstract base class that manages sort constraints for symbols in the Maude interpreter.

## Purpose

The `SortConstraintTable` serves as a mechanism to enforce sort constraints on terms during rewriting. Sort constraints are membership axioms that specify when a term belongs to a particular sort, allowing for more precise type checking and sort refinement during computation.

## Key Components

### Data Members
- `sortConstraints`: A vector storing pointers to `SortConstraint` objects
- `tableComplete`: A boolean flag indicating whether the table has been fully populated and ordered

### Core Methods

**Population**: Sort constraints are added via `offerSortConstraint()`, which calls the pure virtual `acceptSortConstraint()` method that derived classes must implement to filter relevant constraints.

**Ordering**: The `orderSortConstraints()` method performs a multi-pass algorithm to select applicable constraints and sorts them by sort index (smallest sort first) using `sortConstraintLt()`.

**Application**: The main functionality is in `constrainToSmallerSort2()`, which attempts to apply sort constraints to refine a term's sort to a more specific one.

## How It Works

The constraint application algorithm works as follows:

1. **Iterates through constraints** in order of increasing sort specificity
2. **Checks applicability** by testing if the constraint's target sort is smaller than the current term's sort
3. **Attempts matching** using the constraint's LHS automaton
4. **Validates conditions** if the constraint has additional conditions
5. **Updates sort** and restarts the process if a constraint successfully applies

## Population Process

Sort constraints are populated during module compilation. In `Module::indexSortConstraints()` , the system:

1. **Preprocesses** each sort constraint
2. **Analyzes the LHS** to determine which symbols should receive the constraint
3. **Offers to specific symbols** if the LHS has no collapse symbols and isn't a variable
4. **Offers to all symbols** otherwise

## Usage Context

The table is used during rewriting when the system needs to determine the most specific sort for a term. The `constrainToSmallerSort()` and `constrainToExactSort()` methods  provide the public interface, checking if constraints exist before delegating to the main algorithm.

**Notes**

The system includes safety checks for limited rewriting contexts , which don't support sort constraint application. The algorithm is designed to handle the partial ordering of sorts and may restart constraint checking when a successful application changes the term's sort.
