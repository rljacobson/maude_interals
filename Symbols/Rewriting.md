The `Symbol::eqRewrite` method is a virtual function that performs equational rewriting on DAG (Directed Acyclic Graph) nodes. It's the primary mechanism by which symbols apply their equations and built-in operations during term reduction.

## Purpose and Function

The `eqRewrite` method serves as the entry point for applying equations associated with a symbol to reduce terms to their canonical forms. It takes a `DagNode* subject` (the term to rewrite) and a `RewritingContext& context` (providing the rewriting environment), returning a boolean indicating whether any rewriting occurred.

## Implementation Patterns

Different symbol types implement `eqRewrite` with varying strategies:

### Built-in Symbols
For equality operations, `eqRewrite` reduces both arguments and performs the equality test.

### Theory-Specific Symbols
CUI (Commutative, Unit, Idempotent) symbols handle complex reduction strategies.

Successor theory symbols (S_Symbol) implement specialized natural number operations.

### Free Theory Symbols
Simple free symbols apply discrimination net-based rewriting.

## Usage in the Rewriting Process

The method is called during term reduction when the rewriting engine encounters a symbol that needs to apply its equations. The return value indicates whether the subject was modified, allowing the engine to continue reduction if necessary.

The method integrates with Maude's strategy system, supporting both standard eager evaluation and user-defined evaluation strategies.

## Notes

Each symbol type provides specialized implementations of the `eqRewrite` method based on their mathematical properties. The method works in conjunction with the broader rewriting context to ensure termination and confluence properties are maintained during computation.

## Key Differences Between `DagNode::reduce` and `Symbol::eqRewrite`

These functions play distinct but complementary roles in Maude's rewriting system:

1. **`DagNode::reduce`**  
   - Acts as the high-level orchestrator for term reduction  
   - Provides a uniform interface regardless of symbol type  
   - Delegates actual rewriting to `Symbol::eqRewrite`  
   - Called by system operations (narrowing, meta-level operations)  
   - Implemented as an inline non-virtual function in the base class  

2. **`Symbol::eqRewrite`**  
   - Handles low-level, symbol-specific rewriting  
   - Implements equation application and built-in operations  
   - Manages mathematical theories (free, CUI, successor, etc.)  
   - Processes evaluation strategies  
   - called from `DagNode::reduce`

### Control Flow  
The standard execution pattern follows:  
`reduce()` → symbol dispatch → `eqRewrite()` → argument reduction  

This hierarchy ensures clean separation between general reduction logic (`reduce`) and symbol-specific implementations (`eqRewrite`).