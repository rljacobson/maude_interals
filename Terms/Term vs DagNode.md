# Understanding Term vs. DagNode in Maude

## Roles of Term and DagNode

Expression mutation is not exclusive to `DagNode`.
### `Term` Class
- Represents expressions at the syntactic level
- Handles normalization through methods like `Term::normalize()`
- Used during parsing, rule compilation, and pattern preparation
- Maintains information about variables, contexts, and collapse possibilities
- Supports term rewriting at the syntactic level

### `DagNode` Class
- Represents expressions at runtime in a graph structure
- Handles matching and unification operations
- Supports efficient sharing of subexpressions
- Manages runtime reduction through methods like `reduce()`
- Maintains runtime flags like `REDUCED`, `UNREWRITABLE`, etc.

## Evidence from the Code

The code clearly shows that mutation happens in both classes:
### Term Mutation

From `term.cc`:

```cpp
Term* Term::normalize(bool full, bool& changed) {
  // Normalization logic that can transform the term
}
```

This shows that terms can be transformed through normalization at the syntactic level.

### DagNode Mutation

From `AU_Normalize.cc`:

```cpp
AU_DagNode::NormalizationResult AU_DagNode::normalizeAtTop(bool dumb) {
  // Logic that can transform the dag node
  // ...
  if (p < 2) {
    // Eliminating identity causes AU dag node to collapse to its remaining argument
    DagNode* remaining = (s->getPermuteStrategy() == BinarySymbol::EAGER) ?
      argArray[0] : argArray[0]->copyReducible();
    remaining->overwriteWithClone(this);
    return COLLAPSED;
  }
}
```

This shows that DAG nodes can also be transformed during normalization.

## The Real Distinction

The key distinction is not about which one handles mutation, but rather:

1. **When they're used**:
   - `Term` is used during compilation, pattern preparation, and rule definition
   - `DagNode` is used during runtime execution of the rewriting system

2. **How they represent data**:
   - `Term` uses a tree structure with explicit variable representation
   - `DagNode` uses a directed acyclic graph with efficient sharing

3. **Memory management**:
   - `Term` objects are typically allocated on the heap and managed by the compiler
   - `DagNode` objects use specialized memory management (see `MemoryCell` usage)

## Conclusion

`DagNode` is not just for matching - it's the runtime representation of terms that supports efficient execution of the rewriting system. Both `Term` and `DagNode` can undergo mutations, but they do so at different stages of the Maude system's operation.

The relationship is similar to how a compiler might use an abstract syntax tree during compilation but generate optimized bytecode for runtime execution. `Term` is like the AST, while `DagNode` is like the optimized runtime representation.

# Matching in `Terms` Vs `Dagnodes`

Matching in Maude occurs on `DagNode`s rather than `Term`s.

## `Term` to `DagNode` Conversion Flow

1. **Parsing**: Expressions in Maude are first parsed into `Term` objects
   - `Term` is an abstract base class that represents mathematical/logical terms

2. **Conversion**: Before matching, `Term`s are converted to `DagNode`s
   - This conversion happens through the `dagify()` method in the `Term` class
   - Each specific term type implements a `dagify2()` method that creates the appropriate `DagNode`

3. **Matching**: The actual matching operations are performed on the `DagNode` structures

## Evidence from the Code

1. In `term.hh`:
   - `DagNode* dagify()` method is defined to convert a `Term` to a `DagNode`
   - Terms implement `dagify2()` to create their specific `DagNode` representation

2. In various term implementations (like `VariableTerm`, `StringTerm`):
   - Each implements `dagify2()` to create the corresponding `DagNode` type
   - For example, `StringTerm` creates a `StringDagNode`

3. In `DagNode` class:
   - Contains methods for comparison, unification, and pattern matching
   - Has specialized functionality for efficient term rewriting operations

## Why `DagNode`s for Matching?

Matching occurs on `DagNode`s rather than `Term`s for several reasons:

1. **Efficiency**: `DagNode`s implement a Directed Acyclic Graph structure that allows:
   - Sharing of common subterms
   - Avoiding redundant computations
   - More efficient memory usage

2. **State Management**: `DagNode`s maintain state information like:
   - Whether a node is reduced
   - Whether a node is ground (contains no variables)
   - Hash values for quick comparison

3. **Specialized Operations**: `DagNode`s implement specialized methods for:
   - Pattern matching
   - Unification
   - Substitution application

This design separates the parsing/representation concerns (`Term`) from the operational concerns (`DagNode`), allowing each to be optimized for its specific purpose.
