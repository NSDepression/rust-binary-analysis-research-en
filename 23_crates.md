# Methods for Identifying Standard and Third-Party Libraries

When analyzing malware, it is desirable to identify standard library or third-party library code and focus analysis on code created by attackers. In this investigation, we verified methods for identifying standard library and third-party library functions using IDA Pro's FLIRT.

## Investigation Results

* Signatures for standard libraries and third-party libraries can be created using [rustbinsign](https://github.com/N0fix/rustbinsign) and [RIFT](https://github.com/microsoft/RIFT).

* Verification revealed that signature files generated using the same compilation options as the target malware being analyzed are superior in both the number of identified functions and identification accuracy.

## Details

### FLIRT Signature Creation and Application

#### Identifying Used Libraries and Versions

As preparation for signature creation, it is necessary to identify which crates the executable file being analyzed uses and which Rust version is used.
For this purpose, rustbinsign can use the `info` command.
By using this command, it is possible to identify the rust compiler version and the crates being used.
Since the rust compiler and crate names and their versions are extracted from the `file` field of the `Location` structure,
if the `location-detail=none` option is used, the crates being used cannot be obtained.
```
> rustbinsign info s4killer.exe
TargetRustInfo(
    rustc_version='1.87.0',
    rustc_commit_hash='4d30011f6c616be074ba655a75e5d55441232bbb',
    dependencies=[
        Crate(name='crossbeam-deque', version='0.8.5', features=[], repository=None),
        Crate(name='crossbeam-epoch', version='0.9.18', features=[], repository=None),
        Crate(name='crossbeam-utils', version='0.8.19', features=[], repository=None),
        Crate(name='hashbrown', version='0.15.2', features=[], repository=None),
        Crate(name='once_cell', version='1.19.0', features=[], repository=None),
        Crate(name='rayon', version='1.8.1', features=[], repository=None),
        Crate(name='rayon-core', version='1.12.1', features=[], repository=None),
        Crate(name='sysinfo', version='0.30.5', features=[], repository=None),
        Crate(name='windows', version='0.52.0', features=[], repository=None),
        Crate(name='windows-core', version='0.52.0', features=[], repository=None)
    ],
    rust_dependencies_imphash='d9effc64c9481f9d445d7d35ba5db4b9',
    guessed_toolchain='windows-msvc'
)
```
Also, by applying `RIFT` as an IDA Plugin, it can be output as a `json` file.
```
{
    "commithash": "05f9846f893b09a1be1fc8560e33fc3c815cfecb",
    "target_triple": "pc-windows-msvc",
    "arch": "x86_64",
    "crates": [
        "crossbeam-epoch-0.9.18",
        "once_cell-1.19.0",
        "rayon-1.8.1",
        "windows-0.52.0",
        "windows-core-0.52.0",
        "crossbeam-deque-0.8.5",
        "rustc-demangle-0.1.24",
        "sysinfo-0.30.5",
        "hashbrown-0.15.2",
        "crossbeam-utils-0.8.19",
        "rayon-core-1.12.1"
    ]
}
```
![RIFT](images/23-1.png)

#### Standard Library

Standard library signatures can be created with the `sign_stdlib` command.
`rustbinsign` creates signature files for standard library DLLs saved in `C:\Users\<Username>\.rustup\toolchains` using `idat`, `idb2pat.py`, and `sigmake`.
```
> rustbinsign sign_stdlib -t 1.84.0-x86_64-pc-windows-msvc
```
Unless options such as `build-std` described in [No.2 Binary size reduction](gitlab.jpcert.or.jp/irt/rust-binary-analysis-research/-/blob/main/Public/Japanese/2_minimize_binary.md) are added, the compiled standard library is statically linked as is.
Also, since `rustbinsign` creates signatures using the compiled standard library, there are no options to create signatures by changing optimization options etc. for this command.

#### Third-Party Libraries

Third-party library signatures can be created with the `download_sign` or `sign_target` command.
The `download_sign` command specifies the target crate and creates a signature for that crate, whereas the `sign_stdlib` command extracts external library dependencies from the executable and creates signatures.
Signature creation involves downloading crates, creating DLLs by specifying the `crate-type=dylib` option, and then creating signature files using `idat`, `idb2pat.py`, and `sigmake` similar to standard libraries.
```
> rustbinsign download_sign --full-compilation windows-core-0.52.0 1.84.0-x86_64-pc-windows-msvc
```
Additionally, with the `sign_target` command, cargo compilation options can be specified with the `--template` option.

Also, by setting the `RIFT` config file and running a Python script with the `--flirt` option, it is possible to create third-party library signatures with the `release` profile applied.
```
[Default]
PcfPath = <Path to pcf.exe in FLAIR included in IDA>
SigmakePath = <Path to sigmake.exe in FLAIR included in IDA>
DiaphoraPath = <Path to diaphora.py included in Diaphora>
IdatPath = <Path to IDAT>
WorkFolder = <Path for output of signatures etc.>
CargoProjFolder = <Path for output of Cargo projects created for signature generation>
```
```
> py rift.py --cfg rift_config.cfg --input <JSON file output by IDA Plugin> --flirt --output <output destination>
```
In addition to this, `RIFT` can also identify third-party library functions from binary differences instead of FLIRT signatures by running with the `--binary-diff` option and specifying the execution result from the IDA GUI.

### Verification

We verified methods to improve signature identification rates.

#### Verification Method

We compiled [s4killer](https://github.com/gavz/s4killer) using the following Cargo compilation options and rustc options described in [No.2 Binary size reduction](gitlab.jpcert.or.jp/irt/rust-binary-analysis-research/-/blob/main/Public/Japanese/2_minimize_binary.md), and measured function identification rate and identification accuracy.
```
[profile.dev]
strip = false
debug = true

[profile.release]
opt-level = "s"
debug-assertions = false
overflow-checks = false
lto = "fat"
panic = "abort"
codegen-unit = 1
strip = false
debug = true
```

#### Verification Results

As shown in the table below, it was found that when compilation options match, the number of identified functions and identification accuracy tend to be highest.

| Target / Signature | dev          | release      | minsize      |
| ------------------ | ------------ | ------------ | ------------ |
| s4killer(dev)      | 2376 / 72.2% | 352 / 82.6%  | 365 / 79.3%  |
| s4killer(release)  |   97 / 57.6% | 148 / 59.1%  | 151 / 57.8%  |
| s4killer(minsize)  |    2 / 50.0% |  64 / 71.5%  | 274 / 59.5%  |
