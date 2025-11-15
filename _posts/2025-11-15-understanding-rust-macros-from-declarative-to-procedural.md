---
title: Understanding Rust Macros - From Declarative to Procedural
categories: [Programming]
tags: [Rust]
---

## Before we start
Rust provides powerful tools for metaprogramming, this blog aims to make rust beginners be fearless to them ("magical" macros).

Recommended reading (If you familiar with Cpp): [Code Generation in Rust vs C++26](https://brevzin.github.io/c++/2024/09/30/annotations/)

## Part 1: Two Approaches to Metaprogramming

### Declarative Macros vs Classic C Macros
Rust's declarative macros and classic C macros both serve the purpose of metaprogramming, but they operate in fundamentally different ways with significant implications for safety and maintainability.


**Classic C Macros: Text Substitution**


C macros use simple text substitution through a preprocessor. The preprocessor scans the source code before compilation and replaces macro invocations with their definitions.

```c
#include <stdio.h>

#define SQUARE(x) x * x
#define MAX(a, b) ((a) > (b) ? (a) : (b))

int main() {
    int result1 = SQUARE(5);      // Becomes: 5 * 5
    int result2 = SQUARE(2 + 3);  // Becomes: 2 + 3 * 2 + 3 (unexpected!)
    int max_val = MAX(10, 20);    // Becomes: ((10) > (20) ? (10) : (20))
    
    printf("Square of 5: %d\n", result1);        // 25
    printf("Square of 2+3: %d\n", result2);      // 11, not 25!
    printf("Max of 10 and 20: %d\n", max_val);   // 20
    
    return 0;
}
```

The critical issue with C macros is their lack of understanding of code structure. The `SQUARE(2 + 3)` example demonstrates how operator precedence can lead to unexpected results because the macro expands to `2 + 3 * 2 + 3` instead of the intended `(2 + 3) * (2 + 3)`.


**Rust Declarative Macros: Structured Pattern Matching**


Rust's declarative macros operate on the abstract syntax tree (AST) rather than raw text, providing syntax awareness and hygiene.

```rust
macro_rules! square {
    ($x:expr) => {
        $x * $x
    };
}

macro_rules! max {
    ($a:expr, $b:expr) => {
        if $a > $b { $a } else { $b }
    };
}

fn main() {
    let result1 = square!(5);        // Expands to: 5 * 5
    let result2 = square!(2 + 3);    // Expands to: (2 + 3) * (2 + 3)
    let max_val = max!(10, 20);      // Expands to: if 10 > 20 { 10 } else { 20 }
    
    println!("Square of 5: {}", result1);        // 25
    println!("Square of 2+3: {}", result2);      // 25
    println!("Max of 10 and 20: {}", max_val);   // 20
}
```

Key advantages of Rust's approach:

- **Syntax awareness**: Macros understand expressions, statements, and other language constructs

- **Hygiene**: Macros cannot accidentally capture or interfere with external identifiers

- **Type safety**: Better integration with Rust's type system

- **Pattern matching**: Complex matching patterns for different use cases


## Part 2: Procedural Macros and Serde in Depth

### Understanding the Magic Behind Serde's Derive Macros
Procedural macros (aka proc-macro) take metaprogramming to the next level by allowing Rust code to generate Rust code at compile time. The `serde` crate provides an excellent case study of proc-macros in action.

As other crates used, we may want to create a new project for convenience. And the tool ["cargo-expand"](https://github.com/dtolnay/cargo-expand) will be used to get the result of macro expansion and `#[derive]` expansion applied to the example project.

The dependencies shown as below:

```Cargo.toml
[package]
name = "foo"
version = "0.1.0"
edition = "2024"

[dependencies]
serde = { version = "1.0", features = ["derive"]}
serde_json = "1.0"
```


**Serde Example: Original Code**

```rust
// main.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Person {
    age: u32,
    first_name: String,
    last_name: String,
}

fn main() {
    println!("Hello, world!");
}
```


### The Expanded Code
When we expand this code using `cargo expand`, we can see exactly what the proc-macros generate (it will directly printed to terminal).


### 1. Basic Struct and Debug Implementation
The original struct remains unchanged, and the `Debug` implementation is straightforward:

```rust
struct Person {
    age: u32,
    first_name: String,
    last_name: String,
}

impl ::core::fmt::Debug for Person {
    fn fmt(&self, f: &mut ::core::fmt::Formatter) -> ::core::fmt::Result {
        ::core::fmt::Formatter::debug_struct_field3_finish(
            f,
            "Person",
            "age", &self.age,
            "first_name", &self.first_name,
            "last_name", &self.last_name,
        )
    }
}
```

The `Debug` implementation uses a helper function that knows we have exactly three fields. This is efficient but specific to our exact field count.


### 2. Serialize Implementation
The `Serialize` implementation is more interesting. Notice it's wrapped in a `const _: () = { ... }` block:

```rust
const _: () = {
    // ... imports and implementation
    impl _serde::Serialize for Person {
        fn serialize<__S>(&self, __serializer: __S) -> _serde::__private228::Result<__S::Ok, __S::Error>
        where
            __S: _serde::Serializer,
        {
            let mut __serde_state = _serde::Serializer::serialize_struct(
                __serializer, "Person", 3
            )?;
            _serde::ser::SerializeStruct::serialize_field(&mut __serde_state, "age", &self.age)?;
            _serde::ser::SerializeStruct::serialize_field(&mut __serde_state, "first_name", &self.first_name)?;
            _serde::ser::SerializeStruct::serialize_field(&mut __serde_state, "last_name", &self.last_name)?;
            _serde::ser::SerializeStruct::end(__serde_state)
        }
    }
};
```

Key points:

- The const _: () block creates a scope that prevents generated names from leaking into the global namespace

- The macro generates a complete Serialize trait implementation

- It uses a state machine pattern: start struct → serialize each field → end struct

- Each field is serialized by name and reference


### 3. Deserialize Implementation - The Complex Part
The `Deserialize` implementation is significantly more complex because it needs to handle multiple input formats and error conditions.


**Field Identification**

First, the macro generates an enum to represent possible fields:

```rust
enum __Field {
    __field0,  // age
    __field1,  // first_name  
    __field2,  // last_name
    __ignore,  // unknown fields
}
```

This enum is used to track which field we're currently processing.


**Field Visitor**

The macro creates a `__FieldVisitor` that knows how to convert different input types into our field enum:

```rust
impl<'de> _serde::de::Visitor<'de> for __FieldVisitor {
    type Value = __Field;
    
    fn visit_u64<__E>(self, __value: u64) -> _serde::__private228::Result<Self::Value, __E> {
        match __value {
            0u64 => _serde::__private228::Ok(__Field::__field0),  // age
            1u64 => _serde::__private228::Ok(__Field::__field1),  // first_name
            2u64 => _serde::__private228::Ok(__Field::__field2),  // last_name
            _ => _serde::__private228::Ok(__Field::__ignore),     // ignore others
        }
    }
    
    fn visit_str<__E>(self, __value: &str) -> _serde::__private228::Result<Self::Value, __E> {
        match __value {
            "age" => _serde::__private228::Ok(__Field::__field0),
            "first_name" => _serde::__private228::Ok(__Field::__field1),
            "last_name" => _serde::__private228::Ok(__Field::__field2),
            _ => _serde::__private228::Ok(__Field::__ignore),
        }
    }
}
```

This visitor handles both numeric indices (for sequence formats) and string names (for map formats).


**Main Visitor and PhantomData**

Here's where we encounter `PhantomData`:

```rust
struct __Visitor<'de> {
    marker: _serde::__private228::PhantomData<Person>,
    lifetime: _serde::__private228::PhantomData<&'de ()>,
}
```


**Understanding PhantomData**

[PhantomData](https://doc.rust-lang.org/std/marker/struct.PhantomData.html) is a zero-sized type (it takes no memory) that tells the compiler to treat a struct as if it contains certain types or lifetimes, even when it doesn't actually store them.


In our generated code:

- `marker`: `PhantomData<Person>` tells the compiler that this visitor is logically "owning" a `Person` type, even though no `Person` is actually stored. This is necessary because the visitor's associated type is `type Value = Person`.

- `lifetime`: `PhantomData<&'de ()>` indicates that this struct depends on the lifetime `'de`, even though it doesn't actually contain any references with that lifetime. This ensures the visitor is treated correctly with respect to the deserialization lifetime.


**Why PhantomData**:

The generated Visitor types are generic over the value type (e.g., `Person`) and a lifetime `'de` but don't actually store a `Person` or `&'de Person`. `PhantomData` and `PhantomData<&'de ()>` express those phantom associations so the compiler:

- Enforces the correct lifetime bounds (prevents the Visitor from outliving `'de`).

- Gives the Visitor the same variance as if it contained those types.

- Affects auto-trait impls (so the Visitor is `!Send/!Sync` if `Person` or `&'de ()` would be).

- Avoids unused-generic-parameter errors.

In short, `PhantomData` solves both issues by creating a "phantom" usage that satisfies the compiler's ownership and lifetime rules.


**Sequence Deserialization**

For array-like input:

```rust
fn visit_seq<__A>(self, mut __seq: __A) -> _serde::__private228::Result<Self::Value, __A::Error>
where
    __A: _serde::de::SeqAccess<'de>,
{
    let __field0 = match _serde::de::SeqAccess::next_element::<u32>(&mut __seq)? {
        Some(__value) => __value,
        None => return Err(_serde::de::Error::invalid_length(0, &"struct Person with 3 elements")),
    };
    // ... repeats for first_name and last_name
    Ok(Person { age: __field0, first_name: __field1, last_name: __field2 })
}
```

This handles formats like JSON arrays: `[30, "John", "Doe"]` where fields are in declaration order.


**Map Deserialization**

For object-like input:

```rust
fn visit_map<__A>(self, mut __map: __A) -> _serde::__private228::Result<Self::Value, __A::Error>
where
    __A: _serde::de::MapAccess<'de>,
{
    let mut __field0: Option<u32> = None;
    let mut __field1: Option<String> = None;
    let mut __field2: Option<String> = None;
    
    while let Some(__key) = _serde::de::MapAccess::next_key::<__Field>(&mut __map)? {
        match __key {
            __Field::__field0 => {
                if Option::is_some(&__field0) {
                    return Err(Error::duplicate_field("age"));
                }
                __field0 = Some(_serde::de::MapAccess::next_value::<u32>(&mut __map)?);
            }
            // ... similar for other fields
        }
    }
    
    let __field0 = __field0.ok_or_else(|| _serde::de::Error::missing_field("age"))?;
    // ... similar for other fields
    
    Ok(Person { age: __field0, first_name: __field1, last_name: __field2 })
}
```

This handles formats like JSON objects: `{"age": 30, "first_name": "John", "last_name": "Doe"}` and includes:

- Duplicate field detection

- Missing field validation

- Unknown field ignoring


### How the Macro Process Works
1. **Token Stream Input**: When you add `#[derive(Serialize, Deserialize)]`, the compiler passes your struct's tokens to Serde's procedural macro.

2. **AST Parsing**: Serde uses the `syn` crate to parse the token stream into a structured Abstract Syntax Tree (AST).

3. **Analysis**: The macro examines your struct's name, fields, field types, and any Serde attributes.

4. **Code Generation**: Using the `quote` crate, the macro generates the implementations we examined above.

5. **Token Stream Output**: The generated code is returned as a new token stream that gets compiled alongside your code.


### The Power of Procedural Macros
This example demonstrates why procedural macros are so powerful:

- Boilerplate Elimination: What would be hundreds of lines of error-prone code is automatically generated

- Type Safety: The generated code is completely type-safe and follows Rust's ownership rules

- Performance: The generated code is as efficient as hand-written implementations

- Flexibility: Through attributes, you can customize behavior without changing the macro itself


The complexity we see in the expanded code is exactly why proc-macros are valuable, they handle this complexity once in the macro definition, and then everyone benefits from robust, efficient implementations without having to write them manually.


### This is not the End
Understanding this expansion process and concepts like `PhantomData` gives you insight into how Rust's trait system and metaprogramming capabilities work together to create powerful, safe abstractions.



But this is only the beginning.
