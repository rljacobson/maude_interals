# Narrowing Overview

**Narrowing** in Maude is a symbolic execution technique that generalizes term rewriting by incorporating _unification_. While rewriting applies rules by _matching_ the left-hand side of a rule to a subterm, narrowing applies them by _unifying_ the left-hand side with a subterm. This allows it to reason about terms with variables and find substitutions that make computations possible.

Here’s a breakdown of what that means:

- **Rewriting**: Pattern matching-based; used when terms are fully specified.
    
- **Narrowing**: Unification-based; used when terms include variables or unknowns.
    

Narrowing has several applications in Maude:

1. **Solving equations**: In equational logic, narrowing has traditionally been used to find variable bindings that make two terms equal under a given equational theory.
    
2. **Symbolic reachability analysis**: In rewriting logic, narrowing has been extended to compute reachable states symbolically, i.e., with variables rather than fully instantiated terms .
    
3. **Logic programming**: Maude can be used to interpret logic programs via narrowing. A logic program (like a set of Horn clauses) is translated into a rewrite theory, and queries are evaluated by narrowing search (e.g., using vu-narrow) .
    

Narrowing is particularly powerful when combined with **equational theories** (e.g., associativity, commutativity), allowing symbolic execution modulo those properties. However, because it often involves exploring many potential substitutions and paths, narrowing can be nonterminating and behaves as a _semi-decision procedure_—meaning it may not always halt but is complete for certain problems .

# Concrete Example

Let’s walk through a simple example of narrowing using a functional specification in Maude that defines the **addition of natural numbers** using Peano-style encoding.

### **Step 1: Define a Simple Specification**

We’ll define natural numbers using zero and s (successor), and addition with the operator _+_:

 fmod NAT-ADD is  
   sort Nat .  
   op 0 : -> Nat [ctor] .  
   op s_ : Nat -> Nat [ctor] .  
   op _+_ : Nat Nat -> Nat .  
 ​  
   vars X Y : Nat .  
 ​  
   eq 0 + Y = Y .  
   eq s(X) + Y = s(X + Y) .  
 endfm

### **Step 2: Rewriting vs. Narrowing**

Let’s say we want to compute:

 s(s(0)) + s(0)

With **rewriting**, we apply the second equation:

 s(s(0)) + s(0)  
 → s(s(0) + s(0))  
 → s(s(s(0)))

So the answer is s(s(s(0))) (i.e., 2 + 1 = 3).

### **Step 3: Narrowing Example**

Now let’s consider **narrowing**. Suppose we are given a goal:

 X + s(0) = s(s(s(0)))

We want to find a value for X that makes this equation true. Rewriting can’t help here because X is unknown. But narrowing can!

 Goal: Find X such that  
 X + s(0) = s(s(s(0)))

**Try rule 1:**

 0 + Y = Y  
 ​  
 Try to unify: X + s(0) = s(s(s(0))) with 0 + Y = Y  
 ​  
 X = 0, Y = s(0) → 0 + s(0) = s(0) ≠ s(s(s(0))) → no

**Try rule 2:**

 s(X1) + Y = s(X1 + Y)  
 ​  
 Try to unify: X + s(0) = s(s(s(0))) with s(X1) + Y = s(X1 + Y)  
 ​  
 Unification:  
 ​  
 X = s(X1)  
 Y = s(0)  
 s(X1 + s(0)) = s(s(s(0))) ⇒ must have: X1 + s(0) = s(s(0))

**Apply narrowing recursively to** `X1 + s(0) = s(s(0))`

Repeat:

- Try rule 2 again:
    
    - `X1 = s(X2)`
        
    - Then: `s(X2) + s(0) = s(s(0))`
        
    - Then: `X2 + s(0) = s(0)`
        
- Try rule 2 again:
    
    - `X2 = s(X3)`
        
    - Then: `s(X3) + s(0) = s(0)` ⇒ `X3 + s(0) = 0` → doesn’t work
        
- Try rule 1:
    
    - `X2 = 0`
        
    - `0 + s(0) = s(0)` → match!
        

So, unraveling:

- `X2 = 0` ⇒ `X1 = s(0)` ⇒ `X = s(s(0))`
    

Solution: `X = s(s(0))` (i.e., `2`)

### Summary

Narrowing allowed us to **solve the equation**:

 X + s(0) = s(s(s(0)))

by symbolic execution and unification, discovering:

 X = s(s(0))  →  2 + 1 = 3

This is the classic use of narrowing: finding substitutions that make an equation true, which rewriting alone can’t do since it only simplifies terms, not solve equations.

# Narrowing vs. Equational Rewriting

### Equational Rewriting

- **Mechanism**: Applies **rewrite rules** by **matching** the left-hand side of a rule to a subterm of the input.
    
- **Input**: Ground terms (no variables), or terms where pattern variables are instantiated.
    
- **Matching**: Finds a substitution \sigma such that \ell\sigma = t, where \ell is the left-hand side of a rule and t is a subterm.
    
- **Use case**: Deterministic computation or simplification.
    

**Example**:

 eq s(X) + Y = s(X + Y) .

With term `s(0) + s(0)`, you can match `s(X)` with `s(0)` ⇒ `X = 0`.

### Narrowing

- **Mechanism**: Applies **rewrite rules** by **unifying** the left-hand side with a (possibly variable-containing) subterm.
    
- **Input**: Symbolic terms (may include variables).
    
- **Unification**: Finds a substitution \theta such that \ell\theta = t\theta, where \ell is the left-hand side of a rule and t is the current goal term.
    
- **Use case**: Symbolic execution, equation solving, logic programming, symbolic model checking.
    

**Example**:

To solve `X + s(0) = s(s(s(0)))`, you need to unify `s(X1) + Y` with `X + s(0)`, leading to solving for `X`.

### Summary Table

|**Feature**|**Rewriting**|**Narrowing**|
|---|---|---|
|Uses|Matching|Unification|
|Variables|Typically ground|Symbolic (with vars)|
|Direction|Term → Canonical form|Goal → Solution|
|Purpose|Evaluation|Solving / Searching|

**rewriting = pattern-directed computation**, while **narrowing = goal-directed symbolic search**.

# Narrowing vs. Prolog

Narrowing is more powerful than the search procedure used in Prolog, and here’s why:

### What Prolog Does:

### SLD Resolution

Prolog uses **SLD resolution**, which is a restricted form of **backward chaining** based on:

- **First-order unification** (no equational reasoning),
    
- **Clauses in Horn form** (rules with one positive literal),
    
- **Term matching** against rule heads.
    

**Limitations of Prolog’s search**:

- No built-in reasoning modulo **equations** (like associativity or commutativity).
    
- No support for **conditional rules** or rules modulo axioms.
    
- Logic is fixed: cannot redefine what equality means.
    
- Cannot reduce terms like `a + b` and `b + a` unless explicitly programmed.
    

### What Narrowing Adds

Narrowing generalizes SLD resolution by:

- Supporting **equational theories**: reasoning modulo user-defined equations like associativity (assoc), commutativity (comm), identity (id), etc.
    
- Handling **conditional rules** (e.g., ceq in Maude).
    
- Working over **arbitrary rewrite theories**, not just Horn logic programs.
    
- Being **complete** (under suitable conditions) for equational theories, whereas Prolog’s resolution may miss solutions due to its left-to-right strategy and lack of backtracking across predicates.
    

### Concrete Example

Let’s say you define addition as **commutative**:

 op _+_ : Nat Nat -> Nat [assoc comm id: 0] .

Then:

 X + s(0) = s(s(0))

can have a solution `X = s(0)`, and **narrowing** will find it using commutativity. **Prolog cannot**, unless you explicitly code in commutativity by writing multiple versions of rules to cover all possible orders.

### Summary

|**Feature**|**Prolog (SLD Resolution)**|**Maude Narrowing**|
|---|---|---|
|First-order unification|✅|✅|
|Equational reasoning (e.g., a + b = b + a)|❌|✅|
|Conditional rules|Limited or manual|✅|
|Strategies|Fixed (left-to-right)|Flexible (user-defined)|
|Power|Less powerful|More powerful|

So **narrowing subsumes and extends Prolog’s search procedure**. In fact, narrowing has been called “logic programming with equations,” and Maude can be seen as a **logic programming system over rewriting logic** rather than over plain Horn clause logic like Prolog.