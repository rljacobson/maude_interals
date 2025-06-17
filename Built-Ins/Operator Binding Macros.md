# Binding Macros in Maude: How Built-in Operators Work

The macros in `bindingMacros.hh` are part of Maude's implementation of built-in operators. They provide a mechanism for binding operations, symbols, and terms to built-in functionality. 
## Core Purpose of Binding Macros

The binding macros serve several functions:

1. **Operation Code Encoding**: Convert string operation names into compact numeric codes
2. **Symbol Binding**: Associate symbols with specific operations
3. **Term Binding**: Attach terms to symbols for use in rewriting
4. **Data Attachment**: Connect operation declarations with their implementations

## Key Macros and Their Functions

### 1. Code Encoding Macros

```cpp
#define CODE(c1, c2)    ((c1) + ((c2) << 8))
#define CODE3(c1, c2, c3)    ((c1) + ((c2) << 8) + ((c3) << 16))
```

These macros encode operation names (like "+", "-", "abs") into compact numeric codes:

- `CODE` encodes 1-2 character operations
- `CODE3` encodes 3 character operations

**Example**: In `FloatOpSymbol`, the operation "abs" is encoded as `CODE('a', 'b')`, which translates to the ASCII value of 'a' plus the ASCII value of 'b' shifted left by 8 bits.

### 2. Operation Binding Macros

```cpp
#define BIND_OP(purpose, className, op, data)
#define BIND_OP3(purpose, className, op, data)
```

These macros bind string operation names to their numeric codes.

**Example from `floatOpSymbol.cc`**:

```cpp
bool FloatOpSymbol::attachData(const Vector<Sort*>& opDeclaration,
                              const char* purpose,
                              const Vector<const char*>& data)
{
  BIND_OP(purpose, FloatOpSymbol, op, data);
  return FreeSymbol::attachData(opDeclaration, purpose, data);
}
```

When this code executes:

1. It checks if `purpose` equals "`FloatOpSymbol`"
2. If so, it extracts the operation name from `data`
3. It converts the operation name to a numeric code
4. It assigns this code to the `op` member variable

### 3. Symbol Binding Macros

```cpp
#define BIND_SYMBOL(purpose, symbol, name, symbolType)
#define BIND_SYMBOL2(purpose, symbol, name, symbolType, nrArgs)
```

These macros bind symbols to specific operations.

**Example from `stringOpSymbol.cc`**:

```cpp
bool StringOpSymbol::attachSymbol(const char* purpose, Symbol* symbol)
{
  BIND_SYMBOL(purpose, symbol, stringSymbol, StringSymbol*);
  BIND_SYMBOL(purpose, symbol, succSymbol, SuccSymbol*);
  // ...other bindings...
  return FreeSymbol::attachSymbol(purpose, symbol);
}
```

This code:

1. Checks if `purpose` matches a specific symbol name (e.g., "stringSymbol")
2. If so, it either verifies the symbol matches an existing one or assigns the new symbol
3. It ensures the symbol is of the correct type using dynamic casting

### 4. Term Binding Macros

```cpp
#define BIND_TERM(purpose, term, name)
```

This macro binds terms to symbols for use in rewriting.

**Example from `floatOpSymbol.cc`**:

```cpp
bool FloatOpSymbol::attachTerm(const char* purpose, Term* term)
{
  BIND_TERM(purpose, term, trueTerm);
  BIND_TERM(purpose, term, falseTerm);
  return FreeSymbol::attachTerm(purpose, term);
}
```

This code:

1. Checks if `purpose` matches a specific term name (e.g., "trueTerm")
2. If the term is already bound, it verifies equality
3. If not, it binds the new term

## Practical Example: How Float Operations Work

Let's trace how the float addition operation (`+`) works:

1. **Initialization**: When `FloatOpSymbol` is created, its `op` field is set to `NONE`
2. **Binding**: During module loading, `attachData` is called with:

   ```
   purpose = "FloatOpSymbol"
   data = ["+"]
   ```

3. **Code Assignment**: The `BIND_OP` macro:
   - Checks if purpose is "FloatOpSymbol"
   - Extracts "+" from data
   - Converts "+" to a numeric code using `CODE('+', 0)`
   - Assigns this code to the `op` field

4. **Symbol Binding**: `attachSymbol` is called to bind required symbols:

   ```cpp
   BIND_SYMBOL(purpose, symbol, floatSymbol, FloatSymbol*);
   ```

   This binds the float symbol needed for creating float values

5. **Rewriting**: When `eqRewrite` is called on a term like `2.5 + 3.7`:
   - It reduces the arguments to float values
   - It checks the `op` field to determine which operation to perform
   - For `+`, it performs floating-point addition: `a1 + a2`
   - It creates a new float value with the result

## String Operations Example

String operations work similarly but often use `CODE3` for longer operation names:

```cpp
bool StringOpSymbol::attachData(const Vector<Sort*>& opDeclaration,
                               const char* purpose,
                               const Vector<const char*>& data)
{
  BIND_OP3(purpose, StringOpSymbol, op, data);
  return FreeSymbol::attachData(opDeclaration, purpose, data);
}
```

For an operation like "length", the code would be:

```
CODE3('l', 'e', 'n')
```

## Benefits of This Approach

This macro-based approach offers several advantages:

1. **Efficiency**: Operation codes are compact and fast to compare
2. **Flexibility**: New operations can be added without changing the core architecture
3. **Type Safety**: Dynamic casting ensures symbols are of the correct type
4. **Modularity**: Each built-in module can define its own operations independently

In summary, the binding macros in Maude provide a mechanism for implementing built-in operations, allowing the system to handle a wide range of data types and operations while maintaining a modular architecture.

# Connecting Maude's Prelude to Internal Implementation: Float Addition

The addition operation for floating-point numbers in Maude's prelude:

```maude
op _+_ : Float Float -> Float
      [prec 33 gather (E e)
       special (id-hook FloatOpSymbol (+)
                op-hook floatSymbol (<Floats> : ~> Float))] .
```

This declaration connects directly to the internal C++ implementation through several mechanisms:

## 1. The `special` Attribute and Hook Mechanism

The `special` attribute creates a bridge between Maude's surface syntax and the underlying C++ implementation:

- `id-hook FloatOpSymbol (+)` connects this operation to the `FloatOpSymbol` C++ class, passing `'+'` as the operation code
- `op-hook floatSymbol (<Floats> : ~> Float)` connects to the `FloatSymbol` class that handles float constants

## 2. Internal Implementation Flow

When Maude evaluates an expression like `2.5 + 3.7`, the following happens internally:

1. **Term Creation**:
   - `FloatTerm` objects are created for `2.5` and `3.7`
   - These terms store the actual double values (2.5 and 3.7)

2. **DAG Node Creation**:
   - `FloatDagNode` objects are created from these terms
   - Each `FloatDagNode` stores both a symbol pointer and the actual double value
   - A `FreeDagNode` for the `+` operation is created with the two float nodes as arguments

3. **Rewriting Process**:
   - During rewriting, `FloatOpSymbol::eqRewrite` is called (from `floatOpSymbol.cc`)
   - The method extracts the float values directly from the `FloatDagNode` arguments:

     ```cpp
     double a1 = static_cast<FloatDagNode*>(d->getArgument(0))->getValue();
     double a2 = static_cast<FloatDagNode*>(d->getArgument(1))->getValue();
     ```

   - It performs the addition: `r = a1 + a2;`
   - It creates a new `FloatDagNode` with the result via:

     ```cpp
     return floatSymbol->rewriteToFloat(subject, context, r);
     ```

## 3. The Two-Layer Architecture in Action

The code demonstrates how the two-layer architecture (Term and DagNode) works:

- **Term Layer**: `FloatTerm` objects represent the abstract syntax of float values
- **DAG Node Layer**: `FloatDagNode` objects represent concrete nodes in the rewriting system
- Both layers store the actual double values, allowing for efficient operations

## 4. Comparison Mechanism

When comparing float values (which might happen during pattern matching or other operations):

- The `compareArguments` method in `FloatDagNode` is called
- It directly accesses the stored float values from both nodes:

  ```cpp
  int FloatDagNode::compareArguments(const DagNode* other) const {
    double otherValue = static_cast<const FloatDagNode*>(other)->getValue();
    // Compare values and return -1, 0, or 1
  }
  ```

## 5. Connection to `floatOpSymbol.cc`

In the provided `floatOpSymbol.cc` file, we can see the implementation of the addition operation:

```cpp
case '+':
  r = a1 + a2;
  break;
```

This is part of a switch statement in the `eqRewrite` method that handles all float operations, including addition.

## Summary

The Maude prelude declaration creates a high-level interface for float addition that users can work with, while the C++ implementation provides the actual functionality. The connection between these layers is established through the hook mechanism, which maps Maude operations to C++ classes and methods.