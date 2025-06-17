# `DagNodeSet` Vs `HashConsSet`: Understanding the Differences

While both `DagNodeSet` and `HashConsSet` appear similar at first glance, they serve distinct purposes in Maude's architecture. Let me explain their key differences.

## Core Purpose Differences

1. **DagNodeSet**: A simple set data structure for storing unique `DagNode` pointers
2. **HashConsSet**: A specialized structure implementing hash consing with canonical representation

## Functional Differences

### DagNodeSet
- **Primary Purpose**: Acts as a standard set collection for `DagNode` objects
- **Uniqueness**: Ensures uniqueness based on structural equality via `equal()`
- **Behavior**: Simply stores and retrieves nodes without modifying them
- **Usage**: Used when you need a collection of unique terms without canonicalization
- **Garbage Collection:** Does not implement `SimpleRootContainer`. (The `ProtectedDagNodeSet` does, however.)

### HashConsSet
- **Primary Purpose**: Implements hash consing to ensure shared representation of identical terms
- **Canonicalization**: Creates and maintains canonical representations of terms
- **Behavior**: Actively transforms nodes through `makeCanonical()` or `makeCanonicalCopy()`
- **Garbage Collection**: Participates in garbage collection via `SimpleRootContainer`
- **Sort Handling**: Maintains sort information via `upgradeSortIndex()`

## Implementation Differences

The key implementation differences are evident in their `insert()` methods:

### DagNodeSet::insert

```cpp
int DagNodeSet::insert(DagNode* d)
{
  return PointerSet::insert(d, d->getHashValue());
}
```

Simply inserts the node as-is, using its hash value.

### HashConsSet::insert

```cpp
int HashConsSet::insert(DagNode* d)
{
  unsigned int hashValue = d->getHashValue();
  int index = pointer2Index(d, hashValue);
  if (index != NONE)
  {
    getCanonical(index)->upgradeSortIndex(d);
    return index;
  }
  return PointerSet::insert(d->symbol()->makeCanonical(d, this), hashValue);
}
```

This method:

1. Checks if a canonical version already exists
2. If found, updates sort information and returns the existing index
3. If not found, creates a canonical version via `makeCanonical()` before insertion

## Additional HashConsSet Features

`HashConsSet` has several features not present in `DagNodeSet`:

1. **Canonical Copy Creation**: The `insertCopy()` method ensures unreduced eager nodes are always copied
2. **Garbage Collection Support**: Implements `markReachableNodes()` to protect hash-consed nodes. (A `ProtectedDagNodeSet` is a `DagNodeSet` that implements `RootContainer`.)
3. **Collision Analysis**: Contains debugging code for analyzing hash collisions
4. **Sort Upgrading**: Maintains the most specific sort information across all instances

## Conclusion

While both classes use hash-based techniques for managing `DagNode` objects:

- **DagNodeSet** is a straightforward set collection that ensures uniqueness
- **HashConsSet** is a more sophisticated structure that implements true hash consing with canonical representation

The key distinction is that `HashConsSet` actively creates and maintains canonical representations of terms, ensuring that structurally identical terms share the same memory, while `DagNodeSet` simply stores unique references without modifying the nodes or ensuring shared representation.
