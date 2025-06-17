# **Arity In Maude: A Unified Explanation**

In the Maude term-rewriting system, **arity** is the fixed number of arguments that a symbol accepts. Although users may write expressions that _look_ variadic (e.g., associative operators applied to many arguments), Maude’s internal representation and compilation machinery rely on a strictly fixed-arity model. This design choice underpins key optimizations in memory layout, term construction, pattern matching, and generated C++ code.

---

## **1 Arity as an Invariant Symbol Property**

- **Set at construction.** Every Symbol object is created with an explicit arity value. The constructor stores it in an internal field (orderInt), positioning the arity bits high in the ordering word so it becomes part of the symbol’s identity and total ordering.

- **Immutable.** Once assigned, a symbol’s arity never changes. All later checks—during parsing, DAG node construction, and compiler back-end code generation—assume this value is correct.

- **Validation on use.** When the parser or compiler reads a term, it instantiates exactly arity argument subterms before calling makeDagNode(). Any mismatch is a hard error, preventing malformed terms from entering later phases.

---

## **2 Theory-Specific Conventions**

Different Maude theories impose their own fixed arities:

|**Theory concept**|**Example arity**|**Rationale**|
|---|---|---|
|**Variables**|0|A variable stands alone and takes no subterms.|
|**Non-algebraic (“built-in”) symbols**|0|They behave like constants computed by the engine.|
|**CUI (Commutative-Unital-Idempotent) symbols**|2|Binary union/merge operators.|
|**S (Set) symbols**|1|Unary containers (e.g., set insertion).|

Associative-commutative-unit (ACU) operators, discussed next, are a special case that demonstrates how Maude _appears_ to relax fixed arity without actually doing so.

---

## **3 How “Variadic” Operators Really Work**

Associative (or ACU) operators let users write f(a, b, c) even though f is declared with arity 2. Maude reconciles the difference by **flattening**:

```
f(a, b, c)   ➜   f(f(a, b), c)
```

Internally, the term is a right- (or left-) nested chain of binary applications. The theory’s matching and rewriting algorithms are _flattening-aware_, so they treat all the nested structures as equivalent to the user’s flat notation. Thus:

- **No symbol ever truly has variable arity.**

- **All compiler tables and argument-count loops remain simple, because arity is still 2.**

---

## **4 Roles of Arity Throughout the Toolchain**

|**Phase**|**How arity is used**|
|---|---|
|**Parsing / DAG construction**|Determines how many child nodes to read before calling makeDagNode().|
|**Ordering & hashing**|Embedded in orderInt; ensures fast comparison and consistent hashing.|
|**Pattern matching / unification**|Guides iterator bounds when traversing children.|
|**Code generation**|The compiler emits C++ functions whose parameter lists have exactly arity slots, enabling direct stack allocation and inlining.|
|**Memory layout optimizations**|Fixed arity allows pre-sized node headers and contiguous argument arrays, reducing pointer chasing.|

---

## **5 Why Fixed Arity Matters**

1. **Performance.** Knowing the exact child count lets Maude allocate term nodes tightly and generate efficient C++ with minimal indirection.

2. **Simpler algorithms.** Traversals, substitutions, and unification run on predictable, bounded loops instead of dynamic arrays.

3. **Theory modularity.** Each algebraic theory (ACU, CUI, etc.) can add clever interpretations—such as flattening—_without_ complicating the core term representation.

---

## **6 Takeaways**

- Every Maude symbol is created with **one immutable arity**.

- Operators that _appear_ to accept any number of arguments are handled by **theory-level flattening** over fixed-arity binary symbols.

- This disciplined approach enables fast parsing, compact memory usage, and straightforward code generation, all of which would be harder with true variadic symbols.

The fixed-arity model is therefore not a limitation but a deliberate architectural decision that powers many of Maude’s performance advantages.
