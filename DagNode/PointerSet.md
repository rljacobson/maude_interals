# `PointerSet` Implementation Details

## Core Data Structures

```cpp
class PointerSet {
private:
    struct Pair {
        void* pointer;
        unsigned int rawHashValue;
    };
    
    Vector<Pair> pointerTable;    // Stores pointer-hash pairs
    Vector<int> hashTable;        // Hash table for lookups
};
```

The class uses two main data structures:
- `pointerTable`: Stores actual pointer-hash pairs
- `hashTable`: Maps hash values to indices in pointerTable

## Hash Functions

1. Primary Hash:
```cpp
unsigned int PointerSet::hash(void* pointer) const {
    return reinterpret_cast<unsigned long>(pointer) >> NR_PTR_LOSE_BITS;
}
```

2. Local Hash Functions:
```cpp
inline int PointerSet::localHash(unsigned int rawHashValue) {
    return rawHashValue ^ (rawHashValue >> 2);
}

inline int PointerSet::localHash2(unsigned int rawHashValue) {
    return (rawHashValue ^ (rawHashValue >> (WORD_SIZE / 2))) | 1;
}
```

The second hash function always returns odd values to ensure proper collision resolution.

## Core Operations

### 1. Insertion

```cpp
int PointerSet::insert(void* p, unsigned int rawHashValue) {
    if (pointerTable.length() == 0)
        resize(STARTING_HASH_TABLE_SIZE);
    
    int i = findEntry(p, rawHashValue);
    int j = hashTable[i];
    
    if (j == UNUSED) {
        j = pointerTable.length();
        pointerTable.expandBy(1);
        Pair& pair = pointerTable[j];
        pair.pointer = p;
        pair.rawHashValue = rawHashValue;
        
        // Resize if necessary
        if (2 * (j + 1) > hashTable.length())
            resize(2 * (j + 1));
        else
            hashTable[i] = j;
    }
    return j;
}
```

### 2. Lookup

```cpp
int PointerSet::findEntry(void* p, unsigned int rawHashValue) const {
    int mask = hashTable.length() - 1;
    int i = localHash(rawHashValue) & mask;
    int j = hashTable[i];
    
    // Handle collisions
    if (j != UNUSED && !isEqual(pointerTable[j], p, rawHashValue)) {
        int step = localHash2(rawHashValue);
        do {
            i = (i + step) & mask;
            j = hashTable[i];
        } while (j != UNUSED && !isEqual(pointerTable[j], p, rawHashValue));
    }
    return i;
}
```

### 3. Resizing and Rehashing

```cpp
void PointerSet::resize(int minSize) {
    int n = hashTable.length();
    n = (n == 0) ? STARTING_HASH_TABLE_SIZE : 2 * n;
    while (n < minSize)
        n *= 2;
    hashTable.expandTo(n);
    rehash();
}

void PointerSet::rehash() {
    // Clear hash table
    for (int i = 0; i < hashTable.length(); i++)
        hashTable[i] = UNUSED;
    
    // Rehash all entries
    int mask = hashTable.length() - 1;
    for (int i = pointerTable.length() - 1; i >= 0; i--) {
        unsigned int rawHashValue = pointerTable[i].rawHashValue;
        int j = localHash(rawHashValue) & mask;
        
        // Handle collisions
        if (hashTable[j] != UNUSED) {
            int step = localHash2(rawHashValue);
            do
                j = (j + step) & mask;
            while (hashTable[j] != UNUSED);
        }
        hashTable[j] = i;
    }
}
```

## Key Features

1. **Double Hashing**: Uses two hash functions to resolve collisions:
   - Primary hash for initial slot
   - Secondary hash for collision resolution step size

2. **Dynamic Resizing**:
   - Starts with `STARTING_HASH_TABLE_SIZE` (8)
   - Doubles size when needed
   - Maintains load factor by resizing

3. **Collision Resolution**:
   - Uses open addressing with double hashing
   - Secondary hash always returns odd values to ensure complete table coverage

4. **Memory Efficiency**:
   - Stores hash values with pointers to avoid recomputation
   - Uses power-of-two sizes for efficient modulo operations

5. **Set Operations**:
   - Supports union (insert)
   - Intersection (intersect)
   - Difference (subtract)
   - Subset testing (contains)

This implementation provides efficient O(1) average case operations while handling collisions and growth gracefully.

# `PointerSet` Public API

## Basic Operations

### Construction and Destruction
```cpp
PointerSet();                              // Default constructor
PointerSet(const PointerSet& original);    // Copy constructor
virtual ~PointerSet();                     // Destructor
```

### Core Pointer Operations
```cpp
// Basic pointer operations (automatically computes hash)
int insert(void* p);              // Add a pointer to the set
void subtract(void* p);           // Remove a pointer from the set
bool contains(void* p) const;     // Check if pointer exists in set
int pointer2Index(void* p) const; // Convert pointer to index

// Operations with pre-computed hash values
int insert(void* p, unsigned int rawHashValue);
void subtract(void* p, unsigned int rawHashValue);
bool contains(void* p, unsigned int rawHashValue) const;
int pointer2Index(void* p, unsigned int rawHashValue) const;
```

### Set Operations
```cpp
void makeEmpty();                          // Clear all elements
void insert(const PointerSet& other);      // Union with another set
void subtract(const PointerSet& other);    // Remove elements in other set
void intersect(const PointerSet& other);   // Keep only elements in both sets
bool contains(const PointerSet& other);    // Check if other is subset
bool disjoint(const PointerSet& other);    // Check if sets have no common elements
```

### Utility Methods
```cpp
bool empty() const;                        // Check if set is empty
int cardinality() const;                   // Get number of elements
void* index2Pointer(int i) const;          // Convert index to pointer
void swap(PointerSet& other);              // Swap contents with another set

// Comparison operators
bool operator==(const PointerSet& other) const;
bool operator!=(const PointerSet& other) const;
PointerSet& operator=(const PointerSet& original);
```

## Usage Examples

1. Basic Usage:
```cpp
PointerSet set;
void* ptr = someObject;

// Add pointer
int index = set.insert(ptr);

// Check existence
if (set.contains(ptr)) {
    // Handle found case
}

// Remove pointer
set.subtract(ptr);
```

2. Using Pre-computed Hashes:
```cpp
void* ptr = someObject;
unsigned int hash = computeHash(ptr);

// Operations with pre-computed hash
set.insert(ptr, hash);
set.contains(ptr, hash);
set.subtract(ptr, hash);
```

3. Set Operations:
```cpp
PointerSet set1, set2;

// Union
set1.insert(set2);

// Intersection
set1.intersect(set2);

// Difference
set1.subtract(set2);

// Subset check
if (set1.contains(set2)) {
    // set2 is subset of set1
}
```

4. Iteration:
```cpp
PointerSet set;
// … add elements …

for (int i = 0; i < set.cardinality(); i++) {
    void* ptr = set.index2Pointer(i);
    // Process ptr
}
```

5. Custom Hash and Equality:
```cpp
class MyPointerSet : public PointerSet {
protected:
    // Override hash function
    virtual unsigned int hash(void* pointer) const {
        return customHash(pointer);
    }
    
    // Override equality comparison
    virtual bool isEqual(void* pointer1, void* pointer2) const {
        return customEqual(pointer1, pointer2);
    }
};
```

## Key Features

1. **Efficiency**:
   - O(1) average case for insertions and lookups
   - Automatic resizing for performance
   - Hash caching for expensive hash computations

2. **Flexibility**:
   - Supports custom hash and equality functions
   - Provides both pointer and index-based access
   - Comprehensive set operations

3. **Safety**:
   - Copy construction and assignment
   - Clean memory management
   - Thread-safe const operations

# Purpose of Index/Pointer Conversion Methods

## `pointer2Index`
```cpp
int pointer2Index(void* p) const;                    // Computes hash internally
int pointer2Index(void* p, unsigned int hash) const; // Uses provided hash
```

Purpose:
- Converts a pointer to its index position in the `pointerTable`
- Returns `NONE` if pointer isn't in the set
- Useful for:
  - Checking if/where a pointer exists in the set
  - Getting a stable numeric identifier for a pointer
  - Enabling array-like indexing into related data structures

## `index2Pointer`
```cpp
void* index2Pointer(int i) const;
```

Purpose:
- Retrieves the pointer stored at a given index in the `pointerTable`
- Provides direct access to stored pointers
- Useful for:
  - Iterating through all pointers in the set
  - Retrieving pointers when you have their indices

## Usage Pattern

These methods are often used together to create efficient mappings and enable iteration:

```cpp
PointerSet set;

// Store some pointers
void* ptr1 = someObject1;
void* ptr2 = someObject2;
set.insert(ptr1);
set.insert(ptr2);

// Get index for a pointer
int index = set.pointer2Index(ptr1);
if (index != NONE) {
    // Found the pointer
    
    // Can retrieve the same pointer using the index
    void* samePtr = set.index2Pointer(index);
    assert(samePtr == ptr1);
}

// Iterate through all pointers
for (int i = 0; i < set.cardinality(); i++) {
    void* ptr = set.index2Pointer(i);
    // Process ptr
}
```

## Key Benefits

1. **Stable Identifiers**:
   - Indices remain valid until the pointer is removed
   - Can be used as keys in other data structures

2. **Efficient Iteration**:
   - Sequential access to all pointers
   - No need to deal with hash table internals

3. **Bidirectional Mapping**:
   - Can convert between pointers and indices in both directions
   - Maintains relationship between numeric indices and pointers

4. **Performance**:
   - `index2Pointer` is O(1) direct array access
   - `pointer2Index` is O(1) average case hash lookup

These methods work together to provide an interface for managing and accessing pointers in the set, whether you need to work with the pointers directly or through their numeric indices.

