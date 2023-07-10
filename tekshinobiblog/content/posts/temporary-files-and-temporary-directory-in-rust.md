---
title: "Temporary Files and Temporary Directory in Rust"
date: 2021-03-20T11:36:47+03:00
draft: false 
categories: ["rust"]
tags: ["rust","file", "temporary"]
---
I was looking for Python style temporary files and temporary directories in Rust. Very handy when writing tests for simulating actual files, or when we need to use files once or cache large amount of data for short time. We then let the operating system delete the files at a later time automatically or by its clean-up feature manually.

### Doing ourselves using Operating System Temporary Directory

Let's first implement this by ourselves to understand what is going on.

To benefit from the auto-deletion, we must create temporary files in directories the operating system looks for files to delete. Hence, we need to get the temp directory. In Rust, we can use the `temp_dir( std::env::temp_dir)` function to that end.

In Rust, temporary files and directory handling is in a crate called `tempfiles`.
```rust
use std::env::temp_dir;
use std::io::Result;

fn main() {
    let dir = temp_dir();
    println!("{}", dir.to_str().unwrap());
}
```

Running the above code on a Ubuntu machine returns `/tmp`. If there is a directory specified for the **TMPDIR** environment, it will be returned, otherwise `/tmp` is returned. For Android operating system, the `temp_dir` function returns `/data/local/tmp`.

For Windows, the function works a little bit differently. The function first checks the **TMP** environment for value. If **TMP** is empty, it then checks the **TEMP** environment. When **TEMP** is empty or undefined, the function then checks the **USERPROFILE** environment. If **USERPROFILE** is still undefined, the function returns the Windows directory.

### Create Temporary Files in Rust

Temporary files typically have unique names to ensure that our codes refer to a file it had created and used at a particular instance. We could use UUIDs as file names.

In the Cargo.toml, update the dependencies as follows:

```yaml
[package]
edition = "2018"

[dependencies]
uuid = { version = "0.8.2", features = ["serde", "v4"] }
```
And the main.rs file:
```rust
use std::env::temp_dir;
use std::io::Result;
use uuid::Uuid;
use std::fs::File;
 
fn main(){
 
    let mut dir = temp_dir();
    println!("{}", dir.to_str().unwrap());
 
    let file_name = format!("{}.txt", Uuid::new_v4());
    println!("{}", file_name);
    dir.push(file_name);
 
    let file = File::create(dir)?;
}
```
But this is not convenient. In the presence of pathological temporary file cleaner, relying on file paths is unsafe because a temporary file cleaner could delete the temporary file which an attacker could then replace. Also, we might run into issues if we want to have multiple independent references to the same temporary files (useful in consumer/producer patterns).

### Using the tempfile crate

Crate tempfile is used to securely create and manage temporary files. Temporary files created by this create are automatically deleted. In addition to creating temporary files, this library also allows users to securely open multiple independent references to the same temporary file (useful for consumer/producer patterns and surprisingly difficult to implement securely).

This crate provides two temporary file variants: a tempfile() function that returns standard File objects and NamedTempFile. When choosing between the variants, prefer tempfile() unless you either need to know the file's path or to be able to persist it.

tempfile() doesn't rely on file paths so above mentioned vulnerability isn't an issue. However, NamedTempFile does rely on file paths.

Also, tempfile() will (almost) never fail to cleanup temporary files but NamedTempFile will if its destructor doesn't run. This is because tempfile() relies on the OS to cleanup the underlying file so the file while NamedTempFile relies on its destructor to do so.

Minimum required Rust version: 1.40.0

Add this to your Cargo.toml:
```yaml
[package]
edition = "2018"

[dependencies]
tempfile = "3.2.0"
```

And the main.rs file:
```rust
use std::fs::File;
use std::io::{Write, Read, Seek, SeekFrom};
use tempfile::tempfile;
use tempfile::tempdir;

fn main() {
    // Write
    let mut tmpfile: File = tempfile().unwrap();
    write!(tmpfile, "Hello World!").unwrap();

    // Seek to start
    tmpfile.seek(SeekFrom::Start(0)).unwrap();

    // Read
    let mut buf = String::new();
    tmpfile.read_to_string(&mut buf).unwrap();
    assert_eq!("Hello World!", buf);

}
```
We will not be able to see the files and directories when the application completes execution. The operating system deletes them when the handles close.

The structs NamedTempFile and TempDir work the same way but require explicit deletion via their destructors.
```rust
use tempfile::{NamedTempFile, TempDir};
use std::io::Result;
use std::io::Write;
 
fn main() -> Result<()> {
 
    // Create a temporary file inside of the directory returned by `std::env::temp_dir()`.
    let mut temp_file = NamedTempFile::new()?;
 
    write!(temp_file, "This is a temp file")?;
    println!("{:?}", temp_file);
 
 
    // Create a temporary dir inside of the directory returned by `std::env::temp_dir()`.
    let temp_dir = tempdir()?;
    println!("{:?}", temp_dir);
 
    temp_file.close()?;
    temp_dir.close()?;
 
    Ok(())
}
```

