A summary table of comparison operations in the Term class:

| Method/Mechanism                                                                       | Compares                              | Description                                                                                                    |
| -------------------------------------------------------------------------------------- | ------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `bool LessThan::operator()(Term* const& t1, Term* const& t2)`                          | Term vs Term                          | Comparison operator for use with associative containers. Only safe for fully normalized terms from same module |
| `int compare(const Term* other)`                                                       | Term vs Term                          | Compares this term with another term by first comparing top symbols, then arguments if symbols are equal       |
| `int compare(const DagNode* other)`                                                    | Term vs DagNode                       | Compares a term with a DAG node representation                                                                 |
| `bool equal(const Term* other)`                                                        | Term vs Term                          | Tests equality between two terms                                                                               |
| `bool equal(const DagNode* other)`                                                     | Term vs DagNode                       | Tests equality between a term and a DAG node                                                                   |
| `int compareArguments(const Term* other)`                                              | Arguments vs Arguments                | Virtual method for comparing arguments between terms                                                           |
| `int compareArguments(const DagNode* other)`                                           | Arguments vs DagNode args             | Virtual method for comparing term arguments with DAG node arguments                                            |
| `int partialCompare(const Substitution& partialSubstitution, DagNode* other)`          | Partially substituted Term vs DagNode | Runtime heuristic comparison with partial variable bindings                                                    |
| `int partialCompareUnstable(const Substitution& partialSubstitution, DagNode* other)`  | Unstable Term vs DagNode              | Virtual method for partial comparison when term may be unstable                                                |
| `int partialCompareArguments(const Substitution& partialSubstitution, DagNode* other)` | Term args vs DagNode args             | Virtual method for comparing arguments with partial substitutions                                              |
| `bool matchIndependent(const Term* other)`                                             | Term vs Term independence             | Tests if terms are independent for matching purposes                                                           |
| `bool subsumes(const Term* other, bool sameVariableSet)`                               | Term subsumption                      | Tests if this term subsumes another term, considering variable contexts                                        |

The comparison methods use the following return values (defined in the ReturnValues enum):
- GREATER = 1
- LESS = -2 
- EQUAL = 0
- UNKNOWN = -1

## Comparison Mechanisms

We can group the comparison operations into these core mechanisms:

**Group 1 - Symbol-then-Arguments Comparison Pattern**:
- `int compare(const Term* other)` 
- `int compare(const DagNode* other)`
These both follow the same basic algorithm:
   1. First compare top symbols
   2. If symbols are equal, delegate to `compareArguments`

**Group 2 - Direct Equality Testing Pattern**:
- `bool equal(const Term* other)`
- `bool equal(const DagNode* other)`
These appear to use the same mechanism of comparing for exact equality, likely using the compare methods internally and checking if the result is EQUAL.

**Group 3 - Arguments-Only Comparison Pattern**:
- `int compareArguments(const Term* other)`
- `int compareArguments(const DagNode* other)`
These are virtual methods that implement the argument comparison portion of the full comparison, used by the methods in group 1.

**(Non)Group 4- Distinct/Specialized Comparison Mechanisms**:
- `bool LessThan::operator()(Term* const& t1, Term* const& t2)` - This is a specialized comparison for sorted containers, with specific requirements about normalization
- `int partialCompare(const Substitution& partialSubstitution, DagNode* other)` and its "Unstable" variant - These use a different mechanism involving substitutions
- `bool matchIndependent(const Term* other)` - Tests for matching independence, not general comparison
- `bool subsumes(const Term* other, bool sameVariableSet)` - Tests for subsumption relationship, not general comparison

So there are really three pairs of methods that use essentially the same mechanisms (differing only in whether they operate on `Term*` or `DagNode*`), while the others use distinct specialized comparison mechanisms for their specific purposes.

The pairs using the same mechanisms are:
1. `compare()` pair
2. `equal()` pair  
3. `compareArguments()` pair

Each pair differs only in whether they operate on `Term*` or `DagNode*`, but internally use the same logic for their respective comparisons.

