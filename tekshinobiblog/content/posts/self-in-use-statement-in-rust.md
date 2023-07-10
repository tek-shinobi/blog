---
title: "Self in Use Statement in Rust"
date: 2021-03-09T11:51:54+03:00
draft: false 
categories: ["rust"]
tags: ["rust"]
---

I was going through some library code in a rust crate and this pattern had my head scratching:
```rust
std::io::{self,Cursor};
```

What this is equivalent to is:
```rust
use std::io;
use std::io::Cursor;
```
which means explicitly use `std::io::Cursor` and also use `std::io` itself. Thus, Cursor need not be prefixed but you’d still need to do `io::BufReader` and so on.