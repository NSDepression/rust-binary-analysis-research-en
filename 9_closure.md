# Closures

We investigated closure behavior and the memory layout used.

## Investigation Results

* Closures implicitly receive as their first argument the address of a stack region that stores the addresses of variables to be captured (hereinafter referred to as `closure_env`). Even when no variables are captured, the first argument is `closure_env`.

* In closures that capture closures, the address of the child closure being captured is not referenced through the parent's `closure_env`, but is directly referenced inside the parent closure.

* However, variables captured by child closures are structured such that the child's `closure_env` is embedded within the parent's `closure_env`.

## Details

The sample programs used for the investigation are described [later](#sample-programs-used).

### Basic Closure

When the sample program was release built, as shown in the figure below, the return value of `sum()`, which is 15, was embedded in the binary due to optimization.
From this, simple closures cannot be identified due to optimization.

![closure](images/9-1.png)

To avoid optimization, the result of debug building the sample program (function sum) is shown in the figure below.
In 64-bit binaries, normally the first argument is stored in the rcx register, the second argument in the rdx register, and the third argument in the r8 register, but when checking the arguments of the above function, the explicit arguments 5 and 10 are passed as the second and third arguments.

![closure](images/9-2.png)

The rcx register, which stores the first argument, contains the address of the stack named `arguments::main::closure_env$0 *` by IDA Pro.
The address of the stack passed to the first argument of a closure stores the address of variables captured by the closure, `closure_env`, but since this sample program is a closure that does not capture variables, the address passed to the first argument of the closure is unused.

### Closure That Captures Variables

Similar to the previous section, closures become unidentifiable due to optimization, so we investigated with debug builds.
The figure below shows the call site of `add_to_x()` during debug build in the sample program.
The value `0x0A(10)` of local variable x is stored in `var_D4`.
Subsequently, the address of `var_D4` is stored in `var_D0`, which indicates `closure_env`.
In this way, the address of the stack passed to the first argument of a closure stores the addresses of variables captured by the closure.

![closure](images/9-3.png)

Inside the closure `_ZN7capture4main28_$u7b$$u7b$closure$u7d$$u7d$17he4cd0740b21072e5E()`, the value extracted from `closure_env` is added to the explicit argument 5, and this processing matches the processing of `add_to_x()`.

![closure](images/9-4.png)

### Closure That Captures Closures

Similar to the previous section, closures become unidentifiable due to optimization, so we investigated with debug builds.
The following shows the call site of `double()` during debug build in the sample program.

![closure](images/9-5.png)

`_ZN15capture_closure4main28_$u7b$$u7b$closure$u7d$$u7d$17hc3f3366177092b36E()` is `double()`.
`closure_env` stores the user input string, and the `closure_env` of `add_one()` and `sub_one()`.
The structure of `closure_env` in this sample program is described below.

```
closure_env {
   0x00: User input string
   0x08: closure_env of add_one()
   0x10: closure_env of sub_one()
}
```

The following is where `add_one()` is called, and it gives `closure_env + 0x08` as the first argument, showing that the address of the above structure is used as `closure_env`.

![closure](images/9-6.png)

### Differences in 32-bit and Minimized Binaries

In 32-bit binaries, it is the same for all samples except for the argument passing method and the number of bytes in addresses.
Also, in minimized binaries, due to optimization effects, closures are inline-expanded in all samples, and simple calculation results are embedded in the binary.

## Sample Programs Used

* Basic closure

```rust
fn main() {
    let sum = |a: i32, b: i32| -> i32 {
        a + b
    }

    println!("The sum of 5 and 10 is: {}", sum(5, 10));
}
```

* Closure that captures variables

```rust
fn main() {
    let x = 10;

    let add_to_x = |y: i32| -> i32 {
        x + y
    };

    println!("The result of adding 5 to {} is: {}", x, add_to_x(5));
}
```

* Closure that captures closures

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();

    if args.len() != 2 {
        eprintln!("Usage: {} <add|sub>", args[0]);
        return;
    }

    let operation = &args[1];

    let add_one = |x: i32| x + 1;
    let sub_one = |x: i32| x - 1;

    let double = |y: i32| {
        match operation.as_str() {
            "add" => add_one(y) * 2,
            "sub" => sub_one(y) * 2,
            _ => {
                eprintln!("Invalid operation. Use 'add' or 'sub'.");
                std::process::exit(1);
            }
        }
    };

    let input_value = 3;

    println!("The result is: {}", double(input_value));
}
```
