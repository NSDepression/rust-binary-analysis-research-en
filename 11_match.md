# Match Expressions

We investigated the characteristics of match expressions in Rust at the assembly level.

## Investigation Results

* Enum types
  - Although processing may be omitted during optimization, it is basically converted to branching using cmp and jmp instructions, or branching using jump tables. No assembly instructions specific to match expressions were confirmed.

* Strings
  - For strings, we found that specific branching methods may be applied.

## Details

### Enum Types

Omitted

### Strings

Match expressions using strings have similar characteristics in release builds and minimized binaries.
As shown in the assembly of the comparison section below, string comparison is performed after comparing string lengths.
Through optimization, it is considered that unnecessary string comparisons are avoided by first comparing string lengths, which can be processed more quickly.
For string comparison, commonly used string comparisons employ the cmp instruction or APIs such as `memcmp()` or `strcmp()`, but in this binary, the xor instruction was used for string comparison.
This is a comparison method that utilizes the property that the result of xor operations between the same numbers is 0.
Note that 32-bit binaries have similar characteristics.

![match](images/11-1.png)

## Sample Programs Used

* Strings

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();

    if args.len() < 2 {
        return;
    }

    let input = &args[1];

    match input.as_str() {
        "apple" => println!("this is an apple."),
        "banana" => println!("this is a banana."),
        "orange" => println!("this is a orange."),
        "grape" => println!("this is a grape."),
        "kiwi" => println!("this is a kiwi."),
        _ => println!("this is an unknown fruit."),
    }
}
```
