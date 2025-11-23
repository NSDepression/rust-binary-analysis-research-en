# Identifying Functions Generated from the Same Generic

We investigated whether there is a method to identify the source function for functions generated from the same generic function.

## Investigation Results

* When there is processing within a function that triggers a panic using the `core::panic::Location` structure, it is possible to identify functions generated from the same generic function.
  - The `core::panic::Location` structure is a structure that contains the source code path, line number, and column number where the panic occurs.
  - For different functions, line numbers and column numbers do not match, but among functions generated from the same generic function, the source code path, line number, and column number all match.

Note that this method cannot be used for identification when generic functions are inline-expanded (release builds and minimized binaries), or when build options that remove `core::panic::Location` structure information are used.

## Details

In debug builds not affected by optimization, the `core::panic::Location` structure is received as the third argument during function execution as shown below.
By searching for `core::panic::Location` structures in the binary, if each structure has two or more reference locations, the functions referencing it are functions generated from the same generic function.

![generics_function](images/18-1.png)

The contents of the `core::panic::Location` structure are shown below.
In this example, it indicates the path is "src/main.rs", line number is 2, and column number is 5.

![generics_function](images/18-2.png)

If they are functions generated from the same generic function, internal processing similarity is expected to be high, so omitting analysis of functions with the same source or analyzing only differences can reduce analysis time.

Even in 32-bit binaries, although the size of the `core::panic::Location` structure is different, similar characteristics were observed.
