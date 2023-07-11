---
title: "Rust Gotchas Copy and Clone"
date: 2021-05-29T22:21:11+03:00
draft: false 
categories: ["rust"]
tags: ["rust"]
---
Rust has two central concepts: Borrowing and ownership.

In very simple terms, Borrowing conceptually means access to the data through reference, be it mutable or immutable. This is implemented via ***&*** and ***&mut***. For a more exhastive treatment, please refer the docs.

Again, in very simple terms, Ownership conceptually means accessing without ***&*** (i.e. without using a reference). Of course, Ownership is a science in Rust and please refer to docs for in-depth treatment.

When it comes to dealing with ownership issues, Rust offers options like Copy and Clone, among others.

Since Copy and Clone are so common, and particularly intuitive but confusing to people coming from other languages, let me introduce a v simple fact about both. Copy is implicit and Clone is explicit. Please note this, as this can turn into a potential landmine when it comes to performance.

Implicit means not explicit. Like here:
```rust
fn show_copy(arg1: i64){} 
fn main(){ 
    let a: i64 = 1; 
    show_copy(a); // implicit copy happens here 
}
```
Copy, when supported by a type, happens in any place where a transfer of ownership would have happened. Like in the above comments. Since ***i64*** implements Copy, this happens implicitly.

Lets make this more clear:
```rust
#[derive(Copy, Clone)] 
struct Id{ 
    data: i64, 
} 

fn show_copy(arg1: Id){} 

fn main(){ 
    let a: Id = Id {data: 1}; 
    show_copy(a); // implicit copy happens here 
}
```
As you can see here, if you are not paying attention, you can completely miss the fact that an actual copy of  ***Id*** happened, each time its ownership was supposed to have moved across function boundary. This is why Copy is not good for general usage. ***Use Clone. It does the same thing but in an explicit way, making it easier to know the intent.***
```rust
#[derive(Clone)] 
struct Id{ 
    data: i64, 
} 
fn show_copy(arg1: Id){} 
fn main(){ 
    let a: Id = Id {data: 1}; 
    show_copy(a.clone()); // Explicit. Intent to create a copy is explicit here 
```
❗Important point here.
```rust
// this is valid. 
#[derive(Clone)]
struct Id{
  data: i64,
}

// but this is not valid
#[derive(Copy)]
struct Id{
  data: i64,
}
// because Copy requires implementation of Clone.
```
