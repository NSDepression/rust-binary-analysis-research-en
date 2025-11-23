# Identifying Common Traits

We investigated whether representative traits (**Eq**, **Drop**, **Clone**, **Copy**) that are automatically implemented with the `#[derive]` attribute can be identified in assembly.

## Investigation Results

* Similar to the [Traits](14_trait.md) investigation, there are no uniquely identifiable features for all of these traits.
* Many processes are inlined, making identification difficult as there is little difference from generic code.
* In cases like static dispatch references where there are no data structures outside functions such as virtual function tables, trait identification is particularly difficult.
* Other traits may have similar results, but this has not been verified at this time.

## Details

Omitted
