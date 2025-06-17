# Elements of Maude's Syntax Parsed as `Symbol`s

Maude's syntax includes several elements that are parsed as `Symbol`s:

## Core Symbol Types

1. **Function Symbols**:
   - The `Symbol` class is an abstract base class for function and variable symbols
   - `FreeSymbol` represents standard symbols in free theories
   - Specialized variants like `FreeNullarySymbol` (for constants) and `FreeUnarySymbol` (for unary operators)

2. **Built-in Operation Symbols**:
   - `EqualitySymbol`: Handles built-in equality operations (`==` and `=/=`)
   - `StringOpSymbol`: Manages string operations (substring, find, case conversion)
   - `FloatSymbol`: Represents floating-point numbers
   - `MatrixOpSymbol`: Handles matrix operations

**Shared symbols**
- Built-in data types have a single symbol per sort.
- Variables have a single symbol per sort

## Symbol Characteristics

The `Symbol` class includes functionality for:

- Pattern matching
- Sort analysis
- Term and DAG node management
- Unification
- Instruction generation

## Specific Syntax Elements

Based on the code, these syntax elements are parsed as `Symbol`s:

1. **Constants**: Represented by `FreeNullarySymbol` (symbols with arity 0)
2. **Operators**: Function symbols with specific arities
   - Unary operators: `FreeUnarySymbol`
   - Binary and n-ary operators: General `FreeSymbol` instances

3. **Built-in Operations**:
   - Equality operators (`==`, `=/=`)
   - String operations (substring, find, uppercase, lowercase)
   - Floating-point operations
   - Matrix operations

4. **Data Type Constructors**: Symbols that construct terms of specific sorts

The Maude interpreter processes these syntax elements by creating appropriate `Symbol` instances, which then handle term creation, pattern matching, rewriting, and other operations specific to each symbol type.

Each `Symbol` has properties like:

- A unique identifier
- An arity (number of arguments)
- Sort information
- Rewriting strategies

Note:

- The variable symbol (an instance of `VariableSymbol`) is created once per sort (typically by `MixfixModule::instantiateVariable`). Its identifier is essentially the sort’s name, so every variable of that sort refers to the same `VariableSymbol`.
- When a variable is parsed from the source, a `VariableTerm` object is created that uses the corresponding `VariableSymbol` but also carries its own name (via the `NamedEntity` constructor). This name reflects the actual variable name as seen in the input, which can differ from the sort’s name.
