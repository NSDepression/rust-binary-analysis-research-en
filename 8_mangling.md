# Function Name Mangling

We investigated the structure of mangled function names and demangling methods.

## Investigation Results

There are two versions of symbol mangling in Rust: `legacy` and `v0`. The current default is `legacy`, and in this version, the mangled symbol name includes the module path where the function is defined and a hash value based on type information.

Additionally, using the Rust demangling tool [rustfilt](https://github.com/luser/rustfilt), symbols of both `legacy` and `v0` versions can be demangled.

## Details

### Mangling Versions

There are two versions of Rust symbol mangling: `legacy` and `v0`, with `legacy` being the default version.
The `v0` version is proposed in [RFC2603](https://rust-lang.github.io/rfcs/2603-rust-symbol-name-mangling-v0.html) and can be specified with the `symbol-mangling-version` option.

### Structure of Symbols Mangled with `legacy` Version

The structure of symbols mangled with the `legacy` version is as follows:

![legacy-mangling](images/8-1.png)

### Structure of Symbols Mangled with `v0` Version

The structure of symbols mangled with the `v0` version is as follows:
It differs from the `legacy` version in that generic parameter information is not lost.
Details of the `v0` version are described in [v0 Symbol Format](https://doc.rust-lang.org/stable/rustc/symbol-mangling/v0.html).

![v0-mangling](images/8-2.png)

### Demangling Methods

Function name symbols are retained in a mangled state.
The following is a symbol for the function `fn test_add<T>(a: T, b: T) -> T where T: std::ops::Add<Output = T>` monomorphized with `i32`.

```
[legacy]
_ZN3no88test_add17he991e478e44b9c5fE

[v0]
_RINvCsg9W1Qrgvbiz_3no88test_addlEB2_
```

Mangled symbols like the above can be demangled using `rustfilt` included in the Rust Toolchain, regardless of version.
An example of demangling the above function name using `rustfilt` is shown below.

```
PS> rustfilt '_ZN3no88test_add17he991e478e44b9c5fE'
no8::test_add

PS> rustfilt '_RINvCsg9W1Qrgvbiz_3no88test_addlEB2_'
no8::test_add::<i32>

```

### Differences in 32-bit and Minimized Binaries

The structure of mangled symbols is the same even for 32-bit binaries.
However, for the same function, there are differences in the hash portion between 32-bit and 64-bit.
This is considered to be because the function's type information is involved in creating the hash value.
Additionally, in minimized binaries, mangled symbols are not included in the pdb file.
