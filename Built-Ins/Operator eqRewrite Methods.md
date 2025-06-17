# Understanding the `eqRewrite` Method in Maude

## What is the `eqRewrite` Method?

The `eqRewrite` method is a component in Maude's symbol implementation that handles built-in operations and equality testing. It's a virtual method defined in various symbol classes that performs specialized rewriting for built-in data types and operations.

## Purpose and Function

The primary purposes of the `eqRewrite` method are:

1. **Built-in Operation Evaluation**: It evaluates built-in operations like arithmetic, string manipulation, or equality testing directly in C++ rather than through equational rewriting.
2. **Equality Testing**: For equality symbols specifically, it determines whether two terms are equal and returns the appropriate boolean result.
3. **Performance Optimization**: By implementing operations directly in C++, it provides much better performance than would be possible through pure equational rewriting.

## How It Works

Looking at the provided code, we can see two key implementations:

### 1. In `EqualitySymbol` (from `equalitySymbol.cc`)

```cpp
bool EqualitySymbol::eqRewrite(DagNode* subject, RewritingContext& context)
{
  Assert(this == subject->symbol(), "bad symbol");
  FreeDagNode* f = static_cast<FreeDagNode*>(subject);
  DagNode* l = f->getArgument(0);
  DagNode* r = f->getArgument(1);
  
  // Reduce arguments according to strategy
  if (standardStrategy()) {
    l->reduce(context);
    r->reduce(context);
  } else {
    // Apply user-defined strategy
    // ...
  }
  
  // Return true/false term based on equality test
  return context.builtInReplace(subject, l->equal(r) ?
                equalTerm.getDag() : notEqualTerm.getDag());
}
```

This implementation:

1. Extracts the two arguments from the equality operation
2. Reduces them according to the appropriate strategy
3. Tests if they are equal using the `equal` method
4. Replaces the original term with either `true` or `false`

### 2. In `FloatOpSymbol` (from `floatOpSymbol.cc`)

The implementation is more complex, handling various floating-point operations:

```cpp
bool FloatOpSymbol::eqRewrite(DagNode* subject, RewritingContext& context)
{
  // Extract arguments and reduce them
  // ...
  
  // Determine which operation to perform based on op code
  double r;
  switch (op) {
    case '+':
      r = a1 + a2;
      break;
    case '-':
      r = a1 - a2;
      break;
    // ... many other operations
  }
  
  // Return the result as a float or boolean
  return floatSymbol->rewriteToFloat(subject, context, r);
}
```

This implementation:

1. Extracts and reduces the arguments
2. Determines which operation to perform based on the operation code
3. Performs the operation directly in C++
4. Returns the result as the appropriate data type

## Relationship to Maude's Prelude

The `eqRewrite` method directly implements the operations defined in Maude's prelude. For example, the float addition operation defined in the prelude:

```
op _+_ : Float Float -> Float
      [prec 33 gather (E e)
       special (id-hook FloatOpSymbol (+)
                op-hook floatSymbol (<Floats> : ~> Float))] .
```

This connects to the C++ implementation through:

1. The `id-hook FloatOpSymbol (+)` which links to the `FloatOpSymbol` class with operation code `'+'`
2. The `op-hook floatSymbol` which links to the `FloatSymbol` class

## Conclusion

The `eqRewrite` method is a bridge between Maude's high-level equational specification language and efficient low-level C++ implementations. It allows Maude to handle built-in operations with native performance while maintaining the declarative nature of the language. This hybrid approach gives Maude both the flexibility of a rewriting logic language and the performance of compiled code for common operations.
