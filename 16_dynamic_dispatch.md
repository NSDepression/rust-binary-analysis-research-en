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

When passing trait objects in assembly, typically:
- The `rdi` register contains the address of the trait object data
- The `rsi` register contains the vtable pointer

### VTable Structure

In debug build binaries, function calls are made using a VTable structure.
The VTable is a contiguous piece of storage in memory containing:

* 64-bit binaries

```c
vtable {
    0x00: Destructor (MetadataDropInPlace)
    0x08: Size of source structure (MetadataSize)
    0x10: Alignment of source structure (MetadataAlign)
    0x18+: Method pointers (function pointers to trait method implementations)
}
```

* 32-bit binaries

```c
vtable {
    0x00: Destructor (MetadataDropInPlace)
    0x04: Size of source structure (MetadataSize)
    0x08: Alignment of source structure (MetadataAlign)
    0x0C+: Method pointers (function pointers to trait method implementations)
}
```

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
- Watch for rdi/rsi register usage patterns indicating trait object method calls

## References

For more detailed information on Rust trait objects and vtables:
- [Understanding Rust's Trait Objects: Vtables, Dynamic Dispatch, and Memory Deallocation](https://eventhelix.com/rust/rust-to-assembly-tail-call-via-vtable-and-box-trait-free/)
- [Rust Deep Dive: Borked Vtables and Barking Cats](https://geo-ant.github.io/blog/2023/rust-dyn-trait-objects-fat-pointers/)
- [Exploring Dynamic Dispatch in Rust](https://alschwalm.com/blog/static/2017/03/07/exploring-dynamic-dispatch-in-rust/)
