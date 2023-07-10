---
title: "Rust Tips: Box Rc Arc Cell Refcell Mutex"
date: 2021-03-08T11:58:32+03:00
draft: false 
---
Difference Between Box, Rc, Arc, Cell (and its various variants.. RefCell, Mutex, RwLock) in Rust:

* Box is for single ownership. A great use case is to use this when we want to store primitive types (stored on stack) on heap.
* Rc is for multiple ownership.
* Arc is for multiple ownership, but threadsafe.
* RefCell is for “interior mutability”; that is, when you need to mutate something behind a &T.
* Cell is for “interior mutability” for Copy types; that is, when you need to mutate something behind a &T. Cell, is similar to RefCell except that instead of giving references to the inner value, the value is copied in and out of the Cell.
* Mutex, which offers interior mutability that’s safe to use across threads

A `Copy` type is one where all the data it logically encompasses (usually, owns) is part of its stack representation. For example `int/floats` are copy types. `Vec` and `String` are not copy types. Surprisingly, `&T` is a copy type. Below is a link post that explains this very well. It can be very confusing at first but once you get a hang of the ownership, borrowing and borrow-checking rules, it all makes sense.

Cell and its variants can be confusing. Here is an excellent post. https://manishearth.github.io/blog/2015/05/27/wrapper-types-in-rust-choosing-your-guarantees/

Here is another link post explaining the single threaded Read-Write Lock pattern implemented in Rust that makes mutability of referenced objects different from other languages. https://manishearth.github.io/blog/2015/05/17/the-problem-with-shared-mutability/

Another great link discussing interior mutability concepts is: https://ricardomartins.cc/2016/06/08/interior-mutability

Since the above are often used in conjunction with threads, here is another post that also explains how thread safety is dealt with in Rust: https://manishearth.github.io/blog/2015/05/30/how-rust-achieves-thread-safety/

A great article on Raw Pointers which are central to understanding the inner implementation of "inner mutability". http://web.mit.edu/rust-lang_v1.25/arch/amd64_ubuntu1404/share/doc/rust/html/book/first-edition/raw-pointers.html

