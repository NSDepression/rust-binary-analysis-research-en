# Dynamic Dispatch References

We investigated assembly characteristics of dynamic dispatch references using trait objects.

## Investigation Results

In debug build binaries, function calls are made using a VTable structure.
The VTable structure is as follows:

* 64-bit binaries

```c
vtable {
    0x00: Destructor
    0x08: Size of source structure
    0x10: Alignment of source structure
    ..followed by pointers to methods..
}
```

* 32-bit binaries

```c
vtable {
    0x00: Destructor
    0x04: Size of source structure
    0x08: Alignment of source structure
    ..followed by pointers to methods..
}
```

Note that in release builds and minimized binaries, VTable may be removed (converted to general conditional branching).

## Details

In debug build binaries, function calls become indirect calls through the VTable structure due to dynamic dispatch references.
The following is the processing that places the address named VTable by IDA Pro onto the stack.

![iterator](images/16-1.png)

Subsequently, processing to obtain values from VTable and indirect function calls can be confirmed. Each receives the address of the structure as the first argument.

![iterator](images/16-2.png)
