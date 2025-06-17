# Frame Allocation in Maude's Stack Machine

## Memory Allocation Strategy

Frames in Maude's stack machine are allocated from a large pre-allocated block of memory, rather than using dynamic allocation for each frame. This approach offers performance benefits by avoiding the overhead of frequent memory allocation/deallocation.
### 1. Initial Memory Allocation

In the `StackMachine` constructor in `stackMachine.cc`:

```cpp
StackMachine::StackMachine()
{
  memoryBase = static_cast<char*>(operator new[](STACK_SIZE));
  topFrame = reinterpret_cast<Frame*>(memoryBase);
  // ...
}
```

A large block of memory (`STACK_SIZE = 1,000,000,000 bytes` or approximately 1GB) is allocated at initialization. The `topFrame` pointer is set to the beginning of this memory block.

### 2. Frame Allocation Process

New frames are allocated by advancing the `topFrame` pointer within this pre-allocated memory block. The `pushFrame` method in `stackMachine.hh` handles this:

```cpp
inline Frame*
StackMachine::pushFrame(FrameLift frameLift)
{
  topFrame = reinterpret_cast<Frame*>(reinterpret_cast<char*>(topFrame) + frameLift);
  return topFrame;
}
```

This method:

1. Converts the current `topFrame` pointer to a `char*` pointer
2. Advances this pointer by the `frameLift` amount
3. Converts the result back to a `Frame*` pointer
4. Updates `topFrame` and returns it

### 3. Frame Size Calculation

The size of each frame is determined by the `makeFrameLift` function:

```cpp
inline StackMachine::FrameLift
StackMachine::makeFrameLift(int nrSlotsToPreserve)
{
  return sizeof(Frame) + sizeof(DagNode*) * nrSlotsToPreserve;
}
```

This calculates the total size needed for a frame with:

- The fixed size of the `Frame` structure (containing the header fields)
- Additional space for the specified number of slots (each slot is a `DagNode*` pointer)

### 4. Fast Frame Allocation

For performance-critical paths, there's also a `fastPushFrame` static method:

```cpp
inline Frame*
StackMachine::fastPushFrame(Frame* oldFrame, FrameLift frameLift)
{
  return reinterpret_cast<Frame*>(reinterpret_cast<char*>(oldFrame) + frameLift);
}
```

This creates a new frame from an old frame without updating the stack machine's internal state.

## Frame Deallocation

Frames are not individually deallocated. Instead, when a frame is no longer needed, the `topFrame` pointer is simply moved back to an ancestor frame:

```cpp
inline void
StackMachine::popFrame(Frame* frame)
{
  Assert(frame == topFrame, "frame clash");
  topFrame = frame->getAncestorWithValidNextInstruction();
}
```

This effectively "deallocates" all frames between the current frame and the ancestor frame by making them inaccessible.

## Memory Management Considerations

This allocation strategy has important implications:

1. **No Constructors/Destructors**: As noted in the `Frame` class comment, frames can't have constructors, destructors, or virtual functions because of this allocation strategy.
2. **Garbage Collection Integration**: The stack machine integrates with Maude's garbage collection system through the `markReachableNodes` method, which marks all active DAG nodes referenced by slots in active frames.
3. **Memory Efficiency**: This approach is memory-efficient for the typical execution pattern of a stack machine, where frames are allocated and deallocated in a last-in-first-out manner.

## Conclusion

Frame allocation in Maude's stack machine uses a pre-allocated memory block with a simple pointer-advancing mechanism.

# Frame Lifting in Maude's Stack Machine

## What is Frame Lifting?

"Frame lifting" in Maude's stack machine refers to the process of calculating and allocating the appropriate amount of memory for a new frame on the stack. The term "lift" here refers to the offset or distance by which the stack pointer needs to be advanced to accommodate a new frame.

## The `makeFrameLift` Function

The `makeFrameLift` function calculates the size (in bytes) needed for a frame with a specific number of slots:

```cpp
inline StackMachine::FrameLift
StackMachine::makeFrameLift(int nrSlotsToPreserve)
{
  return sizeof(Frame) + sizeof(DagNode*) * nrSlotsToPreserve;
}
```

This function:

1. Takes a parameter `nrSlotsToPreserve` which specifies how many slots the new frame needs
2. Calculates the total memory needed by adding:
   - The fixed size of the `Frame` structure (containing header fields like `nextInstruction`, `returnAddress`, etc.)
   - Additional space for the specified number of slots (each slot is a `DagNode*` pointer)
3. Returns this total size as a `FrameLift` value (which is a `typedef` for `size_t`)

## How Frame Lifting is Used

The calculated `FrameLift` value is then used in the frame allocation process:

```cpp
inline Frame*
StackMachine::pushFrame(FrameLift frameLift)
{
  topFrame = reinterpret_cast<Frame*>(reinterpret_cast<char*>(topFrame) + frameLift);
  return topFrame;
}
```

This method:

1. Converts the current `topFrame` pointer to a `char*` pointer (to allow byte-level arithmetic)
2. Advances this pointer by the `frameLift` amount (the calculated size)
3. Converts the result back to a `Frame*` pointer
4. Updates `topFrame` to point to this new location and returns it

There's also a static version called `fastPushFrame` that creates a new frame without updating the stack machine's internal state:

```cpp
inline Frame*
StackMachine::fastPushFrame(Frame* oldFrame, FrameLift frameLift)
{
  return reinterpret_cast<Frame*>(reinterpret_cast<char*>(oldFrame) + frameLift);
}
```

## Example of Frame Lifting in Action

Here's how frame lifting is used in practice, from the `execute` method in `stackMachine.cc`:

```cpp
Frame* initFrame = pushFrame(makeFrameLift(0));
```

This line:

1. Calls `makeFrameLift(0)` to calculate the size needed for a frame with 0 slots
2. Passes this size to `pushFrame` to allocate a new frame
3. Stores the pointer to this new frame in `initFrame`

## Why This Approach?

This approach to frame allocation offers several advantages:

1. **Efficiency**: It avoids the overhead of dynamic memory allocation for each frame
2. **Control**: It gives precise control over memory layout and size
3. **Performance**: It allows for fast frame allocation and deallocation

## Conclusion

Frame lifting in Maude's stack machine is a memory management technique that calculates and allocates the appropriate amount of memory for frames based on their slot requirements. The `makeFrameLift` function is central to this process, determining exactly how much memory is needed for a frame with a specific number of slots.
