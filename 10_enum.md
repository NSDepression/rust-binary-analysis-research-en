# Enums

Rust's enum type is used when selecting one from several different element types. Particularly, Rust's enum type is characterized by its ability to flexibly specify different types for each element.

In this investigation, we examined the `Option` type, which is widely used among Rust's enum types. The `Option` type is an enum that returns `Some` with a value when a value exists, and returns `None` when it does not exist. We investigated how this `Option` type is implemented at the assembly level. We also investigated the assembly of panic-triggering functions such as `expect()` and `unwrap()` that can be used in relation to the `Option` type.

## Investigation Results

* For the `Option` type, a discriminant value indicating `Some` or `None` is returned in the eax register. When eax is 1, it indicates `Some`; when 0, it indicates `None`.
  - For `Some`, its value is returned in the edx register.

**Understanding Discriminants:**

The **discriminant** is a value that identifies which variant of an enum is being used. In Rust:
- The discriminant is typically stored as an integer tag
- Common idiom in decompiled code: checking the discriminant value to determine enum variant
- The `std::mem::discriminant` function can be used to extract discriminant values
- For `Option<T>`: typically 0 for `None`, 1 for `Some(T)`
- For `Result<T, E>`: typically 0 for `Ok(T)`, 1 for `Err(E)`

**Memory Representation (C-like):**
```c
// Option<i32> is similar to:
enum Option_i32 {
    None = 0,    // discriminant = 0
    Some = 1     // discriminant = 1
};

struct Option_i32_representation {
    int discriminant;  // 0 or 1
    int value;         // only valid when discriminant == 1
};
```

Note that we confirmed the implementation is the same for 32-bit binaries, except for argument passing methods and address sizes.

## Details

The sample programs used for the investigation are described [later](#sample-programs-used).

### Option Type

When building the sample program with release build or minimized binary, optimization removes unnecessary processing, so we investigated with debug build binaries that are not affected by optimization.

In Rust's Option type, whether it is `Some` or `None` is returned in the eax register, with 1 indicating Some and 0 indicating None.
When checking the return value of the Option type related processing in the sample program, 1 is stored in the eax register and 2 is stored in the edx register.
The 2 stored in the edx register is the first even number of the array defined in the sample program.
It checks whether the value of the eax register is 0, and if it is 1 (Some), it outputs the value of the edx register, and if it is 0 (None), it outputs the string `No even number found`.

![enum](images/10-1.png)

### expect()

When building the sample program with release build or minimized binary, optimization removes unnecessary processing and `expect()`, so we investigated with debug build binaries that are not affected by optimization.

`expect()` receives as the first argument a value indicating whether it is `Some` or `None`, as the second argument the Option type value, as the third argument the `panic` message, as the fourth argument the number of characters in the panic message, and as the fifth argument the `core::panic::Location` structure.

![enum](images/10-2.png)

The following is the internal processing of expect(), but internally it checks whether it is `Some` or `None`, and if it is `Some`, it returns the value of the second argument as is, and if it is `None`, it triggers a panic.

![enum](images/10-3.png)

### unwrap()

When building the sample program with release build or minimized binary, optimization removes unnecessary processing and `unwrap()`, so we investigated with debug build binaries that are not affected by optimization.

The following is the result of the sample program, but `unwrap()` has been inline-expanded, and the use of `unwrap()` cannot be confirmed from the processing.

![enum](images/10-4.png)

## Sample Programs Used

* Option type

```rust
fn find_first_even_number(numbers: &[i32]) -> Option<i32> {
    for &number in numbers {
        if number % 2 == 0 {
            return Some(number);
        }
    }
    None
}

fn main(){
    let numbers = [1, 2, 3, 4, 5, 6];
    let result = find_first_even_number(&numbers);
    match result {
        Some(even_number) => println!("Even number: {}", even_number),
        None => println!("No even number found"),
    }
}
```

* expect()

```rust
fn find_first_even_number(numbers: &[i32]) -> Option<i32> {
    for &number in numbers {
        if number % 2 == 0 {
            return Some(number);
        }
    }
    None
}

fn main() {
    let numbers = [1, 3, 7, 9];
    let result = find_first_even_number(&numbers);

    let even_number = result.expect("No even number found");

    println!("Even number: {}", even_number);
}
```

* unwrap()

```rust
fn find_first_even_number(numbers: &[i32]) -> Option<i32> {
    for &number in numbers {
        if number % 2 == 0 {
            return Some(number);
        }
    }
    None
}

fn main() {
    let numbers = [1, 3, 7, 9];
    let result = find_first_even_number(&numbers);

    let even_number = result.unwrap();

    println!("Even number: {}", even_number);
}
```

## References

For more information on Rust enums and discriminants:
- [The Rust Reference: Discriminants](https://doc.rust-lang.org/reference/items/enumerations.html#discriminants)
- [std::mem::discriminant](https://doc.rust-lang.org/std/mem/fn.discriminant.html)
- [Reconstructing Rust Types - RE//verse 2025](https://github.com/cxiao/reconstructing-rust-types-talk-re-verse-2025) - Cindy Xiao's presentation on Rust type reconstruction
