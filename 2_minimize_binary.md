# Binary Size Reduction

Attackers may attempt to reduce binary size to hide traces. Therefore, we investigated the binary size reduction techniques introduced in [public information](https://github.com/johnthagen/min-sized-rust) to determine how much size reduction is possible and what information can be obtained from data that remains after reduction.

## Investigation Results

The build command used to minimize binary size is as follows:

```
$env:RUSTFLAGS="-Zfmt-debug=none -Zlocation-detail=none";cargo +nightly build -Z build-std=std,panic_abort -Z build-std-features="optimize_for_size" -Z build-std-features=panic_immediate_abort --target x86_64-pc-windows-msvc --release
```

## Details

### Remove Location Details

The `location-detail` option is a build option that can control backtrace information.
By specifying this build option as `none`, the value indicating the source code path in the `core::panic::Location` structure used during panics is replaced with `<redacted>`, and the values indicating the line and column of the source code are also replaced with 0.
By specifying the `location-detail` option as `none`, the source code path is removed, enabling binary size reduction.
Additionally, removing this information prevents backtraces from being output normally and is expected to affect investigation of library identification.
A YARA rule using the `core::panic::Location` structure containing `<redacted>` as a signature to determine whether `-Zlocation-detail=none` is used is shown below.

```yara
rule Detect_LocationDetail_Is_None
{
    metadata:
        description = "Detect Rust binary with option remove location details"
        author = "JPCERT/CC Incident Response Group"

    strings:
        $s1 = "<redacted>"
        /*
            pub struct Location<'a> {
                file: &'a str,
                line: u32,
                col:  u32,
            }
        */
        $c1 = {
            0A 00 00 00 00 00 00 00
            00 00 00 00
            00 00 00 00
        }
        $c2 = {
            0A 00 00 00
            00 00 00 00
            00 00 00 00
        }

    condition:
        Rust_Binary and $s1 and ($c1 or $c2)
}
```

### Remove fmt::Debug

The `fmt-debug` option can manipulate the output of `#[derive(Debug)]` and `{:?}`.
When this build option is set to `-Zfmt-debug=none`, output from macros such as `dbg!()`, `assert!()`, and `unwrap()` that internally use `#[derive(Debug)]` and `{:?}` is partially removed.
Therefore, binary size can be reduced by using `-Zfmt-debug=none`.

### Optimize libstd with build-std

The standard library linked to binaries is optimized for speed even in the default state, but not for size.
With this build option `-Z build-std=std,panic_abort -Z build-std-features="optimize_for_size"`, it is possible to rebuild with the standard library specified and optimized for size.
When a binary optimized for size is linked, the executable file size can be reduced.

### Remove panic String Formatting with panic_immediate_abort

The `panic="abort"` setting available in `Cargo`'s `Profile` configuration does not apply to the standard library.
By using the build option `-Z build-std-features=panic_immediate_abort`, `panic="abort"` can also be applied to the standard library.
Using this build option can reduce binary size.

Additionally, binaries with `panic_immediate_abort` specified have the characteristic strings explained in [Identifying Rust binaries](3_identify_rust_binary.md) removed, so they may not be recognized as Rust binaries.
Furthermore, since panic messages are removed, this is expected to affect identification of libraries being used.
