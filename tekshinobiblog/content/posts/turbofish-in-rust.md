---
title: "Turbofish in Rust"
date: 2021-03-10T10:50:37+03:00
draft: false 
categories: ["rust"]
tags: ["rust", "turbofish"]
---
Consider the following code (or run it in rust playground [here](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=7f0e8c951a3cd76a2622cf968ecded68)):

```rust
fn main() {
    let a:Vec = vec![1, 2, 3];

    let doubled = a.iter().map(|&x| x * 2).collect();

    assert_eq!(vec![2, 4, 6], doubled);
}
```


If you ran it in the playground linked above, you will see the following compiler error.
```terminal
error[E0282]: type annotations needed
 --> src/main.rs:4:9
  |
4 |     let doubled = a.iter().map(|&x| x * 2).collect();
  |         ^^^^^^^ consider giving `doubled` a type

error: aborting due to previous error

For more information about this error, try `rustc --explain E0282`.
error: could not compile `playground`

To learn more, run the command again with --verbose.
```

Because `collect()`` is so general, it can cause problems with type inference. As such, `collect()`` is one of the few times you'll see the syntax affectionately known as the **turbofish:** `::<>`. This helps the rust's inference algorithm understand specifically which collection you're trying to collect into.

In the above case, it is not clear if the collection is supposed to be a Vec, VecDeque, HashSet, HashMap or anything else that implements FromIterator.

There are two ways of fixing the above issue:

#### 1st Way: By annotating doubled:
```rust
let doubled:Vec<u32> = a.iter().map(|&amp;x| x * 2).collect();
// OR
let doubled:Vec<_> = a.iter().map(|&amp;x| x * 2).collect();
```

In both cases, the type annotation on doubled makes it clear what type of collection the `collect()` is going to collect into. The second option works because the rust's compiler can infer exactly what type of `Vec<_>` collection it is going to be because iter returns u32. So compiler can exactly infer that the partial type hint `_` in `Vec<_>` has to be `Vec<u32>`.

#### 2nd Way: turbofish:

```rust
let doubled = a.iter().map(|x| x * 2).collect::<Vec<i32>>();
```

Like in 1st way, because collect() only cares about what you're collecting into, you can still use a partial type hint, `_`, with the turbofish:

```rust
let doubled = a.iter().map(|x| x * 2).collect::<Vec<_>>();
```
### turbofish does not always work

There are times where you need to make type explicit via annotation. Turbofish won't work. This has to do with history. Turbofish was created to deal with some corner cases in generics. If you apply on functions that don't support generics, turbofish will not work.

Consider this code:

```rust
fn main() {
    let num = 15;
    let number = num.into();
}
```

Obviously, this gives an error as compiler does not know to what type do we want to convert num to, So, let's try turbofish.

```rust
fn main() {
    let num = 15;
    let number = num.into::<i64>();
}
```

No dice. We get a different error.

```terminal
error[E0107]: wrong number of type arguments: expected 0, found 1
 --> src/main.rs:3:29
  |
3 |     let number = num.into::<i64>();
  |                             ^^^ unexpected type argument

error: aborting due to previous error

For more information about this error, try `rustc --explain E0107`.
error: could not compile `playground`

To learn more, run the command again with --verbose.
```

The only way to resolve this is via annotation in type space. (LHS is type space and RHS is expression space).
```rust
let number :i64 = num.into();
```

This inconsistency is bit confusing and not very obvious in the beginning. This is because the of the way generics are implemented in into() function signature vs collect().
Here is the function signature for into():

```rust
fn into(self) -> T
```

and here is the function signature for collect():

```rust
fn collect<B>(self) -> B
```

it is that `<B>` in collect that makes it possible to use turbofish. Since into does not have this syntax, it will not work here.

Turbofish has its share of haters as it makes generics syntax inconsistent in some cases. There is a whole reddit thread on this topic [here](https://www.reddit.com/r/rust/comments/ld6q8u/bastion_of_the_turbofish_a_brief_tale_lamenting/) and the attempt to remove turbofish from rust that ended in absolute dismal failure.

