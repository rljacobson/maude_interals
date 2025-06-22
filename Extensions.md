The `ExtensionInfo` class is an abstract interface that handles partial matching and reconstruction of terms in Maude's equational theories, particularly for associative theories where only part of a term may match a pattern (`interface.hh:47`).

### What is an Extension?

An "extension" in Maude refers to the unmatched portion of a term when pattern matching occurs. For example, in an associative theory like `f(a, b, c)`, if a pattern matches only `f(a, b)`, then `c` becomes the "extension" - the part that extends beyond what the pattern matched.

### Core ExtensionInfo Functionality

The `ExtensionInfo` class provides several key methods for handling extensions:

**Matching Information**:

- `matchedWhole()`: Returns whether the entire term was matched (no extension exists)
- `buildMatchedPortion()`: Constructs the part of the term that was actually matched

**Reconstruction Operations**:

- `makeClone()`: Creates a copy of the extension info for independent manipulation

These methods are used throughout the matching and rewriting process to handle partial matches correctly.

### Usage in Matching and Rewriting

`ExtensionInfo` is extensively used in equation and rule application. In `EquationTable::applyReplace()`, when applying equations with extensions, the system checks if the whole term was matched . If not, it uses `partialReplace()` to reconstruct the term with the matched portion replaced while preserving the extension.

Similarly, in rule application, `RuleTable::applyRules()` uses the same pattern (`ruleTable.cc:109-115`), checking `extensionInfo->matchedWhole()` to determine whether to do a complete replacement or a partial reconstruction.

### Theory-Specific Implementations

Different algebraic theories implement `ExtensionInfo` differently:

**S Theory (Successor)**: `S_DagNode::makeExtensionInfo()` creates `S_ExtensionInfo` objects (`S_DagNode.cc:175-179`). The S theory uses extensions to handle partial matches in successor structures, where `partialConstruct()` creates new S nodes with unmatched portions (`S_DagNode.cc:167-173`).

**AU Theory**: AU theories also support extension info creation through `makeExtensionInfo()` (`AU_DagNode.hh:59`) and provide `partialReplace()` and `partialConstruct()` methods for handling partial matches in associative-commutative structures.

### Integration with Matching Process

The matching process integrates `ExtensionInfo` at multiple levels. In `DagNode::matchVariable()`, when extension info is present, it delegates to `matchVariableWithExtension()` (`dagNode.cc:114-115`).

For variant equation checking, `DagNode::reducibleByVariantEquation()` creates extension info and passes it to equation matching (`dagNode.cc:348-357`), allowing the system to determine if partial matches with variant equations are possible.

### Position-Based Extensions

The `PositionState` class also works with extension info for position-based rewriting (`positionState.hh:113-123`). It lazily creates extension info when needed and uses it for rebuilding terms after positional replacements.

## Notes

`ExtensionInfo` is crucial for Maude's handling of associative and other complex algebraic theories where pattern matching may only cover part of a term. The abstract interface allows different theories to implement their own extension semantics while providing a uniform interface for the core matching and rewriting algorithms. Most basic theories don't need extensions, which is why `DagNode::makeExtensionInfo()` returns null by default (`dagNode.cc:136-140`).

## Core Extension Classes

`ExtensionMatchSubproblem` is a crucial class that handles matching when extensions are involved (`extensionMatchSubproblem.hh:33-50`). This class takes an `ExtensionInfo` object and uses it to match patterns against the matched portion of a subject term (`extensionMatchSubproblem.cc:63-78`).

## Theory-Specific Extension Implementations

`S_ExtensionInfo` is the successor theory's implementation of extensions (`S_Theory.hh:37`). The S theory creates these through `S_DagNode::makeExtensionInfo()` and uses them in partial construction operations .

`ACU_ExtensionInfo` handles extensions for associative-commutative-unit theories. The ACU base class creates these extension info objects (`ACU_BaseDagNode.cc:53-57`). The ACU matching algorithms extensively use extension info to track which parts of associative terms were matched ACU_GreedyMatcher.cc:306-325 .

`AU_ExtensionInfo` is used in the AU (associative-unit) theory for similar purposes. AU matching algorithms set extension boundaries and validity flags (`AU_GreedyMatcher.cc:38-44`) and handle extension info during full matching (`AU_FullMatcher.cc:40-42`).

## Integration Classes

`PositionState` manages extension info for position-based operations, lazily creating it when needed for DAG node operations (`positionState.cc:53-61`).

The `DagNode` base class provides default implementations that return null for theories that don't support extensions (`dagNode.cc:136-168`), while theory-specific DAG nodes like `AU_DagNode` declare extension-related methods (`AU_DagNode.hh:52-59`).

## Usage Pattern

The extension system works by having theory-specific symbols create appropriate extension info objects, which are then passed to matching algorithms. When partial matches occur, the extension info tracks what wasn't matched, allowing reconstruction methods like `partialConstruct()` and `partialReplace()` to properly handle the unmatched portions while applying transformations to the matched parts.

This architecture allows Maude to handle complex associative theories where patterns may only match part of a term, while preserving the unmatched portions for proper term reconstruction.