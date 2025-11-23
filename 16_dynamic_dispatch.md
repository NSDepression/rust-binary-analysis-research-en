# Dynamic Dispatch References

We investigated assembly characteristics of dynamic dispatch references using trait objects.

## Investigation Results

### Fat Pointers and Trait Objects

Rust uses "fat pointers" for trait objects, which consist of two components:
1. A pointer to the actual data (of some unknown type T)
2. A pointer to the associated vtable instance

Unlike C++ where the vtable pointer is embedded in the struct itself, in Rust the struct never contains a vtable pointer. The fat pointer structure keeps the data and vtable references separate.

For example, a trait object like `&dyn Foo` consists of:
- A 'data' pointer addressing the data
- A 'vtable' pointer pointing to the vtable corresponding to the implementation of Foo for T

### Architecture-Specific Register Usage

When passing trait objects (fat pointers) as function parameters, different architectures use different registers:

**x86-64 (x64):**
- The `rdi` register contains the address of the trait object data
- The `rsi` register contains the vtable pointer

**ARM64 (AArch64):**
- The `x0` register contains the address of the trait object data
- The `x1` register contains the vtable pointer

Both architectures follow their standard calling conventions (System V AMD64 ABI for x86-64, and AAPCS64 for ARM64), where composite types up to 16 bytes are passed in consecutive general-purpose registers.

**⚠️ Important:** Rust's calling convention for Rust-to-Rust function calls is not stable and may change between compiler versions. The register usage patterns described here represent common observations but should not be considered guaranteed.

### VTable Structure

In debug build binaries, function calls are made using a VTable structure.
The VTable is a contiguous piece of storage in memory containing:

**64-bit binaries (x86-64 and ARM64):**

```c
vtable {
    0x00: Destructor (MetadataDropInPlace)
    0x08: Size of source structure (MetadataSize)
    0x10: Alignment of source structure (MetadataAlign)
    0x18+: Method pointers (function pointers to trait method implementations)
}
```

**32-bit binaries (x86 and ARM32):**

```c
vtable {
    0x00: Destructor (MetadataDropInPlace)
    0x04: Size of source structure (MetadataSize)
    0x08: Alignment of source structure (MetadataAlign)
    0x0C+: Method pointers (function pointers to trait method implementations)
}
```

**Note:** The vtable structure layout is identical across architectures with the same pointer size. The difference between x86-64 and ARM64 is not in the vtable structure itself, but in the instruction sets and calling conventions used to access it.

### Important Characteristics

**Single VTable Per Type:** Only one vtable per concrete type is created. All trait object instances of the same concrete type point to the same vtable. This is crucial for understanding binary layout and optimizing analysis.

**VTable Generation:** The compiler generates vtables by:
1. First emitting the header (MetadataDropInPlace, MetadataSize, MetadataAlign)
2. Traversing the supertrait tree in post-order
3. Emitting associated functions as Method or Vacant entries
4. Adding TraitVPtr entries for traits not in the prefix set

**Optimization Impact:** In release builds and minimized binaries, VTable may be removed entirely and converted to general conditional branching if the compiler can determine all concrete types at compile time.

## Details

### Debug Build Analysis

In debug build binaries, function calls become indirect calls through the VTable structure due to dynamic dispatch references.
The following is the processing that places the address named VTable by IDA Pro onto the stack.

![iterator](images/16-1.png)

Subsequently, processing to obtain values from VTable and indirect function calls can be confirmed. Each receives the address of the structure as the first argument.

![iterator](images/16-2.png)

### Method Dispatch Process

When a trait method is called on a trait object:
1. The fat pointer provides both data and vtable addresses
2. The vtable is accessed at specific byte offsets to retrieve function pointers
3. The appropriate method implementation is called indirectly
4. The data pointer is passed as the first argument (self parameter)

This indirect calling mechanism enables runtime polymorphism while maintaining Rust's zero-cost abstraction principles for static dispatch scenarios.

### Reverse Engineering Tips

When analyzing Rust binaries with trait objects:
- Look for fat pointer patterns (two consecutive pointer-sized values)
- Identify vtable structures by their header pattern (destructor, size, align)
- Method offsets in vtables are predictable based on trait definition order
- Watch for architecture-specific register usage patterns:
  - **x86-64:** `rdi`/`rsi` register pairs for trait object method calls
  - **ARM64:** `x0`/`x1` register pairs for trait object method calls
- Indirect function calls through vtable offsets (e.g., `call qword ptr [rsi+0x18]` on x86-64, `ldr x8, [x1, #0x18]; blr x8` on ARM64)
- vtable access patterns often involve fixed offsets (0x18, 0x20, 0x28, etc. on 64-bit platforms)

**ARM64-Specific Patterns:**
- Look for `ldr` (load register) instructions to load vtable function pointers
- `blr` (branch with link to register) instructions for indirect calls
- Pairs of `x0`-`x7` registers used for passing parameters following AAPCS64
- `ldp` (load pair) instructions may be used to load both components of fat pointers efficiently

## References

For more detailed information on Rust trait objects and vtables:
- [Understanding Rust's Trait Objects: Vtables, Dynamic Dispatch, and Memory Deallocation](https://eventhelix.com/rust/rust-to-assembly-tail-call-via-vtable-and-box-trait-free/)
- [Rust Deep Dive: Borked Vtables and Barking Cats](https://geo-ant.github.io/blog/2023/rust-dyn-trait-objects-fat-pointers/)
- [Exploring Dynamic Dispatch in Rust](https://alschwalm.com/blog/static/2017/03/07/exploring-dynamic-dispatch-in-rust/)
- [Understanding Rust's Trait Objects](https://medium.com/software-design/understanding-rusts-trait-objects-224b1d8daede)
- [AArch64 Procedure Call Standard](https://medium.com/@tunacici7/aarch64-procedure-call-standard-aapcs64-abi-calling-conventions-machine-registers-a2c762540278) - ARM64 calling conventions
