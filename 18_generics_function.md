# Identifying Functions Generated from the Same Generic

We investigated whether there is a method to identify the source function for functions generated from the same generic function.

## Investigation Results

* When there is processing within a function that triggers a panic using the `core::panic::Location` structure, it is possible to identify functions generated from the same generic function.
  - The `core::panic::Location` structure is a structure that contains the source code path, line number, and column number where the panic occurs.
  - For different functions, line numbers and column numbers do not match, but among functions generated from the same generic function, the source code path, line number, and column number all match.

Note that this method cannot be used for identification when generic functions are inline-expanded (release builds and minimized binaries), or when build options that remove `core::panic::Location` structure information are used.

### Understanding Monomorphization

Rust implements generics through **monomorphization**, a compile-time process where the compiler generates a separate copy of generic code for each concrete type used. This approach provides zero-cost abstractions and excellent runtime performance, but presents unique challenges for reverse engineers:

**Advantages for Performance:**
- No runtime overhead for generic types
- Enables aggressive optimization for each concrete type
- Allows inline expansion and dead code elimination

**Challenges for Reverse Engineering:**

1. **Binary Size Explosion**: A single generic function can generate dozens of near-identical copies in the binary, significantly increasing binary size and analysis complexity.

2. **Function Similarity Without Identity**: Multiple monomorphized instances will have very similar assembly patterns, making it difficult to determine:
   - Whether two similar functions are monomorphs of the same generic
   - What the original generic function looked like
   - Which concrete types were instantiated

3. **Symbol Confusion**: As noted by CheckPoint Research:
   > "If your similarity engine slaps the label impl some_func for SomeType on a function and you actually suspect this verdict is not just a hallucination and the engine is on to something, consider whether this impl is a monomorph; if it is, then take the impl some_func part seriously, but the for SomeType part not so much."

   In other words, while the function name may be reliable, the specific type annotation may not accurately reflect the instantiated type.

4. **Analysis Time Multiplication**: Analyzing every monomorphized variant separately wastes time, as they share the same logic with minor type-specific variations.

**Practical Implications:**

- **Detection Strategy**: Look for repeated code patterns with slight variations in type sizes or method calls
- **Analysis Optimization**: Once you identify functions as monomorphs, analyze one representative instance and note the variations rather than analyzing each separately
- **Binary Size Trade-offs**: Release builds with aggressive optimization may inline many monomorphs, making them even harder to identify but reducing the total number of distinct function instances

## Details

In debug builds not affected by optimization, the `core::panic::Location` structure is received as the third argument during function execution as shown below.
By searching for `core::panic::Location` structures in the binary, if each structure has two or more reference locations, the functions referencing it are functions generated from the same generic function.

![generics_function](images/18-1.png)

The contents of the `core::panic::Location` structure are shown below.
In this example, it indicates the path is "src/main.rs", line number is 2, and column number is 5.

![generics_function](images/18-2.png)

If they are functions generated from the same generic function, internal processing similarity is expected to be high, so omitting analysis of functions with the same source or analyzing only differences can reduce analysis time.

Even in 32-bit binaries, although the size of the `core::panic::Location` structure is different, similar characteristics were observed.

## References

For more detailed information:
- [CheckPoint Research: Rust Binary Analysis, Feature by Feature](https://research.checkpoint.com/2023/rust-binary-analysis-feature-by-feature/)
