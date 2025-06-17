# Understanding `RuleTable` in Maude

## Purpose of RuleTable

`RuleTable` is an abstract class that manages a collection of rewrite rules associated with a specific symbol in the Maude system. Its primary responsibilities include:

1. **Rule Organization**: It stores and indexes rules that can be applied to terms headed by a particular symbol.
2. **Rule Application**: It provides the machinery to apply these rules during term rewriting.
3. **Fair Rule Selection**: It implements a round-robin strategy to ensure fair application of rules.

## Key Functionality

From the implementation in `ruleTable.cc` and the interface in `ruleTable.hh`, we can see that `RuleTable` provides:

### 1. Rule Management

```cpp
void offerRule(Rule* rule);
const Vector<Rule*>& getRules() const;
```

These methods allow adding rules to the table and retrieving the collection of rules.

### 2. Rule Compilation

```cpp
virtual void compileRules();
```

This method compiles all rules in the table, preparing them for efficient application during rewriting.

### 3. Rule Application

```cpp
virtual DagNode* ruleRewrite(DagNode* subject, RewritingContext& context);
```

This is the core functionality - it attempts to apply rules to a subject term, returning the result of the first successful rule application.

### 4. Fair Rule Selection

```cpp
int nextRule;  // Member variable
```

The class maintains a `nextRule` index to implement a round-robin strategy, ensuring that rules are tried in a fair order across multiple rewriting steps:

```cpp
nextRule = n + 1;  // After a successful rule application
```

### 5. Rule Acceptance

```cpp
virtual bool acceptRule(Rule* rule) = 0;
```

This pure virtual function (implemented by derived classes) determines which rules are accepted into the table, allowing for specialized indexing strategies.

## Rule Application Process

The rule application process in `applyRules()` is particularly interesting:

1. It iterates through rules in a round-robin fashion
2. For each rule, it:
   - Checks if the rule is executable
   - Attempts to match the rule's left-hand side against the subject term
   - If matching succeeds, checks any conditions on the rule
   - If all checks pass, constructs the right-hand side and returns it
3. If no rule applies, it marks the subject as unrewritable

## Relationship to EquationTable

`RuleTable` has a similar structure to `EquationTable` (also provided in the code), but they serve different purposes in Maude's rewriting logic:

- `EquationTable` manages equations for equational simplification (confluent and terminating)
- `RuleTable` manages rules for rewriting (potentially non-confluent or non-terminating)

This separation reflects Maude's two-level approach to rewriting logic, where equations define equivalence classes and rules define transitions between these classes.

## Conclusion

`RuleTable` provides the infrastructure for organizing, compiling, and applying rewrite rules in a fair manner. As an abstract class, it defines the interface and common functionality for rule application, while allowing derived classes to implement specific indexing and acceptance strategies for different types of symbols.

# Understanding Rule Association with Symbols in Maude

How rules are associated with symbols in Maude when rules typically involve multiple symbols on both sides.

## How Rules Are Associated with Symbols

In Maude, a rule is primarily associated with the **topmost symbol** of the left-hand side (LHS) pattern. This is despite the fact that both sides of a rule usually contain multiple symbols.

Looking at the code in `ruleTable.cc` and `symbol.cc`, we can see this association works as follows:

1. Each `Symbol` object maintains its own `RuleTable` (since `Symbol` inherits from `RuleTable`)
2. When a rule is defined, it's "offered" to the symbol at the top of its LHS pattern
3. The symbol decides whether to accept the rule based on its `acceptRule()` method

## Example

Consider this Maude rule:

```
rl [swap-elements] : f(X:Nat, g(Y:Nat)) => f(g(Y:Nat), X:Nat) .
```

In this rule:

- The LHS contains symbols `f` and `g`
- The RHS contains symbols `f` and `g`
- The topmost symbol on the LHS is `f`

Despite multiple symbols appearing in the rule, this rule is associated only with the `f` symbol. The `RuleTable` for `f` will store and manage this rule.

## How This Works in the Code

From `ruleTable.hh`:

```cpp
void offerRule(Rule* rule);
virtual bool acceptRule(Rule* rule) = 0;
```

When a rule is parsed, the system:

1. Creates a `Rule` object
2. Identifies the topmost symbol of the LHS
3. Calls `offerRule()` on that symbol
4. The symbol's `acceptRule()` method decides whether to add the rule to its table

## Why This Design Makes Sense

This design is efficient because:

1. **Rewriting Efficiency**: When rewriting a term with `f` at the top, Maude only needs to check rules in `f`'s rule table
2. **Pattern Matching**: The topmost symbol is the first thing checked during pattern matching
3. **Theory-Specific Handling**: Different symbols (like ACU symbols) can implement specialized rule handling

## Special Cases: ACU_Symbol Example

Looking at `ACU_Symbol.cc`, we can see how specialized symbols handle rules:

```cpp
DagNode* ACU_Symbol::ruleRewrite(DagNode* subject, RewritingContext& context)
{
  if (ruleFree())
    return 0;
  ACU_ExtensionInfo extensionInfo(safeCast(ACU_BaseDagNode*, subject));
  return applyRules(subject, context, &extensionInfo);
}
```

For ACU symbols (associative-commutative with identity), the symbol provides specialized rule application that handles the theory's properties.

## Conclusion

While rules in Maude typically involve multiple symbols, the system associates each rule with just the topmost symbol of its left-hand side. This allows each symbol to maintain its own collection of applicable rules and implement theory-specific rule application strategies.
