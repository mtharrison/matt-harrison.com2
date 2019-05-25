---
title: "Building and using a sys-crate with Rust - let's make a node clone (well kind of...)"
date: 2018-08-22T00:00:00+01:00
---
Rust is an awesome language and platform to use, however there's so much great software already written in c/c++. Luckily it's not too complicated to make use of c/c++ projects in Rust. In this short post I'll show you how.

From a high-level perspective you can take any c/c++ project, for this example I'm going to use [Duktape](https://duktape.org/), the lightweight embeddable JavaScript engine. I'm choosing Duktape because it's _very_ simple to build it - it's just 1 `.c` file. Hopefully this will make the steps easy to understand and we won't get too bogged down in build system issues.

The steps are:

1. Create a *-sys create which has your native dependency included along with knowledge of how to build it and exposes a preferably safe API to consumers. We'll be making `duktape-sys` crate here.
2. Make a crate that consumes the `-sys` crate and uses it to build something cool. Here we'll be building `duk` which is just a binary which you can pass a JavaScript file too and it will print out the last number on the stack

### Creating duktape-sys crate

First we'll create a new library crate using cargo:

```
$ cargo new --lib duktape2-sys
```

We'll download and bundle duktape source code from [https://duktape.org/](https://duktape.org/). I've added this to a `vendor` directory:


```
.
├── Cargo.lock
├── Cargo.toml
├── build.rs
├── src
│   └── lib.rs
├── target
│   ├── ...
└── vendor
    └── duktape-2.3.0
        └── ...
```

We'll need some dependencies:

```toml
// cargo.toml

[package]
name = "duktape2-sys"
version = "0.1.0"
authors = ["Matt Harrison <hi@matt-harrison.com>"]

[dependencies]
libc = "0.2.0"

[build-dependencies]
cc = "1.0"
```

We'll use the libc crate here to use the `c_void` type, we'll pass this around in Rust as a opaque pointer to some memory allocated by C. We'll use the cc crate because it provides us with a crossplatform way to build libduktape from source. Speaking of building duktape, let's create a build.rs file:

```rust
// build.rs

extern crate cc;

fn main() {
    cc::Build::new()
        .file("vendor/duktape-2.3.0/src/duktape.c")
        .compile("duktape");
}
```
What this magical looking script will do is: when building our crate it will compile  `vendor/duktape-2.3.0/src/duktape.c` into a static library called `libduktape.a` inside our `target` directory and also add a linker library search flag to the rust compiler so that it knows how to link the native library. That's all we need to handle building and linking with the native code!

Let's go and implement a Rust API that we can use in other crates:

```rust
// Crates

extern crate libc;

// Use statements

use std::error::Error;
use std::ffi::CString;
use std::fmt;
use std::os::raw::c_char;
use std::ptr;

// Aliases

use libc::c_void as void;

// Duktape constants

const DUK_COMPILE_EVAL: u32 = 1 << 3;
const DUK_COMPILE_SAFE: u32 = 1 << 7;
const DUK_COMPILE_NOSOURCE: u32 = 1 << 9;
const DUK_COMPILE_STRLEN: u32 = 1 << 10;
const DUK_COMPILE_NOFILENAME: u32 = 1 << 11;

// Define the FFI with duktape

extern "C" {
    fn duk_create_heap(
        arg1: *const void,
        arg2: *const void,
        arg3: *const void,
        arg4: *const void,
        arg5: *const void,
    ) -> *const void;

    fn duk_destroy_heap(ctx: *const void);
    fn duk_eval_raw(ctx: *const void, src: *const c_char, arg1: u32, arg2: u32) -> u32;
    fn duk_get_int(ctx: *const void, idx: i32) -> i32;
}

// Create a DukHeap struct which just privately holds the pointer
// to the native duktape heap

pub struct DukHeap {
    ctx: *const void,
}

impl DukHeap {
    // Create a new default heap object

    pub fn new() -> DukHeap {
        unsafe {
            let ctx = duk_create_heap(
                ptr::null(),
                ptr::null(),
                ptr::null(),
                ptr::null(),
                ptr::null(),
            );

            DukHeap { ctx }
        }
    }

    // Evaluates a JavaScript script within the current context
    // Returns the i32 on top of the stack or an error

    pub fn eval_script(&mut self, script: &str) -> Result<i32, Box<Error>> {
        let code = CString::new(script)?;

        let flags = 0
            | DUK_COMPILE_EVAL
            | DUK_COMPILE_NOSOURCE
            | DUK_COMPILE_STRLEN
            | DUK_COMPILE_NOFILENAME
            | DUK_COMPILE_SAFE;

        let res = unsafe { duk_eval_raw(self.ctx, code.as_ptr(), 0, flags) };

        if res != 0 {
            return Err(Box::new(DukError::new("Eval failed")));
        }

        let result = unsafe { duk_get_int(self.ctx, -1) };

        Ok(result)
    }
}

impl Drop for DukHeap {
    // When our DukHeap is dropped we also destroy the native heap

    fn drop(&mut self) {
        unsafe {
            duk_destroy_heap(self.ctx);
        }
    }
}

// A custom error type

#[derive(Debug)]
struct DukError {
    details: String,
}

impl DukError {
    fn new(msg: &str) -> DukError {
        DukError {
            details: msg.to_string(),
        }
    }
}

impl fmt::Display for DukError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{}", self.details)
    }
}

impl Error for DukError {
    fn description(&self) -> &str {
        &self.details
    }
}

// A simple test

#[cfg(test)]
mod tests {

    use super::DukHeap;

    #[test]
    fn my_test() {
        let mut heap = DukHeap::new();
        let script = "12 * 65 / 3";
        let result = heap.eval_script(script);

        assert_eq!(result.unwrap(), 260);
    }
}
```

Running `cargo test` tells us that things are working quite nicely!

The next step is to use this sys-crate in some consumer crate.

Let's create a new crate called `dukrs`:

```
$ cargo new --bin dukjs
```

We'll add our `duktape2-sys` crate as a dependency using the relative path as we won't be publishing it to crates.io.

```
// cargo.toml

[package]
name = "dukjs"
version = "0.1.0"
authors = ["Matt Harrison <hi@matt-harrison.com>"]

[dependencies]
duktape2-sys = { path = "../duktape2-sys" }
```

Let's write our `main.rs` to load a script passed as an argument to our binary and then execute it and print the result.

```
// main.rs

extern crate duktape2_sys;

use std::env;
use std::fs::File;
use std::io::Read;

use duktape2_sys::DukHeap;

fn main() {
    let filename = env::args()
        .nth(1)
        .expect("Provide a script name to execute");

    let mut file = File::open(filename).expect("Cannot read file");
    let mut script = String::new();
    file.read_to_string(&mut script).expect("Can't read script");

    let mut heap = DukHeap::new();

    let result = heap.eval_script(&script).expect("Couldn't evaluate script");

    println!("Result was {}", result);
}
```

Pretty simple, right? Let's create a script to test this on, we'll write a simple script using js to calculate the 20th fibonacci number:

```
// fib.js

function fibonacci(num) {
  if (num == 0) return 0;
  if (num == 1) return 1;

  return fibonacci(num - 1) + fibonacci(num - 2);
}

fibonacci(20);
```

Let's build our binary:

```
$ cargo build --release
```

Now let's give it a spin to see if it all works:

```
➜  dukjs git:(master) ✗ ./target/release/dukjs fib.js
Result was 6765
```

That's pretty sweet. So to recap: we created a sys-crate that can compile a native C library and link with a safe Rust wrapper API. We build a simple crate to consume that sys-crate which creates a binary that knows how to evaulate JavaScript.
