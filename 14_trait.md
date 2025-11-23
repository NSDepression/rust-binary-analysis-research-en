# Traits

We investigated how function calls implementing traits in Rust differ from normal function calls, and whether the source trait can be identified.

## Investigation Results

In the case of static dispatch where no virtual function table (VTable) exists, as in [Dynamic dispatch references](16_dynamic_dispatch.md), no clear differences from normal function calls were confirmed, and identifying the source trait at the assembly level is difficult.

Similar results were obtained for 32-bit binaries, minimized binaries, and debug builds.

## Details

Omitted
