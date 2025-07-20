# Understanding `Symbols` in Maude

## What is a `Symbol`?

In Maude, a `Symbol` is an abstract base class that represents operators in the algebraic specification.

- `Symbol` serves as the foundation for all function and variable symbol definitions within the interpreter
- It inherits from multiple base classes including `RuleTable`, `NamedEntity`, `LineNumber`, `SortTable`,
  `SortConstraintTable`, `EquationTable`, `Strategy`, and `MemoTable`
- Each symbol has properties like a unique identifier, arity (number of arguments it takes), and sort information.

Symbols are essentially the building blocks of terms in Maude's term algebra. They represent both:

1. Function symbols (operators) that can be applied to arguments
2. Constants (operators with zero arguments)

The `Symbol` class in Maude acts as a dispatch hub for theory-specific operations on terms and DAG nodes. While the
operations work on `DagNode` or `Term` objects, the implementation logic is organized within `Symbol` subclasses because
symbols encode the algebraic theory and operational semantics.

## What Gets Parsed into a Symbol?

From the codebase, we can see that various language constructs get parsed into different types of `Symbol` objects:

1. **Operators declared in modules** become specialized symbols:
    - Standard operators become instances of `FreeSymbol`
    - Built-in equality operators become `EqualitySymbol` instances
    - String operations become `StringOpSymbol` instances

2. **Special identifiers**:
    - Quoted identifiers (like `'foo`) are represented by `QuotedIdentifierSymbol`
    - Loop constructs are handled by `LoopSymbol`

3. **Built-in data types**:
    - String literals are managed by `StringSymbol`
    - Numbers, booleans, and other primitive types have their own symbol classes

Each specialized symbol type implements specific behaviors for:

- Term creation (`makeTerm`)
- DAG node creation (`makeDagNode`)
- Pattern matching
- Sort computation
- Rewriting logic

The parser identifies these different constructs in Maude source code and creates the appropriate symbol objects, which
then become part of the interpreter's internal representation of the specification.

## Symbols as Theory-Specific Functionality

Different algebraic theories in Maude have their own `Symbol` subclasses that override core methods:

**Free Theory**: `freeNullarySymbol.cc:52-56` and `freeUnarySymbol.cc:52-57`

**CUI Theory (Commutative, Unit, Idempotent)**: `CUI_Symbol.hh:53-59` shows the theory-specific `eqRewrite` and
`stackArguments` declarations.

**ACU Theory (Associative-Commutative-Unit)**: `ACU_Symbol.hh:45-58` demonstrates how ACU symbols override both
`stackArguments` and `stackPhysicalArguments` for their flattened representation.

**Successor Theory**: `S_Symbol.hh:42-49` shows theory-specific method declarations.

### Concrete Examples of Theory-Specific Operations

**1. eqRewrite Method**: Each theory implements rewriting differently:

- **CUI Theory**: `CUI_Symbol.cc:114-126` shows how CUI symbols handle normalization and equation application.
- **S Theory**: `S_Symbol.cc:137-152` demonstrates successor-specific rewriting with extension info.
- **NA Theory**: `NA_Symbol.cc:61-65` shows a simpler no-argument theory implementation.

**2. stackArguments Method**: Theory-specific argument stacking:

- **CUI Theory**: `CUI_Symbol.cc:280-298` shows how CUI symbols stack their binary arguments while respecting frozen and
  unstackable flags.
- **S Theory**: `S_Symbol.cc:298-309` demonstrates unary argument stacking for successor symbols.

**3. Default Implementation**: The base `Symbol` class provides defaults: `symbol.cc:184-195` shows the default
`stackArguments` does nothing, and `symbol.cc:197-209` shows `stackPhysicalArguments` delegates to `stackArguments`.

### Built-in Data Type Symbols

Built-in types also follow this pattern, as documented in the Data Types and Built-ins wiki. For example, matrix
operations: `matrixOpSymbol.cc:277-365` shows complex theory-specific rewriting for matrix equation solving.

### Design Rationale

This design makes sense for several reasons:

**1. Polymorphic Dispatch**: The `Symbol` serves as the polymorphic interface. When the rewriting engine encounters a
`DagNode`, it calls `subject->symbol()->eqRewrite(subject, context)`, allowing each theory to provide its own
implementation.

**2. Theory Encapsulation**: Each algebraic theory (Free, ACU, CUI, S, etc.) has different operational semantics.
Grouping theory-specific logic in `Symbol` subclasses keeps related functionality together.

**3. Separation of Data and Behavior**: `DagNode` represents the data structure (the term), while `Symbol` contains the
behavior (how to rewrite, normalize, stack arguments). This follows object-oriented design principles.

**4. Extensibility**: New theories can be added by creating new `Symbol` subclasses without modifying existing code, as
shown in the interface declarations: `interface.hh:29-34`

**5. Performance**: The symbol-based dispatch allows the rewriting engine to use theory-specific optimizations. For
instance, ACU symbols can use flattened representations while maintaining the same interface.
