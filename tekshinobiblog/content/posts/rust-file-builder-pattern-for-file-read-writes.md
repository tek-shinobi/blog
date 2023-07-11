---
title: "Rust File Builder Pattern for File Read Writes"
date: 2021-03-20T11:14:09+03:00
draft: false 
categories: ["rust"]
tags: ["rust"]
---

Consider the following test code that creates a file, writes to it and then reads back to confirm what it wrote.

```rust
#[cfg(test)]
mod tests {
    #[allow(unused_imports)]
    use super::*;
    use std::fs::{self, File, OpenOptions};
    use std::io::{Read, Seek, SeekFrom, Write};
    use tempfile::{tempdir, tempfile, NamedTempFile};

    #[test]
    fn test_file() {
        let dir = tempdir().unwrap();
        let file_path = dir.path().join("foo.txt");
        let mut file = File::create(file_path.to_owned()).unwrap();
        file.write_all("Bar Baz".as_bytes())
           .unwrap();
        file.seek(SeekFrom::Start(0)).unwrap();
        let mut contents = String::new();
        file.read_to_string(&mut contents).unwrap();
        assert_eq!(contents, "Bar Baz");
        drop(file);
        dir.close().unwrap();
     }
}
```
This test fails with a very cryptic and unhelpful message:
```terminal
thread 'tests::test_tempfiles' panicked at 'called `Result::unwrap()` 
on an `Err` value: Os { code: 9, kind: Other, message: "Bad file descriptor" }'
```

It will panic at this line 
```rust
file.read_to_string(&mut contents).unwrap();
```

What is happening is that `File::create` and `File::open` in rust are actually syntactic sugar for File Builder pattern. Here the file is being created and when the first write happens, the file mode become write only. If you tried to read from this file-handle, it would panic. It would have been helpful if the error message was less cryptic.

Moving on to file builder pattern, `OpenOptions`. Actually its just the builder pattern with an another unhelpful name. If it was named `FileBuilder`, it would have been super helpful. But for some reason, it is cryptically named `OpenOption`. Further rant. You would then proceed to `std::fs::File` documentation but that does not help either. If you were familiar with `OpenOptions`, then good, if not, you are in for some head scratching. In my opinion, using `OpenOption` should have been detailed first and then its syntactic sugars `File::open` and `File::create`.

If you want to achieve file create, then file write and then file read, you need to give the file-handle permissions for both read and write. This is how you will do it:
```rust
#[cfg(test)]
mod tests {
    #[allow(unused_imports)]
    use super::*;
    use std::fs::{self, File, OpenOptions};
    use std::io::{Read, Seek, SeekFrom, Write};
    use tempfile::{tempdir, tempfile, NamedTempFile};

    #[test]
    fn test_file() {
        let mut file = OpenOptions::new()
            .read(true)
            .write(true)
            .create(true)
            .open("foo.txt")
            .unwrap();
        file.write_all(b"Hello, world!").unwrap();
        file.seek(SeekFrom::Start(0)).unwrap();
        let mut contents = String::new();
        file.read_to_string(&mut contents).unwrap();
        assert_eq!(contents, "Hello, world!");
    }
}
```