The sort diagram is a data structure owned by the `SortTable` used by the Maude term rewriting system to figure out what sorts (types) are valid for a given symbol and its arguments. Think of this as building a decision table that makes sort checking during rewriting fast and correct.

## Step-by-Step Explanation of What’s Happening

### Step 1: What Are We Building?

We’re building a sort diagram for a symbol (like a function) that tells Maude:

- “If the arguments have these sorts, then the result sort must be X.”
- It efficiently encodes all valid sort combinations for a symbol.

### Step 2: Start with All the Operator Declarations

Each symbol can be declared multiple times with different sorts (polymorphism). So we start by gathering all the declarations for the symbol.

```
for (int i = nrDeclarations - 1; i >= 0; i--)
  all.insert(i);
```

We’re saying: “At first, all declarations are possible.”

### Step 3: Special case for Constants

If the symbol takes no arguments, then it’s just a constant. In that case, we look at all the declarations and pick the _least_ sort that fits all of them. That becomes the result.

### Step 4: Handle Symbols with Arguments

Now we walk through the symbol’s arguments from left to right. For each argument:

- We look at all the sorts that this argument could be (from the `ConnectedComponent`).
- For each sort, we figure out which declarations are still viable (i.e., they match this argument’s sort).
- Then we match these viable declarations against the current set of states we’re tracking, updating the diagram.

We use **`NatSet`** to track which operator declarations are still possible based on the argument sorts we’ve seen so far.

### Step 5: Build Transitions for Each State

Each state in the diagram represents a partial combination of argument sorts. For every current state:

- Intersect it with the set of viable declarations for the new argument.
- That gives us the possible operator declarations if we extend the arguments with the new sort.
- This becomes a transition in the diagram.

If this isn’t the final argument, we compute what state to go to next.

If it is the final argument, we compute what sort the result should be, based on the remaining viable declarations.

### Step 6: Computing the Result Sort

When we reach the last argument position, we ask:

> “Of all the remaining declarations, what’s the most specific (minimal) sort for the result?”

This is handled by `findMinSortIndex`. It ensures the result sort is consistent and as specific as possible, and it checks whether it is unique (important for disambiguation).

### Step 7: Store Everything in a Table

Each entry in the sort diagram is either:

- A pointer to the next state (if we’re still processing arguments), or
- A result sort index (if we’ve reached the last argument).

This makes runtime checking simple: walk through the diagram using the sorts of the actual arguments, and you’ll land at a result sort.

## Step 8: Using the Sort Diagram at Runtime

When rewriting terms, Maude can now do:

```
int resultSort = sortDiagram[position + sortIndex];
```

This avoids having to recompute or search among all declarations—it just follows the diagram.

## Why This Data Structure

- **Fast runtime**: Maude doesn’t have to scan all declarations each time; it just traverses a precomputed diagram.
- **Handles polymorphism**: Multiple declarations are handled by encoding their combinations.
- **Error detection**: If the sorts don’t match any declaration, Maude can detect it early.
- **Compact representation**: It flattens a potentially complex decision tree into an array of transitions.

# Example

Let’s walk through a **concrete example** with pretend sorts and operator declarations to see exactly how this sort diagram gets built.

## Pretend Scenario

We’ll define:

- **Sorts**: `Nat` < `Int` < `Any` (so `Nat` is a subtype of `Int`, and `Int` is a subtype of `Any`)
- **Operator**: `f` with two arguments (binary)
- **Declarations for `f`**:

| **Declaration #** | **Argument Sorts** | **Result Sort** |
| ----------------- | ------------------ | --------------- |
| `0`               | `Nat`, `Nat`       | `Nat`           |
| `1`               | `Int`, `Int`       | `Int`           |
| `2`               | `Any`, `Any`       | `Any`           |
|                   |                    |                 |

So `f` is overloaded—it can be used with different sorts of arguments.

---

## Step-by-Step Walkthrough

Let’s build the sort diagram for f. We’ll go one argument at a time.

### Initial State

- All 3 declarations are active: {0, 1, 2}
- Our first argument is position 0 (leftmost)

### Step 1: Argument 0 (First argument)

We look at all the sorts this argument could have. Say we try:

#### Case 1: Argument 0 = Nat

- Which declarations match this?

→ Declaration 0 (Nat) ✅

→ Declaration 1 (Int) ✅ (because Nat < Int)

→ Declaration 2 (Any) ✅ (because Nat < Any)

→ So viable = {0, 1, 2}

We now intersect this with the current state {0,1,2} → still {0,1,2}

This becomes one branch in the diagram.

**Case 2: Argument 0 = Int**

- Which declarations match?

→ Declaration 1 ✅

→ Declaration 2 ✅

→ viable = {1, 2}

Intersect with {0,1,2} → {1, 2}

Another branch.

#### Case 3: Argument 0 = Any

- Matches only Declaration 2

→ viable = {2}

Intersect → {2}

Third branch.

### Step 2: Argument 1 (Second argument)

Now for each state from step 1, we branch again based on the second argument’s sort.

**For state {0,1,2} and second arg = Nat:**

- Declaration 0 → second arg is Nat ✅
- Declaration 1 → second arg is Int ✅ (since Nat < Int)
- Declaration 2 → Any ✅

→ viable = {0,1,2} ∩ {0,1,2} = {0,1,2}

Use this to compute the result sort:

- Result sorts: Nat, Int, Any
- Intersection of all result sorts’ subtypes = just Nat

→ Result sort = Nat

**For state {1,2} and second arg = Int:**

- Declaration 1 → arg2 = Int ✅
- Declaration 2 → Any ✅

→ viable = {1,2}

Result sorts = Int and Any

→ Intersection = Int

→ Result sort = Int

**For state {2} and second arg = Any:**

- Declaration 2 only → Any ✅

→ viable = {2}

Result = Any

---

### Final Sort Diagram Layout

This builds a little decision table that might look like this:

| **Arg 0 Sort** | **Arg 1 Sort** | **Result Sort** |
| -------------- | -------------- | --------------- |
| Nat            | Nat            | Nat             |
| Int            | Int            | Int             |
| Any            | Any            | Any             |

And in the sort diagram array, we’d encode:

- For each argument sort at position 0, what state we go to
- For each of those, what final result sort we get when given the second argument

## At Runtime

Now when Maude sees a term like f(3, 5):

1. It determines the sorts: Nat, Nat
2. Starts at position 0, follows the sort diagram using Nat → goes to state {0,1,2}
3. Takes next sort Nat, follows to final state → result sort = Nat
