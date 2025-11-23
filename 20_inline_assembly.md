# Inline Assembly

We investigated assembly characteristics of each keyword (in, out, inout, lateout, inlateout, options, clobber) used in inline assembly.

## Investigation Results

Although some characteristic assembly can be confirmed, no assembly characteristics can be confirmed for most keywords.

## Details

When a binary using `clobber_abi` (saves registers not saved by ABI) is release built, processing is added to save xmm6-15 registers to the stack in the function prologue and restore them in the function epilogue, but there are no other characteristics.

![inline_assembly](images/20-1.png)

## Sample Program Used

```rust
#![allow(unused)]
fn main() {
    #[cfg(target_arch = "x86_64")] { //Change to x86 when creating 32bit binary
        use std::arch::asm;

        extern "C" fn foo(arg: i32) -> i32 {
            println!("arg = {}", arg);
            arg * 2
        }

        fn call_foo(arg: i32) -> i32 {
            unsafe {
                let result;
                asm!(
                    "call {}",
                    // Function pointer to call
                    in(reg) foo,
                    // 1st argument in rdi
                    in("rdi") arg,
                    // Return value in rax
                    out("rax") result,
                    // Mark all registers which are not preserved by the "C" calling
                    // convention as clobbered.
                    clobber_abi("C"),
                );
                result
            }
        }
        call_foo(2);
    }
}
```
