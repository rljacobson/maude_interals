# Built-in Types and Their Associated Sorts in Maude

Built-in types in Maude automatically have associated sorts.

## How Sorts Work for Built-in Types

How built-in types are associated with sorts:

1. **Sort Hierarchy**: Every value in Maude, including built-in values, belongs to a sort. Sorts are organized in a hierarchy within connected components.
2. **Automatic Association**: When built-in modules (like `NAT`, `INT`, `FLOAT`, `BOOL`, etc.) are loaded, they automatically define appropriate sorts for their data types:
   - `Bool` sort for boolean values
   - `Nat` sort for natural numbers
   - `Int` sort for integers
   - `Float` sort for floating-point numbers
   - `String` sort for strings

3. **Sort Implementation**: The `Sort` class (defined in `sort.hh` and `sort.cc`) provides the infrastructure for all sorts, including those for built-in types:

   ```cpp
   class Sort : public NamedEntity, public LineNumber, public ModuleItem
   {
     // ...
     Sort(int id);
     // ...
     int index() const;
     ConnectedComponent* component() const;
     // ...
   };
   ```

## Sort Checking for Built-in Types

The `SortTable` class (in `sortTable.hh` and `sortTable.cc`) handles sort checking and computation for operators, including those that work with built-in types:

```cpp
void SortTable::fillInSortInfo(Term* subject) = 0;
```

For built-in types, specialized implementations of this method determine the appropriate sort for terms involving built-in values.

## Sort Testing for Built-in Types

The `SortTestSymbol` class (in `sortTestSymbol.hh` and `sortTestSymbol.cc`) provides functionality to test whether a term belongs to a specific sort:

```cpp
SortTestSymbol::SortTestSymbol(int id, Sort* testSort, FreeSymbol* leq, FreeSymbol* nleq, bool eager)
```

This allows Maude to check if built-in values belong to their expected sorts.

## Example: Boolean Sort

For example, boolean values (`true` and `false`) in Maude:

1. Automatically belong to the `Bool` sort
2. Are part of a connected component that includes the `Bool` sort
3. Have their sort computed and checked using the mechanisms described above

## Subsort Relationships

Built-in types also participate in subsort relationships. For example:

- `Nat` (natural numbers) is a subsort of `Int` (integers)
- `Int` is a subsort of `Rat` (rationals) when the RAT module is imported

These relationships are established automatically when the corresponding modules are imported.

## Conclusion

In summary, built-in types in Maude automatically have associated sorts. This is not an optional feature but a core aspect of Maude's type system. The sort system provides a uniform way to handle both built-in and user-defined types, allowing for type checking, operator overloading, and other type-related features.
