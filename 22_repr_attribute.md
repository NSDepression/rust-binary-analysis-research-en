# repr Attribute

The `repr` attribute can be used to control the memory layout of structures and other types. In this investigation, we examined how each available option affects memory layout.

## Investigation Results

The memory layout of structures is as follows:

* **`#[repr(Rust)]`**
  - Regardless of the definition order in the source code, fields are placed in memory in descending order of size (with alignment).

* **`#[repr(C)]`**
  - Memory layout is compatible with C/C++, and fields are placed in memory in the order defined in the source code (with alignment).

* **`#[repr(packed(1))]`**
  - An option to specify memory alignment, and fields are placed in memory in the order defined in the source code (without alignment).

## Details

### \#\[repr(Rust)\]

`#[repr(Rust)]` is the default when no attribute is explicitly specified, but in the case of the sample code, the memory layout of the structure is as follows:
Due to alignment, 2 bytes of memory are allocated even to variable y which is of type i8, totaling 32 bytes.
Also, regardless of the definition order in the source code, variables with larger sizes are placed in memory first.
Variables with the same number of bytes are placed in memory in the order defined in the source code.

![repr_attribute](images/22-1.png)

### \#\[repr(C)\]

When `#[repr(C)]` is specified, the memory layout is compatible with C/C++ language.
In the case of the sample code, the memory layout of the structure is as follows:
The definition order in the source code matches the placement in memory, and like `#[repr(Rust)]`, due to alignment, 2 bytes of memory are allocated even to variable y which is of type i8, totaling 32 bytes.
Note that empty memory is not initialized with `0x00`, and the random data (`0x72`) that was originally placed remains as is.
Also, in 32-bit binaries, the name management structure is handled with 4 bytes, causing differences in structure byte count, but there are no differences in memory placement.
There are also no differences in structure byte count and memory placement in minimized binaries.

![repr_attribute](images/22-2.png)

### \#\[repr(packed(1))\]

`#[repr(packed(1))]` is an option that can specify memory alignment.
In the case of the sample code, the memory layout of the structure is as follows:
Unlike Rust or C language, 1 byte of memory is allocated to variable y which is of type i8, totaling 7 bytes.
Also, like C language, the definition order in the source code matches the placement in memory.
Note that there are no differences in 32-bit binaries and minimized binaries.

![repr_attribute](images/22-3.png)

## Sample Program Used

```rust
use std::mem;

#[derive(Debug)]
#[repr(Rust)]
struct RustStruct {
    x: i16,
    y: i8,
    z: i32,
    name: String,
}

#[derive(Debug)]
#[repr(C)]
struct CStruct {
    x: i16,
    y: i8,
    z: i32,
    name: String,
}

#[derive(Debug)]
#[repr(packed(1))]
struct Packed1Struct {
    x: i16,
    y: i8,
    z: i32,
}

fn main() {
    let rust_struct = RustStruct {x:1, y:3, z:5, name: String::from("Rust Struct")};
    let c_struct = CStruct {x:2, y:4, z:6, name: String::from("C Struct")};
    let packed_1_struct = Packed1Struct {x:1, y:2, z:3};

    println!("{:?}", rust_struct);
    println!("size: {}", mem::size_of::<RustStruct>());
    println!("{:?}", c_struct);
    println!("size: {}", mem::size_of::<CStruct>());
    println!("{:?}", packed_1_struct);
    println!("size: {}", mem::size_of::<Packed1Struct>());
}
```
