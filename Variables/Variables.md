The `Module` class itself is not meant to “store” variables in the way you might store sorts or symbols. Instead, variables declared in a module serve as building blocks when you write pre-equations (or rules) that define relationships between the arguments of operators. In other words, variables are used as placeholders or patterns on the left-hand side of equations (or conditions) to indicate what kind of data should match and how the pieces relate to one another.

In the compilation process, when a variable is declared as part of an equation, rule, or sort constraint, it is incorporated into a `PreEquation` object. The `PreEquation` (and its subclasses like `Equation`) uses the variable occurrences in its left-hand side term to capture the relationships between different parts of the term. These relationships are expressed by the structure of the term itself—as the term is constructed, the variable positions (along with their sorts as given by their associated symbol declarations) indicate which parts of the term are “free” (to be matched, substituted, or further constrained) and how arguments within composite operators relate.

Once these pre-equations are set up, the `VariableInfo` class kicks in during preprocessing and during later phases of equation analysis. `VariableInfo` tracks which variables occur, how many times they occur, and where they are bound or used. This information is then “compiled” into the signature and sort constraints of the module. Rather than keeping a persistent list of declared variables, what matters for the final `Module` is the compiled form of the rewrite theory—the operators, equations, rules, and sort constraints that have been derived from the specification (which originally used variables).

So, to summarize:

- Variables appear only as part of the abstract syntax trees for equations (and similar constructs) and as temporary tokens when constructing these relationships. The `Module` instance itself does not keep a separate list of variables.
- The relationships between arguments in pre-equations are specified by how you write the left-hand side of an equation. The variables in that pattern indicate which parts of a term are “holes” to be matched later. Additionally, condition fragments attached to the pre-equation may put further constraints on the allowed substitutions for these variables.
- The `VariableInfo` class and the preprocessing routines (such as those in `PreEquation` and its subclasses) collect and process these variable occurrences. In effect, they “compile away” the variables by turning the pattern relationships—what variables match what parts—into concrete sort relations, operator declarations, and ultimately into the internal representation of the module.

Their initial role is to specify the relationships among operator arguments via the left-hand side patterns and conditional fragments, and these relationships are preserved in the compiled module (even though the original variable declarations are not stored as separate entities in the Module class).

## `VariabeInfo`

`VariableInfo` is a central data structure in Maude’s compilation and rewriting machinery—it’s constructed early in the process when parsing and processing modules, and then used to track and manage all the variable occurrences that appear in equations, rules, and other rewriting artifacts. Here’s a more detailed picture of its construction and usage:

1. Construction and Basic Structure

 - When a module is parsed, each occurrence of a variable (represented as a VariableTerm) in an equation or rule is processed. As these occurrences are encountered, `VariableInfo` is constructed to keep a record of all the “real” variables. Internally, `VariableInfo` maintains a vector (named “`variables`”) that holds pointers to Term objects corresponding to each variable.
 - The function `VariableInfo::variable2Index(VariableTerm* variable)` is called whenever a variable is encountered. It loops through the stored variables to see if the variable already exists (using a method like `variable->equal(…)`). If the variable is found, its index is returned; otherwise, the variable is added to the vector, and its new index is returned. This mapping from a variable’s syntactic identity to a numeric index is crucial for fast comparisons later on.
 - In addition to “real” variables (those that appear explicitly in the source), `VariableInfo` keeps track of “protected” variables. Protected variables are internal representations that are preserved across certain transformation phases (for instance, during matching, rewriting, or garbage collection). The number of protected variables is managed by a counter (`nrProtectedVariables`), and there is a designated range of indices (those below a defined constant like `MAX_NR_PROTECTED_VARIABLES`) that are reserved for these variables.

2. Handling Construction Indices and Fragmentation

 - As the Maude system processes complex rewriting problems, there can arise temporary or “construction” variables—these are assigned indices that are initially beyond the protected range (i.e., indices at or above `MAX_NR_PROTECTED_VARIABLES`). `VariableInfo`’s method `makeConstructionIndex()` creates these indices and records additional bookkeeping information in a separate vector called constructionIndices.
 - Each construction index holds metadata such as the assigned fragment number and the time of its last use during the rewriting or matching process. The `fragmentNumber` is incremented (via `endOfFragment()`) as the system works through different segments or “fragments” of the problem.
 - Later on, when it’s time to “clean up” the temporary allocation, the `computeIndexRemapping()` method is used. This function builds a conflict graph among the construction indices (to record when two indices might interfere with each other) and uses a graph coloring approach to assign new, non-conflicting protected indices to them. As a result, even if a construction variable was used in more than one fragment, its final index (after remapping) guarantees that the substitution or matching routines can be applied correctly.

3. Integration into the Compilation and Rewriting Pipeline

- During compilation, `VariableInfo` is used to compile away the initial syntax-level declarations of variables—turning them into internal indices that can be used in the abstract syntax trees representing equations or rules. This means that although variables may not be stored explicitly in the final Module class, their relationships and occurrences are “compiled into” the operators, equations, and rules that appear in the Module.
- VariableInfo is also used in related components:
  * In substitution management (as seen in `substitution.hh`), the indices produced by `VariableInfo` serve as keys for binding `DagNode` values—effectively linking a variable’s abstract identity to concrete parts of a term or directed acyclic graph.
  * In local matching and binding contexts (for example, within the `LocalBinding` class described in `localBinding.hh`), these indices determine which temporary bindings are active and ensure that substitutions are applied consistently across the various subproblems.
  * In variable abstraction subproblems (like those managed by `VariableAbstractionSubproblem` in `variableAbstractionSubproblem.hh`), the indices and the information in `VariableInfo` help to resolve and constrain how variables are abstracted during pattern matching.

4. Benefits of This Design

- Mapping variables to a simple integer index (and vice versa using `VariableInfo::index2Variable`) provides fast comparison, which is essential when many variables are being matched or substituted during rewriting.
- The abstraction provided by `VariableInfo` decouples the syntactic representation of a variable from its internal handling. Even though variables might only appear as temporary or “helper” constructs during the construction of pre-equations, the information managed by VariableInfo ensures that all occurrences are correctly unified and that variable scopes remain consistent across fragments.
- The management of construction indices (and the subsequent remapping through graph coloring) enables Maude to efficiently handle complex rewriting problems where variables might be reused in different contexts without conflict.

In summary, `VariableInfo` is built at the onset of the compilation process to collect and index all variables encountered in a module. It converts user-declared or syntactically generated variable names into numeric indices, tracks additional temporary “construction” indices during matching, and later remaps them as needed to avoid conflicts. This mechanism ensures that when the `Module` is finally constructed, all the relationships between arguments in pre-equations—expressed via variable occurrences—are faithfully and efficiently encoded for use during rewriting and other execution phases.

# Relationship to Variable Theory

The key relationship is that the variables written in the source syntax are initially parsed and built into `VariableTerm` objects (each of which is paired with a `VariableSymbol`) and then “compiled away” into the internal variable bookkeeping maintained by `VariableInfo`. Here’s how the pieces fit together:

1. Source Syntax to `VariableTerm`/`VariableSymbol`

 - When the source is parsed, variable declarations and occurrences produce `VariableTerm` objects. These objects represent the actual variable terms that you see in equations and pattern matching constructs.
 - Each `VariableTerm` is associated with a `VariableSymbol`. The `VariableSymbol` carries information such as the variable’s sort (type) and other operational properties. In effect, the `VariableSymbol` provides the “definition” or metadata for the variable (for example, its domain and sort restrictions), while the `VariableTerm` is its concrete appearance in the abstract syntax tree.

1. Integrating into `VariableInfo`

 - As the interpreter processes equations and rewriting rules, it needs to keep track of all the variable occurrences. This is where `VariableInfo` comes into play.
 - Each time a `VariableTerm` is encountered (for example, when building the left-hand side of an equation), the `VariableInfo::variable2Index` method is invoked. This method checks whether the variable (the `VariableTerm`, with its associated `VariableSymbol`) has already been seen (using an equality comparison) and, if not, adds it to its internal vector of “real” variables.
 - The mapping that VariableInfo builds (from a `VariableTerm` to a numeric index) is crucial later for operations such as substitution, unification, and local binding during matching.

3. From Syntax to Internal Representation

 - In effect, the variable components in the source syntax are encoded in two stages: first, as VariableTerm objects (with VariableSymbol providing the necessary semantics), and then as entries in `VariableInfo`, where they are assigned numeric indices.
 - These indices are used throughout the rewriting engine to manage variable bindings (for example, within `Substitution` objects and `LocalBinding` structures) and to enforce constraints (like condition variables and unbound variable tracking).

4. Summary of the Relationship

 - The `VariableTerm` and `VariableSymbol` pair capture the surface-level appearance and the semantic properties of source-level variables.
 - When these terms are processed during compilation, `VariableInfo` “compiles” them into a simpler index-based representation. This conversion allows the Maude interpreter to manage variable matching, substitution, and constraint propagation efficiently, using integer indices rather than more heavyweight syntax tree structures.

Thus, the relationship is that source-level variables are first turned into `VariableTerm` objects (backed by `VariableSymbols` for type and metadata), and then these objects are mapped into the internal variable bookkeeping (via `VariableInfo`). This two-step process ensures that all the variable relationships specified in the source are faithfully encoded and can be efficiently used during rewriting and matching.
