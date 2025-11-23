# Strings

We investigated how Rust's String type and &str type are handled at the assembly level.

## Investigation Results

* String type
  - The string body is placed on the heap. A structure is created on the stack that manages the String type string by holding the maximum number of characters that can be placed on the heap, a pointer to the string, and the current number of characters.

* &str type
  - The string body is placed in the .rdata section. A structure is created on the stack that manages the &str type string by holding a pointer to the string and the string length.

* Raw string literal, C string literal, binary string literal
  - In the binary, there is no difference in handling compared to regular strings.

Additionally, even in 32-bit binaries and minimized binaries, no differences were observed in the structure or handling for managing strings, except for address size.

## Details

### String Layout

* String type

According to Rust's official website, the String type is Vec<u8>.
As described in [Collections](17_collection.md), the Vec type is composed of a structure consisting of the maximum number of elements, an address to the buffer, and the current number of elements.

![strings](images/7-1.png)

* &str type

A data structure consisting of a buffer to the string and the number of characters is constructed on the stack, and the &str type is managed with the following data structure.

![strings](images/7-2.png)

### Various String Literals

* Raw string literal

In normal programs, `\n` should be converted to 0x0A (newline code), but when using `r""`, `\n` is represented as 0x5C (representing `\`) and 0x6E (representing `n`) and is embedded in the binary in an unescaped state.
Otherwise, there is no difference in handling compared to regular strings.

![strings](images/7-3.png)

* C string literal

There is no difference in handling compared to regular strings.
Embedded in the `.text` section.

* binary string literal

There is no difference in handling compared to regular strings.
Defined in the `.rdata` section.
