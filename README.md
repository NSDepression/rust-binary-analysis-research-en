# Reverse Engineering Research of Rust Binaries

## Introduction

The Rust programming language (hereinafter referred to as Rust) is a language developed by Mozilla and is attracting attention as an alternative to C and C++. Rust excels in memory safety and performance, leading to increased adoption in recent years.

On the other hand, as Rust becomes more widespread as a programming language, there is a growing trend of malware developed using Rust (hereinafter referred to as Rust malware), such as variants of SysJoker and BlackCat ransomware.

However, reverse engineering techniques and knowledge for Rust malware are not yet as well-established as those for malware written in other languages. Binaries created with newer languages like Rust often have different structures compared to binaries created with traditional languages like C, requiring different approaches and knowledge from conventional analysis methods.

This report summarizes the results of our investigation into the knowledge required for reverse engineering binaries created with Rust (hereinafter referred to as Rust binaries).


## Target Audience

This report is intended for the following technical professionals:

* Malware analysts
* Those who reverse engineer Rust binaries
* Those who want to deepen their understanding of Rust's internal structure

## Investigation Environment

This section shows the versions of rustc and cargo used in this investigation. Compilation was performed on Windows using the MSVC environment.

* cargo: 1.82.0
* rustc: 1.82.0

IDA Pro was used for disassembling binaries. The version of IDA Pro used in this investigation is shown below.

* IDA Pro v8.3.230608

## Investigation Topics

This report summarizes the results of investigations on the following topics:

| No. | Topic                                       | Overview                                                                                               |
| --- | -------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| 1   | [Binary differences from Cargo Profile setting changes](1_profile.md) | Investigation of how much size reduction is possible with binary size reduction techniques using cargo obtained from public information, and what information remains                                      |
| 2   | [Binary size reduction](2_minimize_binary.md)                           | Investigation of how much size reduction is possible with binary size reduction techniques using rustc obtained from public information, and what information remains |
| 3   | [Identifying Rust binaries](3_identify_rust_binary.md)                           | Investigation of methods to identify whether a binary is a Rust binary                                                                     |
| 4   | [Exception Directory](4_exception_directory.md)                          | Investigation of information obtained from the Exception Directory structure                                                                    |
| 5   | [TLS Directory](5_tls_directory.md)                                | Investigation of information obtained from the TLS Directory structure and TLS Callback contents                                                           |
| 6   | [Identifying the main function and initialization process](6_identify_main_function.md)                   | Methods for identifying the user-defined main function                                                                         |
| 7   | [Strings](7_strings.md)                                       | How strings are handled                                                                                     |
| 8   | [Function name mangling](8_mangling.md)                         | Structure of mangled function names and demangling methods                                                   |
| 9   | [Closures](9_closure.md)                                   | Closure behavior and memory layout used                                                             |
| 10  | [Enums](10_enum.md)                                       | Investigation of how enum types in Rust are implemented in assembly                                           |
| 11  | [Match expressions](11_match.md)                                      | Investigation of how match expressions in Rust are implemented in assembly                                        |
| 12  | [Panic](12_panic.md)                                      | Assembly differences between unwind and abort behaviors during panic                                               |
| 13  | [Iterators](13_iterator.md)                                   | Investigation of how code using iterators and next functions is implemented in assembly                               |
| 14  | [Traits](14_trait.md)                                     | Differences between trait-based function calls and regular function calls |
| 15  | [Identifying common traits](15_derive_attribute.md)                       | Methods for identifying traits used with the #\[derive\] attribute in assembly                                          |
| 16  | [Dynamic dispatch references](16_dynamic_dispatch.md)                         | Assembly characteristics and differences between dynamic/static dispatch calls                                       |
| 17  | [Collections](17_collection.md)                                 | Memory layout used                                                                               |
| 18  | [Identifying functions generated from the same generic](18_generics_function.md)     | Investigation of methods to identify the source function                                                               |
| 19  | [Smart pointers](19_smart_pointer.md)                             | Characteristics and memory layout of smart pointers                                                                 |
| 20  | [Inline assembly](20_inline_assembly.md)                         | Characteristic code patterns                                                                                   |
| 21  | [link attribute](21_link_attribute.md)                                     | Differences in library linking methods                                                                             |
| 22  | [repr attribute](22_repr_attribute.md)                                     | Investigation of how memory layout changes with available options                                       |
| 23  | [Methods for identifying standard and third-party libraries](23_crates.md)     | Methods for identifying statically linked standard library and third-party library functions                                 |

## Requests and Corrections

If you have requests for additional investigations or find errors in this report, please contact us via Issue or Pull Request.
