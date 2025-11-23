# Identifying Rust Binaries

For the purpose of identifying Rust binaries, we investigated strings and byte sequences specific to release build and minimized build Rust binaries.

## Investigation Results

As a result of the investigation, we confirmed that it is partially possible to detect minimized Rust binaries with YARA rules by combining multiple characteristic strings with the `core::panic::Location` structure that uses those strings.

## Details

Based on the investigation results from [Binary differences from Cargo Profile setting changes](1_profile.md), [Binary size reduction](2_minimize_binary.md), and [Identifying the main function and initialization process](6_identify_main_function.md), we created a YARA rule to detect Rust binaries.

```yara
private rule PE_Signature
{
    condition:
        uint16(0) == 0x5A4D
}

private rule Plain_Rust_Binary
{
    metadata:
        description = "Detect plain Rust binary"
        author = "JPCERT/CC Incident Response Group"

    strings:
        $s1 = "run with `RUST_BACKTRACE=1` environment variable to display a backtrace"
        $s2 = "called `Result::unwrap()` on an `Err` value"
        $s3 = "called `Option::unwrap()` on a `None` value"

    condition:
        PE_Signature and all of them
}

private rule Minsized_Rust_Binary
{
    metadata:
        description = "Detect minsized Rust binary"
        author = "JPCERT/CC Incident Response Group"

    strings:
        $s1 = "<unknown>"
        $s2 = "<redacted>"
        $s3 = "failed to write whole buffer"
        $s4 = "failed to write the buffered data"
        $c1_64 = {1C 00 00 00 00 00 00 00 17 00 00 00 00 00 00 00}
        $c1_32 = {1C 00 00 00 17 00 00 00}
        $c2_64 = {21 00 00 00 00 00 00 00 17 00 00 00 00 00 00 00}
        $c2_32 = {21 00 00 00 17 00 00 00}

    condition:
        PE_Signature and 3 of ($s*) and 1 of ($c1*) and 1 of ($c2*)
}

rule Rust_Binary
{
    metadata:
        description = "Detect Rust binary"
        author = "JPCERT/CC Incident Response Group"

    condition:
        Plain_Rust_Binary or Minsized_Rust_Binary
}
```
