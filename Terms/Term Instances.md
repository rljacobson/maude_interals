# `Term` Instances in Maude

A `Term` instance is created for each appearance of an expression in the source code, unlike `Symbol` instances which are shared.

## Evidence from the Codebase

1. **`Term` Creation Process**:
   - When parsing expressions, each occurrence creates a new `Term` instance
   - This is evident in constructors like `ACU_Term::ACU_Term` and `FreeTerm::FreeTerm` that create fresh instances

1. **`Term` Copying**:
   - Methods like `deepCopy2()` in `ACU_Term` and `FreeTerm` create new instances when terms need to be duplicated
   - These methods wouldn't be necessary if terms were shared

1. **`Term` Destruction**:
   - The `deepSelfDestruct()` methods in term classes show that terms own their subterms
   - This ownership model wouldn't work with shared instances

1. **`Term` Normalization**:
   - The `normalize()` methods modify terms in place
   - This would cause problems if terms were shared across multiple contexts

## Contrast with `Symbol` Sharing

This differs from `Symbol` instances which are shared:

- `Symbols` represent fixed operators with well-defined semantics
- `Term`s represent specific expressions that may be modified during rewriting
- Sharing symbols is efficient since their properties don't change
- Not sharing terms allows each occurrence to be manipulated independently

## Practical Implications

This design choice has important implications:

1. **Memory Usage**: More memory is used since each term appearance creates a new instance
2. **Independence**: Each term can be manipulated without affecting other occurrences
3. **Garbage Collection**: Terms can be safely destroyed when no longer needed
4. **Rewriting**: The term rewriting process can modify terms without side effects

