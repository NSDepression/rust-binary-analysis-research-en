# Smart Pointers

We investigated assembly characteristics and memory layout of Rust smart pointers: Box type, Rc type, Cell type, and RefCell type.

## Investigation Results

The structure and specifications of each type are explained below.

## Details

### Box<T>

#### 1. Box<T> Creation

Heap memory is allocated with `__rust_alloc()`. (Ultimately, memory is allocated with the Windows API `HeapAlloc()`.)

#### 2. Memory Initialization

When `Box<T>` is defined in a chain structure, the heap is initialized as follows.
In Rust, memory is placed by default in descending order of size.

```c
Heap memory for variable first {
    Offset+0x00: next: second address
    Offset+0x08: value: 0x00000001
}
Heap memory for variable second {
    Offset+0x00: next: third address
    Offset+0x08: value: 0x00000002
}
Heap memory for variable third {
    Offset+0x00: next: 0x00000000
    Offset+0x08: value: 0x00000003
}
```

#### 3. Automatic Deallocation

`Box<T>` implements automatic deallocation.
The following is the internal processing of automatic deallocation.
`sub_7FF7AFC41030()` is a recursive function.
This allows memory in a chain structure to be deallocated.

![smart_pointer](images/19-1.png)

### Rc<T>

#### 1. Memory Initialization Rc::new()

`Rc<T>` memory initialization is inline-expanded.
Memory is allocated on the heap, and further `Rc<T>` management structure is allocated on the heap.

![smart_pointer](images/19-2.png)

The `Rc<T>` management structure is as follows.
Similar management structures are created even in minimized binaries.
32-bit binaries are similar except for handling addresses as 4 bytes.

```c
Rc<T> management structure {
    0x00: Strong reference count
    0x08: Weak reference count
    0x10: Concrete type specified in type parameter
}
```

### Cell<T>

In release builds and minimized binaries, the relevant processing is removed due to optimization.
Also, even in debug build binaries, there is no characteristic processing, so it cannot be identified whether the Cell type is used.

### RefCell<T>

#### 1. Memory Initialization RefCell<T>

The management structure of RefCell is as follows:

```c
RefCell<T> management structure {
    0x00: Ref Counter
    0x08: Concrete type specified in type parameter
}
```

RefCell has the concepts of borrowing and mutable borrowing, where borrowing is possible from multiple variables, but mutable borrowing can only be done by one variable.
When calling the `borrow()` function that performs borrowing, the `Ref Counter` is incremented.
Also, when calling `borrow_mut()` that performs mutable borrowing, the `Ref Counter` becomes `-1`.
Therefore, a situation where `Ref Counter` is less than 0 indicates mutable borrowing is in progress, and performing borrowing or mutable borrowing at this time causes a runtime error.

![smart_pointer](images/19-3.png)

## Sample Programs Used

* Box type sample program

```rust
struct Node {
    value: i32,
    next: Option<Box<Node>>,
}

impl Node {
    fn new(value: i32) -> Self {
        Node { value, next: None }
    }
}

fn main() {
    let mut first = Box::new(Node::new(1));
    first.next = Some(Box::new(Node::new(2)));

    if let Some(ref mut next_node) = first.next {
        next_node.next = Some(Box::new(Node::new(3)));
    }

    println!("First value: {}", first.value);
    if let Some(ref next_node) = first.next {
        println!("Next value: {}", next_node.value);
        if let Some(ref next_next_node) = next_node.next {
            println!("Next next value: {}", next_next_node.value);
        }
    }
}
```

* Rc type sample program

```rust
use std::rc::{Rc, Weak};

fn main() {
    // Create shared data using Rc
    let data = Rc::new("Hello, Rc!".to_string());

    // Check Rc reference count
    println!("Initial reference strong count: {}", Rc::strong_count(&data));
    println!("Initial reference weak count: {}", Rc::weak_count(&data));

    {
        // Weak reference increases
        let weak_ref: Weak<String> = Rc::downgrade(&data);
        println!("Reference strong count after downgrade: {}", Rc::strong_count(&data));
        println!("Reference weak count after downgrade: {}", Rc::weak_count(&data));
    }

    // Clone data to create a new Rc (strong reference count increases)
    let data_clone1 = Rc::clone(&data);
    println!("Reference strong count after clone1: {}", Rc::strong_count(&data));
    println!("Reference weak count after clone1: {}", Rc::weak_count(&data));

    // Another clone (strong reference count increases)
    let data_clone2 = Rc::clone(&data);
    println!("Reference strong count after clone2: {}", Rc::strong_count(&data));
    println!("Reference weak count after clone2: {}", Rc::weak_count(&data));

    // Each variable references the data
    println!("Original data: {}", data);
    println!("Clone 1 data: {}", data_clone1);
    println!("Clone 2 data: {}", data_clone2);

    // Reference count decreases when data_clone1 and data_clone2 go out of scope
    drop(data_clone1);
    println!("Reference strong count after dropping clone1: {}", Rc::strong_count(&data));
    println!("Reference weak count after dropping clone1: {}", Rc::weak_count(&data));

    drop(data_clone2);
    println!("Reference strong count after dropping clone2: {}", Rc::strong_count(&data));
    println!("Reference weak count after dropping clone2: {}", Rc::weak_count(&data));

    // Finally, memory is released when data goes out of scope
    drop(data);
    println!("Dropped the original data.");
}
```

* Cell type sample program

```rust
use std::cell::Cell;

pub struct Immutable {
    a: i32,
    b: Cell<i32>,
}

fn main() {
    let x = Immutable { a: 10, b: Cell::new(5) };
    x.b.set(x.b.get() + 10);

    println!("Immutable.a = {}", x.a);
    println!("Immutable.b = {}", x.b.get());
}
```

* RefCell type sample program

```rust
use std::cell::RefCell;

struct MyStruct {
    value: i32,
    name: String,
    description: String,
}

impl MyStruct {
    fn new(value: i32, name: &str, description: &str) -> Self {
        MyStruct {
            value,
            name: name.to_string(),
            description: description.to_string(),
        }
    }
}

fn main() {
    let my_struct = RefCell::new(MyStruct::new(5, "Example", "This is a sample struct."));

    // Borrow value to read
    {
        let borrowed_struct = my_struct.borrow();
        println!("Borrowed value: {}", borrowed_struct.value); // Displays 5
        println!("Name: {}", borrowed_struct.name); // Displays "Example"
        println!("Description: {}", borrowed_struct.description); // Displays "This is a sample struct."
    } // borrowed_struct scope ends and borrow is released

    // Perform mutable borrow to change value
    {
        let mut borrowed_mut_struct = my_struct.borrow_mut();
        borrowed_mut_struct.value += 10; // Add 10 to 5
        borrowed_mut_struct.name = "Updated Example".to_string(); // Update name
        borrowed_mut_struct.description = "This struct has been updated.".to_string(); // Update description
    } // mutable borrow ends here

    // Borrow value again to read
    {
        let updated_borrowed_struct = my_struct.borrow();
        println!("Updated value: {}", updated_borrowed_struct.value); // Displays 15
        println!("Updated Name: {}", updated_borrowed_struct.name); // Displays "Updated Example"
        println!("Updated Description: {}", updated_borrowed_struct.description); // Displays "This struct has been updated."
    }
}
```
