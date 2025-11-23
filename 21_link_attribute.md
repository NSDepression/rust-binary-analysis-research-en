# link Attribute

Using Rust's `link` attribute allows specifying library linking methods. In this investigation, we examined what differences occur in assembly when specifying each setting value: `dylib`, `static`, and `raw-dylib`.

## Investigation Results

No differences were confirmed from normal functions in calling methods or implementation.

## Details

### static

The following shows calling add() in a binary linked with static.
Library functions are embedded in the executable, and there are no differences from normal functions in calling methods or implementation.

![link_attribute](images/21-1.png)

### dylib

Not investigated because Windows only supports dynamic linking with `raw-dylib`.

### raw-dylib

The following shows calling `add()` in a binary linked with raw-dylib.
Like calling Windows APIs, it becomes a function call using IAT.

![link_attribute](images/21-2.png)
