# How `DagNode` Comparison Works in Maude's `NA_Theory`

## Data Is Duplicated in Both Layers

For built-in data types in Maude, the actual value is stored in both the `Term` and the `DagNode`. This is evident from examining the code:

### For Float Values

In `floatDagNode.hh`:

```cpp
class FloatDagNode : public NA_DagNode {
private:
  const union {
    double value;
    Int64 bitPattern;
  };
};
```

In `floatTerm.hh`:

```cpp
class FloatTerm : public NA_Term {
private:
  const union {
    double value;
    Int64 bitPattern;
  };
};
```

### For String Values

In `stringDagNode.hh`:

```cpp
class StringDagNode : public NA_DagNode {
private:
  const Rope value;  // assume that sizeof(Rope) <= DagNode::nrWords
};
```

In `stringTerm.hh`:

```cpp
class StringTerm : public NA_Term {
private:
  const Rope value;
};
```

## How Comparison Works

The comparison between `DagNode` instances works because:

1. Each specialized `DagNode` subclass (like `FloatDagNode` or `StringDagNode`) implements its own `compareArguments` method:

   ```cpp
   int compareArguments(const DagNode* other) const;
   ```

2. Inside this method, the implementation can directly access its own value and cast the `other` parameter to the appropriate type to access its value:

   ```cpp
   // Hypothetical implementation for FloatDagNode
   int FloatDagNode::compareArguments(const DagNode* other) const {
     const FloatDagNode* otherFloat = static_cast<const FloatDagNode*>(other);
     if (value < otherFloat->value)
       return -1;
     if (value > otherFloat->value)
       return 1;
     return 0;
   }
   ```

3. The comparison is type-safe because the system ensures that `compareArguments` is only called when comparing nodes of the same type.

## The Two-Layer Architecture

This design reflects Maude's two-layer architecture for term representation:

1. **Term Layer**: Represents the abstract syntax of terms and is used for pattern matching, normalization, and term construction.
2. **DAG Node Layer**: Represents the concrete instances of terms in the rewriting system and is used for efficient rewriting and comparison.

The duplication of data between these layers is a deliberate design choice that allows each layer to efficiently perform its specific operations without needing to constantly convert between representations.

## Benefits of This Approach

This approach offers several advantages:

1. **Performance**: Direct access to values in both layers avoids indirection costs.
2. **Simplicity**: Each layer can operate independently without complex cross-layer references.
3. **Specialization**: Each layer can optimize its representation for its specific use cases.

In summary, comparison between `DagNode` instances works because the actual data values are duplicated in both the `Term` and `DagNode` layers, allowing direct access and comparison without needing to reference the other layer.
