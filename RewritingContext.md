# The RewritingContext Class in Maude

## Overview

The `RewritingContext` class manages the state and environment during term rewriting operations. It serves as the operational context in which all rewriting, matching, and equation application takes place.

## Purpose and Functionality

The `RewritingContext` class has several key purposes:

1. **State Management**: It maintains the current state of a rewriting computation, including:
   - The root term/DAG node being rewritten
   - A substitution for variable bindings
   - Counters for various operations (equations, rules, membership axioms)
   - A stack of redex positions for tracking subterms during rewriting

2. **Substitution Handling**: It inherits from `Substitution` to efficiently store and manage variable bindings without creating new substitution objects for each match attempt.
3. **Rewriting Control**: It provides methods to perform different types of rewriting:
   - `reduce()` - Applies only equations and membership axioms
   - `ruleRewrite()` - Applies both equations and rules with a standard strategy
   - `fairRewrite()` - Applies rules using a position-fair strategy
   - `eRewrite()` - Specialized for object-based concurrent systems

4. **Tracing and Debugging**: It includes extensive tracing capabilities that can be enabled to monitor the rewriting process.
5. **Continuation Support**: It maintains state information to allow pausing and resuming computations.

## Implementation Details

The `RewritingContext` is implemented with several important features:

```cpp
class RewritingContext : public Substitution, private SimpleRootContainer
{
    // ...
    DagNode* rootNode;
    Int64 mbCount, eqCount, rlCount, narrowingCount, variantNarrowingCount;
    Vector<RedexPosition> redexStack;
    int currentIndex;
    int staleMarker;
    // ...
}
```

- It inherits from `Substitution` to efficiently reuse the substitution infrastructure
- It maintains counters for different types of operations (membership axioms, equations, rules)
- It uses a stack of redex positions to track the current position in the term being rewritten
- It includes mechanisms to handle "stale" DAG nodes that need rebuilding

## Usage in Maude

The `RewritingContext` is used throughout the Maude system:

1. **Command Execution**: When a user enters a command like `reduce`, `rewrite`, or `frewrite`, the interpreter creates a `RewritingContext` for the term and calls the appropriate method.
2. **Rule Application**: When applying rules, the `RuleTable::applyRules` method uses a `RewritingContext` to:
   - Store variable bindings during matching
   - Track statistics about rule applications
   - Handle tracing if enabled
   - Construct the replacement term

3. **Subcontexts**: During condition evaluation or sort computation, subcontexts are created using `makeSubcontext()` to isolate these computations.
4. **Continuation**: The `cont` command in Maude uses the saved state in a `RewritingContext` to resume a previously paused computation.

## Example Flow

When a user enters a command like `rewrite term`, the following happens:

1. The interpreter creates a `RewritingContext` with `term` as the root node
2. It calls `context.rewrite()` or `context.fairRewrite()` depending on the command
3. The context manages the rewriting process, applying equations and rules
4. It tracks statistics about how many rules and equations were applied
5. When rewriting is complete, the interpreter retrieves the final term using `context.root()`

## Special Features

1. **Limited Contexts**: The class supports "limited" contexts without a root node, used for sort computations and matching when a full context isn't available.
2. **Tracing**: Extensive tracing capabilities can be enabled with `setTraceStatus(true)`, allowing detailed monitoring of the rewriting process.
3. **Interrupt Handling**: Methods like `handleInterrupt()` and `blockAndHandleInterrupts()` allow for graceful handling of user interrupts during long computations.
4. **Performance Optimization**: The design avoids creating new substitutions for each match attempt, significantly improving performance.
