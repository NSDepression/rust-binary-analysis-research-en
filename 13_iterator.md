# Iterators

We investigated the memory layout of iterators in Rust, aiming to clarify the iterator structure and characteristics of the frequently used `Vec` type and `VecDeque` type.

## Investigation Results

* **Vec Type Iterator**

  * Composed of two elements: the start address and end address of the buffer.
  * In release builds and minimized binaries, no traces of iterator usage can be confirmed due to optimization.

* **VecDeque Type Iterator**

  * Since it is designed as a circular buffer, it is composed of the following four pointers:

    1. Address indicating the first data
    2. Buffer end address
    3. Buffer start address
    4. Address indicating the end data + 1
  * This structure realizes efficient iteration that leverages the characteristics of circular data.

### Memory Layout of Vec Type Iterator

The memory layout of the Vec type iterator is shown in the figure below.
For the management structure of the Vec type, refer to [Collections](17_collection.md).

![iterator](images/13-1.png)

### Memory Layout of VecDeque Type Iterator

The memory layout of the VecDeque type iterator is shown in the figure below.
For the management structure of the VecDeque type, refer to [Collections](17_collection.md).

![iterator](images/13-2.png)

## Details

The sample programs used for the investigation are described [later](#sample-programs-used).

### Vec Type Iterator

#### When Not Affected by Optimization (Debug Build)

The following shows the call site of iter() in the sample program.

![iterator](images/13-3.png)

iter() receives the start address of Vec as the first argument and the current number of elements of Vec as the second argument.
These values are obtained from the management structure of the Vec type.
Internally, to calculate the offset of the end element of Vec, it performs the calculation of current number of elements × bytes per element (4 bytes).
Then, it adds the offset of the end element to the start address to calculate the end address.
Finally, it stores the start address in the rax register and the end address in the rdx register, and returns.

![iterator](images/13-4.png)

The subsequent calls to `filter()`, `cloned()`, and `collect()` receive the start address as the first argument and the end address as the second argument, and access each element of Vec.

#### When Affected by Optimization (Release Build)

In binaries built with release build, iterators are not used, and high-level processing such as `filter()`, `cloned()`, and `collect()` cannot be confirmed in assembly.
These processes are considered to have been inline-expanded by compiler optimization and unnecessary parts removed.

The following is the part in the sample program that checks each element of Vec for even numbers, extracts only odd numbers, and creates a new Vec.

![iterator](images/13-5.png)

At address `0x00007FF6A4041360`, a Vec end check is performed.
The r12 register stores the offset to the currently pointed element, and `0x1C(28)` indicates the offset of the end in a Vec of 4 bytes × 7 elements.
In the block starting from address `0x00007FF6A4041366`, the following processing is performed:

* The current element is retrieved from Vec, and 4 is added to the offset stored in the r12 register to point to the next element.
* Even/odd determination is performed using the test instruction and immediate value 1. This determination utilizes the property that even numbers have 0 in the 1st bit and odd numbers have 1 in the 1st bit.
  - As a result of the determination, if even, it transitions to the end check, and if odd, it transitions to processing that adds the element to a new Vec of only odd numbers.

At address `0x00007FF6A4041374`, a Vec capacity check is performed.
If the maximum capacity has already been reached, it transitions to address `0x00007FF6A404137A` to secure new capacity.
Then, at address `0x00007FF6A4041389`, an element is added to Vec, and the current number of elements in the management structure is incremented.

In this way, in binaries where optimization has been performed, iterators are not used, and the assembly is similar to array access in C/C++ language binaries.

### VecDeque Type Iterator

In binaries built with release build, high-level processing such as `filter()` and `map()` cannot be confirmed in assembly, but the call to iter() has been inline-expanded due to optimization, and the processing to obtain an iterator can be confirmed.
The following shows the assembly of `iter()` in the sample program.

![iterator](images/13-6.png)

From address `0x00007FF67AD11820` to `0x00007FF67AD11864`, the following data is obtained:

1. Address indicating the first data
2. Address indicating the end of the buffer
3. Start address of the buffer
4. Address indicating the end data + 1

These data are used for accessing the VecDeque type and loop processing, and this structure corresponds to the iterator.
Additionally, the function called at address `0x00007FF67AD11880` receives the iterator as the second argument and executes the closure passed to `filter()` and `map()`.

### Differences in 32-bit and Minimized Binaries

* For Vec type and VecDeque type, there are no differences in iterator structure between 32-bit and 64-bit programs except for address size.
* For Vec type, iterators are omitted in release builds and minimized builds.
* For VecDeque type, iterators are used in release builds and minimized builds.
  - No differences were confirmed in iterator structure or acquisition processing.

## Sample Programs Used

* Vec type sample program

```rust
fn main() {
    // 1. Create Vec
    let mut vec_num = vec![1, 2, 3, 4, 5];

    // 2. Vec length and capacity
    println!("vec_num length: {}, capacity: {}", vec_num.len(), vec_num.capacity());

    // 3. Add element
    vec_num.push(6);
    println!("vec_num length: {}, capacity: {}", vec_num.len(), vec_num.capacity());

    vec_num.insert(3, 99);

    // 4. Iteration
    // Make vec_num a Vec of only odd numbers
    let odd_numbers: Vec<i32> = vec_num.iter()
        .filter(|&num| num % 2 != 0)
        .cloned()
        .collect();

    println!("odd_numbers elements:");
    for value in &odd_numbers {
        println!("{}", value);
    }
}
```

* VecDeque type sample program

```rust
use std::collections::VecDeque;

fn main() {
    // 1. Create VecDeque
    let mut vec_num = VecDeque::from([2, 3, 4]);
    let vec_iter_test = VecDeque::from([0, 1, 2]);

    let it = vec_num.iter();

    // 2. VecDeque length and capacity
    println!("vec_num length: {}, capacity: {}", vec_num.len(), vec_num.capacity());

    // 3. Add elements
    vec_num.push_front(1);
    vec_num.push_front(0);
    vec_num.push_back(5);
    vec_num.push_back(6);
    vec_num.push_back(7);
    vec_num.push_back(8);
    vec_num.push_back(9);
    vec_num.push_back(10);

    let it2 = vec_num.iter();

    let num = vec_num[3];
    println!("{}", num);

    // 4. Remove element
    vec_num.pop_front();

    // 5. Iterator trait
    let numbers: VecDeque<i32> = vec_num.iter()
        .filter(|&&num| num % 2 != 0)
        .map(|&num| num * 2)
        .collect();

    println!("numbers elements:");
    for value in &numbers {
        println!("{}", value);
    }

    let numbers: VecDeque<i32> = vec_iter_test.iter()
        .filter(|&&num| num % 2 != 0)
        .map(|&num| num * 2)
        .collect();

    println!("numbers elements:");
    for value in &numbers {
        println!("{}", value);
    }
}
```
