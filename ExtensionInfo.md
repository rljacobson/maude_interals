# Understanding ExtensionInfo in Maude

## What is ExtensionInfo?

`ExtensionInfo` is an abstract base class that records and manages information about which parts of a term were matched and which parts remained unmatched during pattern matching operations. It serves as a foundation for theory-specific extension information classes like `ACU_ExtensionInfo` and `AU_ExtensionInfo`.

From the code, we can see that `ExtensionInfo` defines a minimal interface:

```cpp
class ExtensionInfo
{
public:
  virtual ~ExtensionInfo() {}

  bool validAfterMatch() const;
  void setValidAfterMatch(bool status);

  bool matchedWhole() const;
  void setMatchedWhole(bool status);

  virtual DagNode* buildMatchedPortion() const = 0;
  virtual ExtensionInfo* makeClone() const = 0;
  virtual void copy(const ExtensionInfo* extensionInfo) = 0;

private:
  bool validAfterMatchFlag;
  bool matchedWholeFlag;
};
```

## Purpose of ExtensionInfo

The primary purposes of `ExtensionInfo` are:

1. **Track Matching Status**: It records whether a term was completely matched (`matchedWholeFlag`) or only partially matched.
2. **Build Matched Portions**: It provides functionality to construct a term representing the portion of the subject that was successfully matched.
3. **Support Backtracking**: Through its `makeClone()` and `copy()` methods, it enables saving and restoring the state of matching operations during backtracking.
4. **Theory-Specific Extensions**: It serves as a base for theory-specific implementations that handle the unique matching requirements of different equational theories.

## How ExtensionInfo is Used

### In Pattern Matching

During pattern matching, an `ExtensionInfo` object is created and passed to matching operations. As the matching progresses:

1. The system records which parts of the subject term match the pattern
2. The `ExtensionInfo` tracks this information in a theory-specific way
3. If the match is successful, the `ExtensionInfo` can be used to:
   - Determine if the entire subject was matched
   - Extract the matched portion of the subject
   - Extract the unmatched portion (in some implementations)

### Theory-Specific Implementations

Different equational theories in Maude implement their own subclasses of `ExtensionInfo`:

#### ACU_ExtensionInfo (Associative-Commutative With Identity)

This implementation:

- Tracks unmatched portions either as a DAG node or as a vector of multiplicities
- Provides methods to build both matched and unmatched portions
- Supports an upper bound on the amount of stuff that can go into the extension
- Handles special cases for tree-form DAG nodes

```cpp
// Example usage from ACU_ExtensionInfo
DagNode* buildMatchedPortion() const;
DagNode* buildUnmatchedPortion() const;
void setUnmatched(int argIndex, int multiplicity);
```

#### AU_ExtensionInfo (Associative With Identity)

This implementation:

- Tracks which contiguous segment of arguments was matched using indices
- Records whether an extra identity element was used in the match
- Ensures that at least two subterms are matched

```cpp
// Example usage from AU_ExtensionInfo
void setFirstMatched(int firstMatched);
void setLastMatched(int lastMatched);
void setExtraIdentity(bool flag);
bool bigEnough() const;  // checks if at least 2 subterms were matched
```

### In Matching Algorithms

The `ExtensionInfo` is typically used in this workflow:

1. A matching algorithm creates an appropriate `ExtensionInfo` object for the subject term
2. As matching proceeds, the algorithm updates the `ExtensionInfo` to reflect what was matched
3. The `setValidAfterMatch` method indicates whether the extension info is valid immediately after matching or only after solving associated constraints
4. If backtracking is needed, the current state is saved using `makeClone()`
5. After a successful match, `buildMatchedPortion()` can be called to construct the matched part

### Example from ACU_ExtensionInfo

Here's how `ACU_ExtensionInfo` builds the matched portion of a term:

```cpp
DagNode* ACU_ExtensionInfo::buildMatchedPortion() const
{
  if (matchedWhole())
    return subject;  // If everything matched, just return the original subject

  // Create a new node to hold the matched portion
  ACU_DagNode* s = getACU_DagNode(subject);
  ACU_DagNode* n = new ACU_DagNode(s->symbol(), s->argArray.length(), ACU_DagNode::ASSIGNMENT);
  
  // Fill in the matched portion based on what's recorded in the extension info
  // (implementation details omitted for brevity)
  
  return n;
}
```

## Conclusion

`ExtensionInfo` is a component in Maude's pattern matching infrastructure that enables:

1. Tracking which parts of a term were matched during pattern matching
2. Supporting theory-specific matching behaviors for different equational theories
3. Building representations of matched and unmatched portions of terms
4. Facilitating backtracking during complex matching operations

The abstract base class defines the common interface, while theory-specific implementations like `ACU_ExtensionInfo` and `AU_ExtensionInfo` provide the specialized behavior needed for their respective equational theories.
