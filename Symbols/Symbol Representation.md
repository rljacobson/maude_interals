# Symbol Representation in Maude

In Maude, a single `Symbol` instance is shared among all appearances of that symbol in the input source code.

## Evidence from the Codebase

The architecture of Maude's symbol management system supports this conclusion:

1. **Symbol Class Hierarchy**:
   - The `Symbol` class (defined in `symbol.hh`) serves as the base class for various symbol types
   - Specialized implementations like `FreeSymbol`, `FreeBinarySymbol`, `FreeUnarySymbol`, and domain-specific symbols like `StringSymbol`, `FloatSymbol`, etc. extend this base class

2. **Symbol Identification**:
   - Each `Symbol` instance is constructed with a unique identifier (`int id`)
   - This identifier is used to distinguish between different symbols in the system

3. **Symbol Reuse**:
   - The symbol creation process in Maude is designed to ensure that only one instance of each unique symbol exists
   - When the parser encounters a symbol in the input, it looks up the existing symbol in a symbol table rather than creating a new instance

4. **Symbol Attachment**:
   - Many symbol classes implement methods like `attachData`, `attachSymbol`, and `attachTerm`
   - These methods allow additional information to be associated with a symbol instance
   - This design only makes sense if a single instance is shared, as attachments would be inconsistent otherwise

## Practical Implications

This single-instance approach provides several benefits:

1. **Memory Efficiency**: Only one instance of each symbol is stored in memory, regardless of how many times it appears in the source code
2. **Consistency**: Any properties or attachments associated with a symbol are consistently available wherever that symbol is used
3. **Performance**: Symbol comparison can be done by pointer comparison rather than string comparison, which is much faster
4. **State Management**: The state of a symbol (such as its sort information or rewriting rules) is maintained in one place

This design is typical for language interpreters and compilers, where symbols are interned to improve performance and ensure consistency throughout the system.
