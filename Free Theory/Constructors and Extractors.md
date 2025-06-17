# Extractors and Constructors in Maude's Free Theory Implementation

## Overview

In Maude's implementation of the Free Theory, extractors (`extors`) and constructors (`ctors`) are specialized stack machine instructions that handle the creation, manipulation, and transformation of terms with free symbols. These components are part of Maude's term rewriting engine, handling symbols that don't have special equational properties (like associativity or commutativity).

## Constructors (Ctors)

Constructors are stack machine instructions responsible for creating new term instances with free symbols.

### Purpose and Functionality

1. **Term Creation**: Constructors build new DAG (Directed Acyclic Graph) nodes representing terms with free symbols.
2. **Sort Computation**: They calculate the sort of the newly created term based on the sorts of its arguments.
3. **Optimization**: Different constructor classes exist for different arities (nullary, unary, binary, ternary, and general) to optimize performance.

### Implementation Details

From the provided code, we can see two main constructor implementations:

1. **`FreeGeneralCtor`**: Handles free symbols with more than three arguments:

   ```cpp
   void FreeGeneralCtor::execute(StackMachine* machine) const {
     Frame* frame = machine->getTopFrame();
     FreeDagNode* d = new(NONE) FreeDagNode(symbol);
     
     // Fill out arguments and compute sort
     int nrArgs = d->symbol()->arity();
     DagNode** argArray = d->argArray();
     
     int state = 0;
     for (int i = 0; i < nrArgs; ++i) {
       DagNode* a = frame->getSlot(args[i]);
       int t = a->getSortIndex();
       state = symbol->traverse(state, t);
       argArray[i] = a;
     }
     d->setSortIndex(state);
     
     // Save result and continue execution
     saveResultAndContinue(machine, frame, d);
   }
   ```

2. **Specialized Constructors**: From the `freeInstruction.hh` file, we can see that there are specialized constructors for different arities:
   - `FREE_NULLARY_CTOR` (0 arguments)
   - `FREE_UNARY_CTOR` (1 argument)
   - `FREE_BINARY_CTOR` (2 arguments)
   - `FREE_TERNARY_CTOR` (3 arguments)

## Extractors (Extors)

Extractors are stack machine instructions that handle the application of equations and rules to terms with free symbols.

### Purpose and Functionality

1. **Pattern Matching**: Extractors check if a term matches the left-hand side of an equation or rule.
2. **Equation Application**: They apply matching equations to transform terms.
3. **Rule Execution**: They execute the rewriting logic when a term matches a rule.
4. **Optimization**: Like constructors, different extractor classes exist for different arities to optimize performance.

### Implementation Details

From the provided code, we can see:

1. **`FreeGeneralExtor`**: Handles free symbols with more than three arguments or complex behavior:

   ```cpp
   void FreeGeneralExtor::execute(StackMachine* machine) const {
     Frame* frame = machine->getTopFrame();
     
     // Assemble arguments from stack
     int nrArgs = args.size();
     DagNode** savedArgs = machine->getProtectedScratchpad();
     for (int i = 0; i < nrArgs; ++i)
       savedArgs[i] = frame->getSlot(args[i]);
     
     // Run discrimination net to find applicable equations
     FREE_NET& net = symbol->GET_NET();
     long index = net.findRemainderListIndex(savedArgs);
     
     // If equations found, try to apply them
     if (index >= 0) {
       // ... (equation application logic)
     }
     
     // If no equation applies, create a new term
     FreeDagNode* d = new(NONE) FreeDagNode(symbol);
     // ... (fill out arguments and set sort)
     
     // Save result and continue
     frame->setSlot(getDestinationIndex(), d);
     getNextInstruction()->execute(machine);
   }
   ```

2. **Specialized Extractors**: From `freeInstruction.hh`, specialized extractors exist for different arities:
   - `FREE_NULLARY_EXTOR` (0 arguments)
   - `FREE_UNARY_EXTOR` (1 argument)
   - `FREE_BINARY_EXTOR` (2 arguments)
   - `FREE_TERNARY_EXTOR` (3 arguments)

## Final vs. Non-Final Instructions

Both constructors and extractors can be "final" or "non-final":

1. **Final Instructions**: These are the last instructions in a sequence, often optimized to avoid unnecessary stack frame operations.
   - Examples: `FREE_NULLARY_CTOR_FINAL`, `FREE_UNARY_EXTOR_FINAL`

2. **Non-Final Instructions**: These are followed by other instructions in a sequence.
   - Examples: `FREE_BINARY_CTOR`, `FREE_TERNARY_EXTOR`

## Relationship with RHS Automata

The Free Theory also includes specialized automata for handling right-hand sides of equations:

1. **`FreeRhsAutomaton`**: A general automaton for constructing terms on the right-hand side of equations.
2. **`FreeUnaryRhsAutomaton`**: An optimized automaton specifically for unary symbols.

These automata work with constructors and extractors to implement term rewriting.

## Summary

In Maude's Free Theory implementation:

- **Constructors (Ctors)** create new term instances with free symbols, computing their sorts based on argument sorts.
- **Extractors (Extors)** apply equations and rules to terms with free symbols, using discrimination nets to efficiently find applicable transformations.
- Both are implemented as stack machine instructions with specialized versions for different arities to optimize performance.
- They form the core of Maude's efficient term rewriting engine for free symbols (symbols without special equational properties).

This architecture allows Maude to handle complex term rewriting systems while maintaining a clean separation between term construction and transformation logic.
