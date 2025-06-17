# How Boolean Literals Are Represented Internally in Maude

## Basic Definition in the Prelude

In Maude, boolean literals are defined in the `TRUTH-VALUE` module, which is part of the standard prelude (`prelude.maude`):

```maude
fmod TRUTH-VALUE is
  sort Bool .
  op true : -> Bool [ctor special (id-hook SystemTrue)] .
  op false : -> Bool [ctor special (id-hook SystemFalse)] .
endfm
```

This definition shows that:

1. Boolean values belong to the `Bool` sort
2. `true` and `false` are defined as constant operators (nullary operations)
3. Both are marked with the `ctor` attribute, indicating they are constructors
4. Both have `special` attributes with hooks to system-level implementations

## Internal Representation

### 1. Symbol Definition

Each boolean literal is represented by a `Symbol` object in the C++ implementation. The `ctor` attribute indicates that these symbols are constructors, meaning they create data values directly rather than being defined by equations.

### 2. Special Hooks

The `special` attribute with `id-hook` indicates that these symbols have special handling in the system:

- `SystemTrue` hook for the `true` symbol
- `SystemFalse` hook for the `false` symbol

These hooks connect the Maude-level symbols to their C++ implementations, allowing the system to recognize and handle them efficiently.

### 3. Term Representation

When a boolean literal appears in a term, it's represented as a `DagNode` (Directed Acyclic Graph node) with:

- A pointer to the corresponding `Symbol` (true or false)
- No arguments (since these are nullary operators)

For example, when the literal "false" appears in a term, it's represented by a `DagNode` that points to the `false` symbol defined in the `TRUTH-VALUE` module.

### 4. Term Hooks in Other Modules

The boolean literals are also used as term hooks in other modules. For example, in the `TRUTH` module:

```maude
op _==_ : Universal Universal -> Bool 
      [prec 51 poly (1 2)
       special (id-hook EqualitySymbol
                term-hook equalTerm (true)
                term-hook notEqualTerm (false))] .
```

Here, the `term-hook` attributes specify that the equality operator should use the `true` and `false` symbols as its result values.

## SMT Representation

Maude also provides an alternative representation for boolean values in its SMT (Satisfiability Modulo Theories) integration, defined in `smt.maude`:

```maude
fmod BOOLEAN is
  sort Boolean .
  op true : -> Boolean [special (id-hook SMT_Symbol (true))] .
  op false : -> Boolean [special (id-hook SMT_Symbol (false))] .
  ...
endfm
```

This representation uses a different sort (`Boolean` instead of `Bool`) and different hooks (`SMT_Symbol` instead of `SystemTrue`/`SystemFalse`), allowing these boolean values to be handled by external SMT solvers.

## Summary

In summary, a boolean literal like "False" in Maude is represented internally as:

1. A nullary constructor symbol defined in the `TRUTH-VALUE` module
2. Connected to the C++ implementation via special hooks
3. Represented in terms as a `DagNode` with no arguments
4. Used consistently throughout the system as term hooks for various operations
