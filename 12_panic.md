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

However, by using `-Zlocation-detail=none` used for building minimized binaries, the `core::panic::Location` structure becomes as follows, and panic-related information is removed.

![panic](images/12-3.png)

### Assembly Differences Between unwind and abort

Rust specifies either `unwind` or `abort` for behavior during panic.
`unwind` is the default behavior, and when a program panics, it unwinds the stack, releases resources, and then terminates the program.
`abort` does not unwind the stack and immediately terminates the program.
During `unwind` behavior, the following code (such as `drop_in_place` functions) is added to the assembly to release resources.

![panic](images/12-4.png)

### Differences in 32-bit and Minimized Binaries

In 32-bit binaries, the characteristics shown in [Characteristics of Panic-Related Assembly](#characteristics-of-panic-related-assembly) were confirmed.

In minimized binaries, panic-related processing was omitted due to the effect of `-Zbuild-std-features=panic_immediate_abort` specified in the build options.
