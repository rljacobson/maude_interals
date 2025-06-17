# Reading and Writing from Inlined Bytes
Suppose you have a struct in which you want to store various types of data inlined. 
```rust
struct MyDataContainer {
    inline: [NUMBER_OF_BYTES]
}
```
Suppose that the data you are storing in this field has a specific known `Copy` type and lives at a known offset in this field.

**Q: How do we read and write typed data to this field?**
A: Use `read_unaligned` & `write_unaligned` to directly read the types. 

## Using `read_unaligned` & `write_unaligned` to directly read the types
This method first interprets the data to be read/written as the data's true type.

Reading:
```rust
let ptr = self.inline.as_ptr()
    .add(VARIABLE_INDEX_OFFSET) as *const VariableIndex;
let index = std::ptr::read_unaligned(ptr);
```

and writing:

```rust
let ptr = self.inline.as_mut_ptr()
    .add(VARIABLE_INDEX_OFFSET) as *mut VariableIndex;
std::ptr::write_unaligned(ptr, index);
```

### Pros
- **Zero extra copies:** you load or store the 4-byte (or N-byte) value in one go.
- **Minimal overhead:** generates a single unaligned load/store instruction on most architectures.
- **Type-safe width:** read_unaligned knows exactly how many bytes to grab from memory.
### Cons
- **More pointer fiddling:** you have to compute the raw pointer and do an unsafe read/write.
- **Repetition:** for each field you write, you have to duplicate that pattern (ptr math + read_unaligned).

## Copy bytes via slice

This method reads and writes data by interpreting it as bytes first.

Write data to the field:
```rust
fn as_bytes<T>(v: &T) -> &[u8] {  
  unsafe {  
    std::slice::from_raw_parts(value as *const T as *const u8, std::mem::size_of::<T>())  
  }  
}
// And then write the bytes
node.inline[..size_of::<T>()]
    .copy_from_slice(as_bytes(&value));
```

and then reading back:

```rust
let bytes = &self.inline[offset..offset + size_of::<T>()];
let val: T = unsafe { std::ptr::read_unaligned(bytes.as_ptr() as *const T) };
```

### **Pros**

- **Reuses copy_from_slice:** you don’t hand-write pointer arithmetic every time—just take a byte slice of your field and copy into it.
- **Clear intent:** “copy these raw bytes in/out” reads more like “serialization.”
- **Generic over T:** your as_bytes helpers work for _any_ sized type, so you standardize how you copy raw representations everywhere.
### **Cons**

- **Extra copy pass:** copy_from_slice will loop over each byte (or call a small memcpy), rather than a single unaligned word instruction. On hot code paths this can be measurably slower for multi-word types.
- **Bounds-checks on the slice:** the compiler will emit range checks for `[..size_of::<T>()]` even though you know the inline buffer is large enough.
- **One more unsafe on read:** you still end up doing a raw read_unaligned for reconstruction.
    
