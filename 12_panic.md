# Panic

We investigated panic-related assembly characteristics in Rust and aimed to clarify the assembly differences between `unwind` and `abort` behaviors during panic.

## Investigation Results

* The `panic!()` macro expands to a call to the `core::panicking::panic_fmt()` function. At this time, a `core::panic::Location` structure containing the filename, line number, and column number where the panic occurred is passed as an argument.

* Panic-related assembly was confirmed to include calls to non-returning functions (functions that do not return), and calls to Windows APIs such as `RtlFailFast()` and `_CxxThrowException()`.

* Regarding panic behavior, when using the `unwind` option, there is processing that releases resources such as strings. On the other hand, when using the `abort` option, such resource release processing does not exist.

## Details

### Characteristics of Panic-Related Assembly

The `panic!()` macro expands to the following processing that calls the function `core::panicking::panic_fmt()`.
`core::panicking::panic_fmt()` is the entry point function that triggers a panic along with a panic message.

![panic](images/12-1.png)

This function receives the formatted panic message as the first argument and the `core::panic::Location` structure as the second argument.
The `core::panic::Location` structure stores the path to the source code, line number, and column number where the panic occurred, which can be identified as information for the start of panic-related processing.

![panic](images/12-2.png)

### Using Panic Metadata for Reverse Engineering

The `core::panic::Location` structure is particularly valuable for reverse engineering because:

**Structure Layout:**
The Location type contains:
- A string slice reference (`&str`) for the file path, which consists of:
  - A pointer (address where the source file path string resides)
  - A length (64-bit integer on 64-bit systems)
- A `u32` for the line number
- A `u32` for the column number

**Binary Representation (64-bit, both x86-64 and ARM64):**
```
Offset 0x00: Pointer to file path string (8 bytes)
Offset 0x08: Length of file path string (8 bytes)
Offset 0x10: Line number (4 bytes)
Offset 0x14: Column number (4 bytes)
Total: 20 bytes (with 4 bytes padding to 24 bytes for alignment)
```

**Calling Convention for Panic Functions:**

When `rust_begin_unwind` or similar panic functions are called:

**x86-64:**
- First argument (panic message `&str`): pointer in `rdi`, length in `rsi`
- Second argument (`&Location`): pointer to Location structure in `rdx`

**ARM64:**
- First argument (panic message `&str`): pointer in `x0`, length in `x1`
- Second argument (`&Location`): pointer to Location structure in `x2`

**⚠️ Important:** Rust's calling convention for Rust-to-Rust function calls is not stable and may change between compiler versions. The register usage patterns described represent common observations but should not be considered guaranteed.

**Extraction Benefits:**
All of this information is embedded in Rust binaries by default and is recoverable statically. You can:
- Automatically extract all panic location metadata from the binary
- Find the locations in the code where they are referenced
- Annotate each location with tags displaying the extracted source file information
- Use panic strings to locate relevant source code positions

**Practical Application:**
Since Rust's standard library and the majority of third-party Rust libraries are open-source, you can use the panic strings to find the relevant location in the source code. This is especially useful for identifying library functions and understanding program flow.

However, by using `-Zlocation-detail=none` used for building minimized binaries, the `core::panic::Location` structure becomes as follows, and panic-related information is removed.

![panic](images/12-3.png)

**Note:** Developers can build binaries without location details using:
```bash
RUSTFLAGS="-Zlocation-detail=none" cargo +nightly build --release
```
This removes the valuable source code path information, making reverse engineering more difficult.

### Assembly Differences Between unwind and abort

Rust specifies either `unwind` or `abort` for behavior during panic.
`unwind` is the default behavior, and when a program panics, it unwinds the stack, releases resources, and then terminates the program.
`abort` does not unwind the stack and immediately terminates the program.
During `unwind` behavior, the following code (such as `drop_in_place` functions) is added to the assembly to release resources.

![panic](images/12-4.png)

### Differences in 32-bit and Minimized Binaries

In 32-bit binaries, the characteristics shown in [Characteristics of Panic-Related Assembly](#characteristics-of-panic-related-assembly) were confirmed.

In minimized binaries, panic-related processing was omitted due to the effect of `-Zbuild-std-features=panic_immediate_abort` specified in the build options.

## References

For more detailed information on using panic metadata for reverse engineering:
- [Using panic metadata to recover source code information from Rust binaries](https://cxiao.net/posts/2023-12-08-rust-reversing-panic-metadata/) by Cindy Xiao
- [Rust Binary Analysis, Feature by Feature](https://research.checkpoint.com/2023/rust-binary-analysis-feature-by-feature/) by Check Point Research
