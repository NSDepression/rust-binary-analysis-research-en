# Strings

We investigated how Rust's String type and &str type are handled at the assembly level.

## Investigation Results

### String Type (Heap-Allocated)

The String type is heap-allocated, growable, and not null-terminated. It consists of three components stored on the stack:

1. **Pointer** - Points to the memory on the heap that holds the string contents
2. **Capacity** - Maximum number of bytes that can be stored without reallocation
3. **Length** - Current number of bytes actually used

On 64-bit systems, each field is 8 bytes (usize), making the total stack structure 24 bytes.

**Key Characteristics:**
- String data itself is on the heap
- Metadata structure (ptr, capacity, length) is on the stack
- Growable and mutable
- Ownership and automatic deallocation through Drop trait

### &str Type (String Slice)

The &str type is a string slice that always points to valid UTF-8 sequence. The slice consists of:

1. **Pointer** - Points to the first character of the string
2. **Length** - Number of bytes in the string

This creates a "fat pointer" structure on the stack (16 bytes on 64-bit systems).

**Key Characteristics:**
- Actual string data is NOT on the stack
- Can point to various memory locations:
  - `.rdata` or `.data` segment (for string literals)
  - Heap memory (when slicing a String)
  - Memory-mapped files
  - Any valid UTF-8 byte sequence
- Immutable view into string data
- More efficient than String for read-only operations

### String Literals

String literals are known at compile time and stored in the read-only data section (`.rdata` or `.rodata`) of the executable. They:
- Do not consume stack or heap space at runtime
- Are embedded directly in the binary
- Provide the most efficient string access
- Are visible in the binary's read-only section (useful for reverse engineering)

### Comparison Summary

* **String type:** Heap-allocated, owning, mutable, growable
* **&str type:** View/slice, immutable, can reference various memory locations
* **String literals:** Compile-time, embedded in binary, read-only section

Additionally, even in 32-bit binaries and minimized binaries, no differences were observed in the structure or handling for managing strings, except for address size (4 bytes instead of 8 bytes for pointers and size values).

## Details

### Detailed Memory Layout

#### String Type Structure

According to Rust's official documentation, the String type is internally Vec<u8>.
As described in [Collections](17_collection.md), the Vec type is composed of a structure consisting of the maximum number of elements, an address to the buffer, and the current number of elements.

**Stack Structure (64-bit):**
```
Offset 0x00: Pointer to heap buffer (8 bytes)
Offset 0x08: Capacity in bytes (8 bytes)
Offset 0x10: Length in bytes (8 bytes)
Total: 24 bytes on stack
```

**Heap:** Contains the actual UTF-8 encoded string data

![strings](images/7-1.png)

#### &str Type Structure

A data structure consisting of a pointer to the string and the length is constructed on the stack. This is a "fat pointer" or "slice" type.

**Stack Structure (64-bit):**
```
Offset 0x00: Pointer to string data (8 bytes)
Offset 0x08: Length in bytes (8 bytes)
Total: 16 bytes on stack
```

**Data Location:** Varies based on source (see examples below)

![strings](images/7-2.png)

#### Memory Location Examples

```rust
// 1. String literal - data in .rdata section
let s1: &str = "hello";

// 2. Slice of String - data on heap
let s2: String = String::from("hello");
let s3: &str = &s2[..];

// 3. Static string - data in .data section
static S4: &str = "world";
```

Each `&str` has the same stack structure (pointer + length), but the actual string data resides in different memory regions.

### Various String Literals

* Raw string literal

In normal programs, `\n` should be converted to 0x0A (newline code), but when using `r""`, `\n` is represented as 0x5C (representing `\`) and 0x6E (representing `n`) and is embedded in the binary in an unescaped state.
Otherwise, there is no difference in handling compared to regular strings.

![strings](images/7-3.png)

* C string literal

There is no difference in handling compared to regular strings.
Embedded in the `.text` section.

* binary string literal

There is no difference in handling compared to regular strings.
Defined in the `.rdata` section.

## Reverse Engineering Implications

When analyzing Rust binaries:

1. **String Literals:** Look in `.rdata`/`.rodata` sections for embedded strings. These are directly visible and provide valuable context.

2. **String Type Recognition:** Look for the 24-byte (64-bit) or 12-byte (32-bit) stack structure pattern with heap pointer references.

3. **&str Type Recognition:** Look for the 16-byte (64-bit) or 8-byte (32-bit) fat pointer pattern.

4. **Heap Allocations:** String heap allocations use the standard allocator (`__rust_alloc`), which ultimately calls system allocation functions like `HeapAlloc()` on Windows.

5. **String Manipulation:** String operations that modify content require a String type (heap-allocated), while read-only operations can use &str slices efficiently.

### Architecture-Specific Details

#### x86-64 (x64)
When passing `&str` as a function parameter:
- The **pointer** (address of string data) is typically passed in the `rdi` register
- The **length** (usize) is typically passed in the `rsi` register

This follows the System V AMD64 ABI calling convention commonly used on Unix-like systems.

#### ARM64 (AArch64)
When passing `&str` as a function parameter:
- The **pointer** (address of string data) is passed in the `x0` register
- The **length** (usize) is passed in the `x1` register

This follows the ARM64 (AArch64) calling convention where the first eight parameters use registers x0-x7. Since `&str` is a fat pointer consisting of two pointer-sized values (16 bytes total on 64-bit platforms), it occupies two consecutive registers.

**Key Points:**
- Both architectures use the same memory layout (pointer + length)
- The difference is only in which registers are used for parameter passing
- Return values follow similar patterns (x86-64 uses `rax`/`rdx`, ARM64 uses `x0`/`x1`)
- On both platforms, the fat pointer structure is 16 bytes (two 8-byte values)

**⚠️ Important:** Rust's calling convention for Rust-to-Rust function calls is not stable and may change between compiler versions. The register usage patterns described above represent common observations but should not be considered guaranteed.

## References

For more detailed information:
- [Rust: How are Strings stored in memory?](https://medium.com/rustaceans/rust-how-are-strings-stored-in-memory-01d29ec79844)
- [Memory layout of a Rust program](https://shbhmrzd.github.io/2024/08/31/memory_layout_of_a_rust_program.html)
- [Exploring Rust Fat Pointers](https://iandouglasscott.com/2018/05/28/exploring-rust-fat-pointers/)
- [AArch64 Procedure Call Standard](https://medium.com/@tunacici7/aarch64-procedure-call-standard-aapcs64-abi-calling-conventions-machine-registers-a2c762540278) - ARM64 calling conventions
