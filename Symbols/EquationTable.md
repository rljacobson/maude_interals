# Understanding `EquationTable` in Maude

## Purpose of `EquationTable`

`EquationTable` is an abstract class that manages a collection of equations associated with a specific symbol in the Maude system. Its primary responsibilities include:

1. **Equation Organization**: It stores and indexes equations that can be applied to terms headed by a particular symbol.
2. **Equation Application**: It provides the machinery to apply these equations during term simplification.
3. **Equation Compilation**: It manages the compilation of equations for efficient execution.

## Key Functionality

From the implementation in `equationTable.cc` and the interface in `equationTable.hh`, we can see that `EquationTable` provides:

### 1. Equation Management

```cpp
void offerEquation(Equation* equation);
const Vector<Equation*>& getEquations() const;
bool equationFree() const;
```

These methods allow adding equations to the table, retrieving the collection of equations, and checking if the table is empty.

### 2. Equation Compilation

```cpp
virtual void compileEquations();
```

This method compiles all equations in the table, preparing them for efficient application during rewriting.

### 3. Equation Application

```cpp
bool applyReplace(DagNode* subject, RewritingContext& context, ExtensionInfo* extensionInfo = 0);
bool applyReplaceNoOwise(DagNode* subject, RewritingContext& context, ExtensionInfo* extensionInfo = 0);
```

These are the core functionality methods - they attempt to apply equations to a subject term, returning whether a replacement was successful.

### 4. Equation Acceptance

```cpp
virtual bool acceptEquation(Equation* equation) = 0;
```

This pure virtual function (implemented by derived classes) determines which equations are accepted into the table, allowing for specialized indexing strategies.

## Equation Application Process

The equation application process in `applyReplace()` is particularly interesting:

1. It iterates through equations in the table
2. For each equation, it:
   - Attempts to match the equation's left-hand side against the subject term
   - If matching succeeds, checks any conditions on the equation
   - If all checks pass, replaces the subject with the right-hand side
3. Returns true if any equation was successfully applied

The `applyReplaceNoOwise()` method works similarly but stops before considering equations with the "`owise`" (otherwise) attribute, which are meant to be tried only after all other equations fail.

## Relationship to Equations

Each `Equation` object represents a single equation in Maude, with:

- A left-hand side pattern
- A right-hand side pattern
- Optional conditions
- Various attributes (like "owise" or "variant")

The `EquationTable` organizes these equations and provides the machinery to apply them systematically during term rewriting.

## Conclusion

`EquationTable` provides the infrastructure for organizing, compiling, and applying equations to simplify terms according to the equational theory defined in a Maude specification. Equations define equivalence classes of terms that are considered semantically identical.

## Equation Acceptance Mechanism

The equation acceptance mechanism in `EquationTable` works through two key methods:

### 1. offerEquation

From `equationTable.hh`:

```cpp
inline void EquationTable::offerEquation(Equation* equation)
{
   if (acceptEquation(equation))
     equations.append(equation);
}
```

This method:

- Takes an `Equation` pointer as input
- Calls the `acceptEquation` method to determine if the equation should be added
- If accepted, appends the equation to the internal vector of equations

### 2. acceptEquation

```cpp
virtual bool acceptEquation(Equation* equation) = 0;
```

This is a pure virtual method that:

- Must be implemented by concrete subclasses
- Returns `true` if the equation should be added to the table, `false` otherwise
- Allows subclasses to define their own criteria for equation acceptance

## Comparison with Rule Acceptance

The equation acceptance mechanism works exactly like the rule acceptance mechanism in `RuleTable`:

```cpp
inline void RuleTable::offerRule(Rule* rule)
{
  if (acceptRule(rule))
    rules.append(rule);
}

virtual bool acceptRule(Rule* rule) = 0;
```

Both follow the same pattern of offering an item (rule or equation) and then using a virtual method to determine acceptance.

## Why Would a Table Reject a Rule or Equation?

There are several reasons why a table might reject a rule or equation:

### 1. Symbol-Specific Constraints

Different symbol types (like associative, commutative, or identity symbols) may have specific constraints on the equations or rules they can handle. For example:

- An ACU symbol (associative-commutative with identity) might reject equations that don't respect its algebraic properties
- A built-in symbol might reject equations that conflict with its hardcoded semantics

### 2. Optimization Strategies

Tables might reject rules or equations to optimize rewriting:

- A symbol might reject equations that would be better handled by another symbol
- Equations that would lead to non-terminating rewriting might be rejected

### 3. Theory-Specific Indexing

Different symbols implement different indexing strategies for efficient pattern matching:

- A symbol might reject equations whose left-hand sides don't fit its indexing scheme
- Equations that would require special handling might be directed to specialized tables

### 4. Structural Requirements

Some tables might have structural requirements for the equations they accept:

- Requirements on the form of the left-hand side pattern
- Restrictions on variable occurrences or conditions

## Concrete Example

While the exact acceptance criteria depend on the specific subclass implementation, a typical example might be:

```cpp
bool SomeSymbol::acceptEquation(Equation* equation)
{
  // Only accept equations where this symbol is at the top of the LHS
  Term* lhs = equation->getLhs();
  if (lhs->symbol() != this)
    return false;
    
  // Additional criteria specific to this symbol type
  // ...
  
  return true;
}
```

## Conclusion

The `offerEquation`/`acceptEquation` mechanism provides a way for different symbol types to control which equations they handle. This design allows Maude to organize equations according to the algebraic theories they belong to, optimizing the rewriting process.
