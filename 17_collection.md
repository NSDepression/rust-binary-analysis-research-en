# Collections

We investigated the memory layout and assembly characteristics of Vec type, VecDeque type, and HashMap type, aiming to clarify their structure and behavior.

## Investigation Results

### Vec Type Memory Layout

The memory layout of the Vec type is described below.

![collection](images/13-1.png)

### VecDeque Type Memory Layout

The memory layout of the VecDeque type is described below.

![collection](images/13-2.png)

### HashMap Type Memory Layout

The memory layout of the HashMap type is described below.

![collection](images/17-1.png)

### Management Structures

Types are managed by the following structures.
In debug builds, `capacity()` and `len()` refer to these structures.

* Vec type

```c
stack {
    offset + 0x00: Maximum number of elements,
    offset + 0x08: Address to heap,
    offset + 0x10: Current number of elements,
}
```

* VecDeque type

```c
stack {
    offset + 0x00: Maximum number of elements,
    offset + 0x08: Address to heap,
    offset + 0x10: Index of first data,
    offset + 0x18: Current number of elements,
}
```

* HashMap type

```c
stack {
    offset + 0x00: Address of bucket (data structure storing hash values generated from keys),
    offset + 0x08: Maximum number of elements,
    offset + 0x10: Number of free elements,
    offset + 0x18: Current number of elements,
    offset + 0x20: Random number,
}
```

## Details

The sample programs used for the investigation are described [later](#sample-programs-used).

### Vec Type

#### Type Creation

The Vec type is allocated in heap memory as follows:

![collection](images/17-2.png)

The following is the code that places data in the allocated heap.
At the same time, the management structure is also initialized.

![collection](images/17-3.png)

#### Element Addition

The following shows `push()` and the management structure and buffer before and after it.
In release builds, `push()` is inline-expanded due to optimization and cannot be confirmed as a function call, but data addition and Vec management structure updates are performed.

* Management structure and buffer before push

![collection](images/17-4.png)

* Management structure and buffer after push

![collection](images/17-5.png)

The following shows `insert()` and the management structure and buffer before and after it.
Similarly, in release builds, `insert()` is also inline-expanded due to optimization, but processing is performed to move data after the insertion position backward and insert new data.
However, due to optimization, the current number of elements in the management structure has not been updated. On the other hand, in debug builds, the management structure was also updated.

* Management structure and buffer before insert

![collection](images/17-6.png)

* Management structure and buffer after insert

![collection](images/17-7.png)

### VecDeque Type

#### Type Creation

The VecDeque type allocates a buffer as follows, then places data in that memory.
Finally, it constructs the management structure on the stack.

![collection](images/17-8.png)

#### Element Addition

The VecDeque type is designed as a circular buffer.
The following is an example of memory placement during data insertion using `push_front()` (see sample program).
In release builds, `push_front()` is inline-expanded due to optimization and function calls could not be confirmed, but data addition and VecDeque management structure updates are performed.

* Management structure and buffer before push_front

![collection](images/17-9.png)

* Management structure and buffer after push_front

![collection](images/17-10.png)

By executing `push_front()`, data is added to the end of existing data in the buffer.
This behavior is considered to be due to the circular buffer characteristics of VecDeque, and the management structure increments the current number of elements after data insertion.
Also, since the first data was 2 before data insertion, the index of the first data in the management structure is `0`.
After insertion, since 1 is added before 2 by `push_front()`, the first data becomes 1, and the index of the first data in the management structure becomes `5` as data is added to the end.

### HashMap Type

#### Type Creation

The following is the assembly showing the HashMap creation process by `HashMap::new()`.
Although the call to `HashMap::new()` cannot be confirmed due to optimization, processing to create the HashMap management structure can be confirmed.
A 16-byte random number is obtained using the Windows API `ProcessPrng()`.
This random number is handled in 8-byte chunks, and HashMap generates a hash value from the input key and manages it in association with the key and value.
This random number is used when generating a hash value from a key. Then, the HashMap management structure is constructed on the stack.
In Rust, since version `v1.35`, `SwissTable` is used as the HashMap algorithm, and `SipHash` is used as the hash function.

![collection](images/17-11.png)

#### Element Addition

The element addition for HashMap type, `insert()`, receives the management structure as the first argument, the key as the second argument, and the value as the third argument in assembly.
Then, value addition is performed in the following flow:

* 1. Hash Value Generation

The standard library function `core::hash::BuildHasher::hash_one()` generates a hash value based on the key and the random number in the management structure.
This hash value is used to generate an identifier associated with the key and value, and to generate the index where the identifier is placed in the bucket.

* 2. Identifier Generation and Duplication Check

After generating the hash value, identifier generation and duplication check are performed.
By right-shifting the generated hash value by `0x39` bits, the upper 7 bits are extracted.
This value is placed in the bucket as an identifier associated with the key and value.
As a characteristic of the identifier, since the upper 7 bits are extracted and that value is treated as a 1-byte number, the upper 1 bit of the identifier is always 0.
This property is used when determining whether an identifier already exists in the bucket.

Then, identifier duplication check is performed.
The logical AND of the hash value and the maximum number of elements is calculated, and using that result as an offset, a 16-byte value is obtained from the bucket.
Using the `pcmpeqb` instruction, it checks whether the identifier already exists in the 16-byte value obtained from the bucket.
The `pcmpeqb` instruction compares 16 bytes byte by byte, and if there is a matching byte sequence, it places `0xFF` at the same position in the first operand.
`0x00` is placed where there is no match.
If the first operand after instruction execution is filled with `0x00`, it can be said that the identifier is not duplicated.

![collection](images/17-12.png)

* 3. Index Generation

After generating the identifier, generate an index indicating where the identifier is placed in the bucket.
Using the `pmovmskb` instruction, generate a mask based on the 16-byte data obtained from the bucket in "2. Identifier Generation and Duplication Check".
The `pmovmskb` instruction extracts the most significant bit of each byte in the 16-byte register specified in the second operand and aggregates it as a bit string in the first operand.
Unused areas in the bucket are filled with `0xFF`, and the most significant bit is 1.
As explained in the previous section, the most significant bit of the identifier is always 0, so in the bit string aggregated by the `pmovmskb` instruction, the bit where the identifier is placed becomes 0, and the bit of the unplaced part becomes 1.
Next, execute the `tzcnt` instruction on this mask.
The `tzcnt` instruction counts the number of consecutive 0s from the lower bits of the value specified in the second operand until the first 1 appears, and stores the result in the first operand.
Through this processing, it is possible to calculate how many areas from the lower part of the bucket are already filled.
Furthermore, add the result of the logical AND of the hash value and the maximum number of elements used in the previous section to the count calculated by the `tzcnt` instruction.
By this addition, a candidate for the index where the identifier should be placed is obtained.
Finally, by taking the logical AND of this addition result and the maximum number of elements, adjust so that the index fits within the maximum number of elements.
This final value is used as the placement index for the identifier.

![collection](images/17-13.png)

* 4. Management Structure, Bucket, and Buffer Updates

Finally, update the management structure, bucket, and buffer (storage location for keys and values).
In the case of element addition, the number of elements in the management structure increases, the number of free elements decreases, etc.


## Sample Programs Used

* Vec type

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

* VecDeque type

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
    // vec_num.pop_back();

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

* HashMap type

```rust
use std::collections::HashMap;

fn main() {
    // Create HashMap
    let mut processes = HashMap::new();

    // Add elements
    processes.insert("ProcessA", 1001);
    processes.insert("ProcessB", 1002);
    processes.insert("ProcessC", 1003);

    // Get capacity and number of elements
    println!("processes length: {}, capacity: {}", processes.len(), processes.capacity());

    // Add data exceeding capacity
    processes.insert("ProcessD", 1004);

    println!("processes length: {}, capacity: {}", processes.len(), processes.capacity());

    // Get existing element
    let pid = processes.get("ProcessB");
    match pid {
        Some(&id) => println!("PID of ProcessA.exe: {}", id),
        None => println!("ProcessA.exe not found"),
    }

    // Get non-existent element
    let pid = processes.get("ProcessE");
    match pid {
        Some(&id) => println!("PID of ProcessD.exe: {}", id),
        None => println!("ProcessD.exe not found"),
    }

    // Remove existing data
    processes.remove("ProcessA");

    // Remove non-existent data
    processes.remove("ProcessE");
}
```
