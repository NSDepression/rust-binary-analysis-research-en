# Identifying the main Function and Initialization Process

This investigation examined methods to identify the user-defined main function in Rust, the processing flow leading to its invocation, and the initialization processes executed before the main function is called.

## Investigation Results

An overview of the process of calling the user-defined main function is shown in the figure below.
The user-defined main function is specified as the first argument when calling the `lang_start_internal` function.
Therefore, the user-defined main function can be identified by confirming the call to the `lang_start_internal` function.
We confirmed that the processing flow is identical even for 32-bit binaries.

![main function flow](images/6-1.png)

Note that in minimized binaries, the `lang_start_internal` function is inlined and the user-defined main function is called directly.

## Details

### Identifying the main Function

#### For 64-bit Binaries

The call site of the `lang_start_internal` function in a release build binary is shown below.
The user-defined main function is placed on the stack, and its address is specified as the first argument to the `lang_start_internal` function.
Additionally, the second argument of the `lang_start_internal` function receives the address named unk_140019330, which is a virtual table for dynamic dispatch references.

![main function flow](images/6-2.png)

Furthermore, a portion of the internal processing of the `lang_start_internal` function is shown below.

![main function flow](images/6-3.png)

It calls offset 0x28 of the virtual table received as the second argument.
It also passes the address where the user-defined main function is placed as the first argument.
Offset 0x28 of the virtual table received as the second argument is shown below.

![main function flow](images/6-4.png)

At offset 0x28, the address of sub_140001000() is placed.
This function internally calls the user-defined main function.

#### For 32-bit Binaries

For 32-bit binaries, although there are differences in argument passing methods, the flow of how the user-defined main function is called is identical to 64-bit binaries.

![main function flow](images/6-5.png)

#### For Minimized Binaries

In minimized binaries, the `lang_start_internal` function is inlined, and there is no location where the `lang_start_internal` function is directly called.
Inside the function named main, processing similar to the internal processing of the `lang_start_internal` function can be confirmed.

![main function flow](images/6-6.png)

The following is the call site of the user-defined main function.
Unlike release build binaries, the user-defined main function is called directly.
Additionally, characteristic processing before and after includes accessing TLS slots and checking values placed in the .data section.

![main function flow](images/6-7.png)

### Internal Processing of lang_start_internal Function

#### Exception Handler Configuration

Configures the exception handler for when stack overflow occurs.
`std__sys__pal__windows__stack_overflow__vectored_handler()` is registered as the exception handler through the Windows API `AddVectoredExceptionHandler()`.

![main function flow](images/6-8.png)

#### Minimum Stack Size Configuration

Configures the minimum stack size for the current thread and thread description.
Settings are configured using Windows APIs such as `SetThreadStackGuarantee()` and `SetThreadDescription()`.

![main function flow](images/6-9.png)

#### Cleanup Processing

Performs cleanup processing after calling the user-defined main function.

![main function flow](images/6-10.png)
