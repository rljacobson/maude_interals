# Structural Hash
A *structural hash* is computed using only an expression's (head) symbol and arguments. (Whether the order and how the multiplicity of arguments is incorporated into the computation depends on the theory.) This hash will *not* depend on metadata, flags, pointers/addresses, and other data that is independent of the expression the data structure represents.

The structural hash is used for:
- `DagNodeSet`
- `PointerSet`
- `HashConsSet` via `PointerSet`
- `TermSet`
## `Term`
The value stored in `Term::hashValue` is a *structural hash* computed using the term's symbol and the hash values of its arguments. The computation is done in `Term::normalize()` one time and then stored. This hash value will not correspond to a derived `Hash` implementation. It's getter is `Term::getHashValue()`.

It's not clear if this hash can or should be used in an `impl Hash for Term`. 
## `DagNode`
The function `DagNode::getHashValue()` computes the structural hash of the DAG node in the same way as the structural hash of its corresponding term's structural hash is computed, except `DagNode`s do not store the structural hash—it is computed anew each time. The structural hash of a `Term` and its corresponding `DagNode` should correspond.  Thus, the algorithms in `DagNode::getHashValue()` and `Term::normalize()` should be kept in sync.

As with `Term`, it's not clear if this hash can or should be used in an `impl Hash for DagNode`.

