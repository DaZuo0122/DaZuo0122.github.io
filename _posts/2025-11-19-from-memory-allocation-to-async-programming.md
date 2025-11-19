---
title: From Memory Allocation to Async Programming
categories: [Programming]
tags: [Rust]
---

> It's recommended to read [The Book](https://doc.rust-lang.org/stable/book/), if you don't have experience with C/C++. 
> 
> *Further reading*: [Rustonomicon](https://doc.rust-lang.org/nomicon/)

## Memory Allocation in Rust: The Foundation
Before diving into asynchronous programming complexities, we must first understand how Rust manages memory. Rust employs a dual approach to memory allocation, utilizing both the stack and the heap, with clear rules governing when each should be used.


### Stack Allocation
The stack is used for local variables with fixed sizes known at compile time. Stack allocation and deallocation are exceptionally fast—they simply involve moving a stack pointer. Each time a function is called, a new stack frame is created containing all its local variables, and when the function returns, this frame is discarded by adjusting the pointer.

Here's the example:

```rust
fn main() {
    let x: u32 = 5; // Stored on the stack
    let y: [i32; 3] = [1, 2, 3]; // Array with fixed size on stack
}
```

### Heap Allocation
The heap comes into play for data with sizes determined at runtime or data that needs to outlive the current function scope. Heap allocation involves more complex book-keeping and is consequently slower than stack allocation.

> The following uses `Box` and `Vec` as example, however, `String` is also a good choice (you may notice which makes handling strings in rust easy and convenient).

```rust
fn main() {
    let x: Box<u32> = Box::new(5); // Integer allocated on the heap
    let mut v: Vec<i32> = Vec::new(); // Vector with dynamic size on heap
}
```

### The Ownership
Rust's innovative ownership system eliminates the need for garbage collection while preventing common memory errors (not 100% perfect, we might discuss it in another blog post):

- Each value has a single owner

- Ownership can be transferred (moved) between variables

- When the owner goes out of scope, the value is dropped automatically

> Note: If you are familiar with *RAII* (Resource Acquisition Is Initialization), like in c++, you might feel comfortable with rust's lifetime.

For detailed explaination, please refer chapter 4 [What is Ownership](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html) in *The Book*.


## Synchronous Programming Paradigm
In "*traditional*" synchronous Rust, memory management follows predictable, linear patterns. Each function call establishes a clear hierarchy of stack frames, with values being allocated when variables come into scope and deallocated when they leave scope.

```rust
fn process_data() {
    let data = vec![1, 2, 3]; // Allocated when entering scope
    let result = transform(data); // Ownership moved
    println!("{:?}", result);
} // result is dropped here

fn transform(v: Vec<i32>) -> Vec<i32> {
    v.into_iter().map(|x| x * 2).collect()
} // v was moved, so nothing to drop
```

The synchronous model's beauty lies in its **straightforward relationship between code structure and memory behavior**. The compiler can statically determine each value's lifetime and generate appropriate drop calls. This linear execution flow, where each operation completes before the next begins, simplifies reasoning about memory.


## The Challenge Asynchronous Brings
Asynchronous programming introduces a fundamental shift: operations may suspend execution partway through and **resume later**, all within the same thread. This enables efficient handling of numerous concurrent I/O operations without the overhead of multiple threads.


### The Self-Referential Struct Problem
When Rust generates state machines for async functions, it often creates self-referential structs which containing references to their own fields. This occurs when you reference a local variable across an await point:

```rust
async fn process_file() {
    let data = vec![1, 2, 3]; // Local variable
    let data_ref = &data; // Reference to local variable
    read_from_network().await; // SUSPENSION POINT
    println!("{}", data_ref); // Using reference after await
}
```

The generated state machine must preserve the relationship between `data_ref` and `data` across the await point. If this state machine were moved in memory, `data_ref` would become a **dangling pointer** that still pointing to the original location of `data`, which has since moved.


### State Machines and Memory Stability
Rust compiles async functions into state machines that track progress through the function. Each await point represents a potential suspension/resumption boundary. For these state machines to work correctly, their memory locations must remain stable once they contain self-references.


*Async State Transitions*

```text
[Start State] → [Polled First Time] → [Suspended at Await] → [Resumed] → [Complete]
     ↓               ↓                     ↓                   ↓           ↓
Memory         Self-references        Must preserve      References    Clean up
allocated      established            memory layout      still valid    resources
```


## Pin: The Solution to Async Memory Challenges
[Pin](https://docs.rs/pin-project/latest/pin_project/) doesn't necessarily prevent all movement initially, it provides guarantees **once the value is pinned**. The key insight is that many types don't care about being moved (they implement `Unpin` automatically), while those that do care (like async state machines) can be protected.

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

struct SelfReferential {
    data: String,
    self_ref: *const String,
    _pin: PhantomPinned, // Opt-out of Unpin
}

impl SelfReferential {
    fn new(data: String) -> Pin<Box<Self>> {
        let mut res = Box::pin(Self {
            data,
            self_ref: std::ptr::null(),
            _pin: PhantomPinned,
        });
        
        // After pinning, establish self-reference
        let raw_ptr = &res.data as *const String;
        unsafe {
            let mut_ref = Pin::as_mut(&mut res);
            Pin::get_unchecked_mut(mut_ref).self_ref = raw_ptr;
        }
        res
    }
}
```

In this example, `SelfReferential` contains a pointer to its own field. By pinning it to memory, we guarantee that the relationship between `data` and `self_ref` remains valid for the duration of its lifetime.


### Pin in Async/Await
When you use async/await, the compiler automatically generates pinned state machines. The runtime system ensures these state machines are properly pinned before being polled:

```rust
// Under the hood, this async function generates a state machine
// that must be pinned for correct operation
async fn example_async() -> usize {
    let x = 42;
    let y = &x; // Creates self-reference in state machine
    some_async_operation().await;
    *y
}

// The runtime pins the future before polling
let future = example_async();
let pinned_future = Box::pin(future);
```

The `Pin` type ensures the compiler's generated state machine can safely hold self-references across await points without risk of invalidation due to moves.

## Memory Management in Sync and Async

### Similar Foundations, Different Challenges
Both paradigms build on the same ownership system, stack/heap allocation, and lifetime checking. However, they face different memory safety challenges:

- **Synchronous Rust** must prevent dangling pointers, double frees, and data races in a linear execution model

- **Asynchronous Rust** must maintain memory stability for self-referential structures across suspension points

### How Pin Bridges the Gap
Pin extends Rust's ownership system to handle the unique challenges of async programming without compromising safety:

1. **Zero-cost abstraction** - Pin doesn't add runtime overhead; it's a compile-time guarantee

2. **Gradual enforcement** - Types that don't need pinning (implementing Unpin) work transparently

3. **Safe APIs** - Properly constructed pinning APIs prevent misuse while allowing necessary operations

```rust
// For types that are Unpin (most types), Pin is transparent
let mut x = 5;
let pinned_x = Pin::new(&mut x);
// We can still use pinned_x normally because i32 is Unpin

// For !Unpin types, Pin restricts moving
let pinned_data = SelfReferential::new("hello".to_string());
// pinned_data cannot be moved, ensuring memory stability
```

### Send + Sync Requirements in Async
In async Rust, the combination of `Send` and `Sync` becomes crucial because async tasks might be scheduled across different threads by the runtime:

```rust
use tokio::task;
use std::sync::Arc;

// This works because all types are Send
async fn send_safe_async() -> i32 {
    let arc_data = Arc::new(42); // Arc<T> is Send + Sync
    let cloned = Arc::clone(&arc_data);
    
    // Spawn can send the task to another thread
    let handle = task::spawn(async move {
        *cloned
    });
    
    handle.await.unwrap()
}

// This WON'T work - Rc<T> is not Send
async fn non_send_async() {
    use std::rc::Rc;
    
    let rc_data = Rc::new(42); // Rc<T> is !Send
    
    // ERROR: future cannot be sent between threads safely
    // task::spawn(async move {
    //     println!("{}", rc_data);
    // });
}
```

## The Trinity: Send, Sync, and Pin in Async Rust

### Send + Pin: Thread-Safe Immovable Types
For async tasks to be scheduled across threads, they must be both `Send` (thread-transferable) and properly pinned:

```rust
use tokio::task;
use std::pin::Pin;

// A Send future that requires pinning
async fn complex_async() -> i32 {
    let local_data = vec![1, 2, 3];
    let data_ref = &local_data;
    
    // The generated future is self-referential but Send
    some_io_operation().await;
    
    data_ref.len() as i32
}

#[tokio::main(flavor = "multi_thread")]
async fn main() {
    let future = complex_async();
    let pinned_future = Box::pin(future);
    
    // This works because:
    // 1. The future is Send (no non-Send types)
    // 2. The future is properly pinned
    // 3. The runtime can move the Pin<Box<Future>> between threads
    let handle = task::spawn(pinned_future);
    let result = handle.await.unwrap();
    println!("Result: {}", result);
}
```

### Sync + Pin: Shared Immovable State
When you need to share pinned data between async tasks, it must be `Sync`:

```rust
use std::sync::Arc;
use std::pin::Pin;
use tokio::sync::Mutex;

struct SharedState {
    data: String,
    // ... potentially self-referential fields
}

async fn multiple_async_access() {
    let shared_state = Arc::new(Mutex::new(SharedState {
        data: "hello".to_string(),
    }));
    
    let mut handles = vec![];
    
    for i in 0..3 {
        let state_clone = Arc::clone(&shared_state);
        let handle = tokio::spawn(async move {
            let guard = state_clone.lock().await;
            println!("Task {}: {}", i, guard.data);
        });
        handles.push(handle);
    }
    
    for handle in handles {
        handle.await.unwrap();
    }
}
```

### Common Patterns and Solutions

```rust
use std::sync::Arc;
use tokio::sync::{Mutex, RwLock};

// Pattern 1: Shared mutable state
struct AppState {
    counter: Arc<Mutex<i32>>,
    config: Arc<RwLock<Config>>,
}

// Pattern 2: Thread-safe clones
async fn spawn_tasks() {
    let state = Arc::new(AppState {
        counter: Arc::new(Mutex::new(0)),
        config: Arc::new(RwLock::new(Config::default())),
    });
    
    for i in 0..10 {
        let state_clone = Arc::clone(&state);
        tokio::spawn(async move {
            process_task(i, state_clone).await;
        });
    }
}

// Pattern 3: Send bounds in generic functions
fn spawn_send_future<F, T>(future: F) -> tokio::task::JoinHandle<T>
where
    F: std::future::Future<Output = T> + Send + 'static,
    T: Send + 'static,
{
    tokio::spawn(future)
}
```



## Ending
From stack allocation to async pinning, Rust maintains a consistent philosophy: provide **zero-cost abstractions** that enforce safety at compile time. The journey from basic memory management to advanced async programming demonstrates how Rust tackles increasingly complex problems while preserving its core guarantees.


The ownership system that prevents memory errors in synchronous code evolves into the pinning system that prevents move errors in asynchronous code. This elegant progression means Rust developers can build on their fundamental knowledge to master advanced concepts that each layer resting securely on the verifiable foundations of the layer below.


As Rust continues to evolve, these core principles ensure that whether you're writing simple synchronous utilities or complex asynchronous systems, the compiler remains your vigilant partner in building safe, efficient software.