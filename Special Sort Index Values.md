# Core Sort Special Values

From `src/Core/sort.hh`, the `Sort` class defines these special values:

- `KIND = 0` - represents the kind (top sort in a connected component)
- `ERROR_SORT = 0` - same value as KIND, used for error sorts
- `FIRST_USER_SORT = 1` - the starting index for user-defined sorts
- `SORT_UNKNOWN = -1` - indicates an unknown/uncomputed sort

## Local Special Indices in Sort Table Construction

In `src/Core/sortTable.cc`, during sort diagram construction, there are local special indices :

- `UNINITIALIZED = 0` - indicates an uninitialized state
- `IMPOSSIBLE = -1` - indicates an impossible/invalid state

## Usage Context

These special values are used throughout the sort system:

1. **Runtime sort checking**: `SORT_UNKNOWN` is used to indicate when a `DagNode`'s sort hasn't been computed yet

2. **Sort diagram construction**: The local `UNINITIALIZED` and `IMPOSSIBLE` values are used during the complex state machine construction for determining sort relationships

3. **Symbol sort handling**: The `uniqueSortIndex` field in symbols can be set to `-1` for fast theory-dependent handling

## Memory Allocation Context

There's also usage in memory allocation where `SORT_UNKNOWN` is explicitly set during object creation

### Special Sort Index Values

The main special sort index values are:
- `0`: Used for `KIND`/`ERROR_SORT`, `UNINITIALIZED`, and `FIRST_USER_SORT - 1`
- `-1`: Used for `SORT_UNKNOWN`, `IMPOSSIBLE`, and fast symbol handling
- `1`: Used for `FIRST_USER_SORT`

These values serve as sentinels and special cases throughout Maude's sort system for indicating uncomputed, error, or special states.

**Notes**

The overlap between `KIND` and `ERROR_SORT` both being `0` is intentional - they represent the same concept of the least informative sort in a connected component. The negative values (`-1`) consistently indicate "unknown" or "impossible" states across different subsystems.

# Can special values be used to index into arrays?

Can special sort index values like `Sort::SORT_UNKNOWN` (-1), `Sort::ERROR_SORT` (0), or `SpecialSortIndices::IMPOSSIBLE` (-1) ever be passed as the `sortIndex` parameter to methods like `SortTable::traverse` and `SortTable::ctorTraverse`, which perform array indexing operations like `sortDiagram[position + sortIndex]`?

Passing -1 as `sortIndex` would indeed cause array underflow, accessing `sortDiagram[position - 1]`.

These special negative values should never be used in traverse methods. Here's why:

## How traverse methods are called

The primary usage of these traverse methods is in sort computation and constructor checking. Let me trace through the key call sites:

In , `ctorTraverse` is called with sort indices obtained from `getSortIndex()`:

The code explicitly asserts that the sort index is not `SORT_UNKNOWN` before calling `ctorTraverse`. This is a safety check to prevent exactly the problem you're concerned about.

Similarly, in , there are multiple assertions checking for `Sort::SORT_UNKNOWN` before using sort indices in traverse operations.

## Sort index validation

The system has multiple layers of protection:

1. **Explicit assertions**: As shown above, the code asserts that sort indices are not `SORT_UNKNOWN` before traverse operations
2. **Sort computation**: Before traverse methods are called, the system ensures sorts are properly computed via `computeBaseSort()`
3. **Range validation**: Sort indices used in traverse operations come from valid sorts within connected components, which have indices ≥ 0

## Special values are for different contexts

The special negative values serve different purposes:

- `SORT_UNKNOWN` (-1) is used as a sentinel in  to indicate uncomputed sorts, but these are resolved before traverse operations
- `SpecialSortIndices::IMPOSSIBLE` (-1) is used locally in  during diagram construction, not during traversal

The special negative sort index values should never be passed to traverse or `ctorTraverse` methods. The codebase includes explicit assertions and validation to prevent this, ensuring that only valid non-negative sort indices (representing actual sorts within connected components) are used for array indexing operations.

**Notes**

The assertions serve as both documentation and runtime safety checks, making it clear that passing `SORT_UNKNOWN` or other special values to traverse methods would be a programming error that should be caught during development.
