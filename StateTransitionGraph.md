# The `StateTransitionGraph`

## Core Architecture and Function

The `StateTransitionGraph` class is designed to build state transition graphs on-the-fly with hash consing for efficient
memory management during rewrite searches. (`stateTransitionGraph.hh:23-24`)

The class maintains the following internal data structures:

- `Vector<State*> seen` - stores all discovered states
- `Vector<int> hashCons2seen` - maps hash-consed DAG indices to state numbers
- `HashConsSet hashConsSet` - manages canonical DAG representations
- `RewritingContext* initial` - the initial rewriting context (`stateTransitionGraph.hh:69-73`)

### State Representation and Management

Each state in the graph is represented by a `State` struct that contains:

- A hash-consed index pointing to the canonical DAG representation
- A parent state number for path reconstruction
- A vector of next states (successors)
- A `RewriteSearchState` for lazy exploration
- Forward arcs mapping to the rules that generated transitions (`stateTransitionGraph.hh:55-65`)

The constructor initializes the graph with the initial state by reducing the initial context and hash-consing its root
DAG (`stateTransitionGraph.cc:44-54`).

### Lazy State Exploration via getNextState

The heart of the system is the `getNextState` method, which implements lazy exploration of the state space. When you
request the $n$th successor of a state, it either returns a cached result or explores new transitions on-demand.
(`stateTransitionGraph.cc:67-89`)

The method follows this logic:

1. If the requested successor index is already cached, return it immediately.
2. If the state is fully explored, return `NONE`.
3. If no `RewriteSearchState` exists yet, create one with the canonical DAG.

The `RewriteSearchState` is created with specific flags for garbage collection, unrewritable marking, and unstackable
handling to ensure proper rewriting behavior.

### Hash Consing and State Deduplication

When a new transition is found through rewriting, the resulting DAG goes through hash consing to ensure canonical
representation and detect equivalent states (`stateTransitionGraph.cc:130-164`).

The hash consing process:

1. Inserts the replacement DAG into the hash cons set
2. If it's a completely new DAG, creates a new state
3. If the DAG was seen before as a subterm but not as a state, creates a new state
4. If it was seen before as a state, reuses the existing state number

This deduplication prevents exponential blowup in cyclic state spaces.

### Memory Management and Performance

The hash consing mechanism provides several performance benefits:

- Memory efficiency through DAG sharing
- Fast equality checking via pointer comparison of canonical forms
- Automatic cycle detection in the state space

The lazy exploration ensures that only reachable states are computed, and the hash consing prevents redundant state
creation even in highly connected graphs.

## Usage

### Integration with Search Systems

The `StateTransitionGraph` serves as a base class for more specialized search algorithms. For example,
`RewriteSequenceSearch` inherits from it to implement pattern-based search (`rewriteSequenceSearch.hh:32`).

In `RewriteSequenceSearch`, the state transition graph is used to systematically explore reachable states while looking
for matches against a goal pattern. The search maintains exploration state and uses the graph's lazy expansion to avoid
building unnecessary parts of the state space. (`rewriteSequenceSearch.cc:113-203`)

### Model Checking Integration

The class is also used in model checking, where `ModelCheckerSymbol` maintains a `StateTransitionGraph` to represent the
system being verified.

### Usage in Condition Solving

Another important usage is in `RewriteConditionState`, which uses `StateTransitionGraph` to solve rewrite condition
fragments by exploring possible rewrite sequences (`rewriteConditionState.hh:45`, `rewriteConditionState.cc:105-134`).

The condition solver iterates through states and their transitions to find solutions that satisfy the rewrite
conditions.

## Typical Use Case: Pattern-Based Search

A typical use case occurs when Maude performs a search operation to find states that match a given pattern.

### 1. Initialization Phase

The process begins when a `RewriteSequenceSearch` is created, which inherits from `StateTransitionGraph`
`(rewriteSequenceSearch.hh:32)`.

The constructor initializes the state transition graph with an initial rewriting context
(`stateTransitionGraph.cc:44-54`).

At this point, **state 0** represents the initial term that we want to explore from. In practice, this could be
something like `f(a, b)` where `f` is a function symbol and `a`, `b` are constants.

### 2. State Exploration Loop

The main exploration happens in `RewriteSequenceSearch::findNextInterestingState()` (`rewriteSequenceSearch.cc:113-203`)

This method systematically explores states by calling `getNextState(explore, nextArc)` to find successor states.

### 3. Lazy State Generation

When `getNextState` is called, it performs lazy exploration (`stateTransitionGraph.cc:67-89`).

If this is the first time we're exploring transitions from a state, it creates a `RewriteSearchState` to find applicable
rewrite rules. The key insight is that each state represents a **canonical DAG (Directed Acyclic Graph) representing a
term**.

### 4. Hash Consing and State Deduplication

When a rewrite rule is applied and produces a new term, the system hash-conses it to ensure canonical representation
(`stateTransitionGraph.cc:130-164`).

## What "State" Means in Practice

In this concrete example, let's say we start with the term `f(g(x), h(y))` and have rewrite rules:

- `g(x) → a`
- `h(y) → b`
- `f(a, b) → result`

Then:

- **State 0**: `f(g(x), h(y))` (initial term)
- **State 1**: `f(a, h(y))` (after applying first rule)
- **State 2**: `f(g(x), b)` (after applying second rule)
- **State 3**: `f(a, b)` (after applying both rules)
- **State 4**: `result` (after applying final rule)

Each state is a **snapshot of the term at a particular point in the rewriting process**. The transitions between states
represent the application of specific rewrite rules.

## Data Flow Summary

1. **Input**: Initial term + goal pattern + rewrite rules
2. **State Creation**: Terms are canonicalized via hash consing
3. **Transition Discovery**: `RewriteSearchState` finds applicable rules
4. **State Deduplication**: Hash consing prevents duplicate states
5. **Pattern Matching**: Each new state is checked against the goal pattern
6. **Result**: Sequence of states leading to a match (if found)

The `StateTransitionGraph` efficiently manages this process by only computing transitions on-demand and sharing
identical terms through hash consing, making it suitable for exploring potentially infinite state spaces in rewriting
systems.

## The algorithm of `getNextState()`

The `StateTransitionGraph::getNextState(int stateNr, int index)` method implements an on-demand state space exploration
algorithm for building state transition graphs during rewriting operations . This method is part of Maude's model
checking and search infrastructure, providing lazy generation of successor states in a rewrite system
(`stateTransitionGraph.cc:67-68`) .

### Algorithm Overview

The algorithm maintains a state transition graph where states are DAG nodes and transitions are rewrite rules
(`stateTransitionGraph.hh:24-25`) . It uses hash consing for canonical term representation and lazy exploration to
generate successor states only when requested (`stateTransitionGraph.hh:30`) .

### Detailed Algorithm Steps

#### 1. Initial State Lookup and Validation

The method first retrieves the current state and checks if the requested successor already exists
(`stateTransitionGraph.cc:70-75`) :

- If `index` is within the existing `nextStates` array, return the cached successor
- If the state is `fullyExplored`, no more successors exist, return `NONE`

#### 2. Lazy RewriteSearchState Creation

If no `RewriteSearchState` exists for the current state, create one with specific flags
(`stateTransitionGraph.cc:76-89`) :

- `GC_CONTEXT`: Enable garbage collection for the rewriting context
- `SET_UNREWRITABLE` and `RESPECT_UNREWRITABLE`: Mark and respect unrewritable terms
- `SET_UNSTACKABLE` and `RESPECT_UNSTACKABLE`: Handle unstackable DAG nodes

These flags ensure proper handling of frozen arguments and optimization constraints during rewriting
(`ChangeLog:443-445`) .

#### 3. Successor State Generation Loop

The core exploration loop continues until enough successors are found (`stateTransitionGraph.cc:93-95`) :

For each potential rewrite:

- Call `findNextRewrite()` to find the next applicable rule
- Transfer rewrite counts to maintain statistics (`stateTransitionGraph.cc:96`)

#### 4. Rewrite Application and Tracing

When a rewrite is found (`stateTransitionGraph.cc:98-121`) :

- Extract the rule and handle optional tracing
- Apply the rewrite to get the replacement term
- Rebuild the DAG structure using `rebuildDag()`
- Create a new rewriting context and perform reduction
- Update rewrite counters and handle trace output

#### 5. Hash Consing and State Deduplication

The algorithm uses hash consing to ensure canonical representation (`stateTransitionGraph.cc:131-164`) :

- Insert the resulting DAG into the `hashConsSet`
- Check if this creates a new state or references an existing one
- Extend the `hashCons2seen` mapping if needed
- Create new `State` objects only for genuinely new states

#### 6. Transition Recording

For each successful rewrite (`stateTransitionGraph.cc:165-172`) :

- Add the successor state to `nextStates`
- Record the rule in the forward arc map (`fwdArcs`)
- Enable garbage collection after non-equational rewrites

#### 7. Termination Handling

When no more rewrites are possible (`stateTransitionGraph.cc:174-180`) :

- Mark the state as `fullyExplored`
- Clean up the `RewriteSearchState`
- Return `NONE` to indicate no more successors

### Integration with Search Infrastructure

This method is used by higher-level search algorithms like `RewriteSequenceSearch` (`rewriteSequenceSearch.cc:157`) and
model checking components (`modelCheckerSymbol.hh:61`) . The lazy generation approach allows these algorithms to explore
large state spaces efficiently without pre-computing all possible states.
