---
title: Run Rust in WASM
date: 2022-11-21 16:57:48
categories: tech
tags:
- rust
- wasm
- esp32
---

Based on the article [Run WASM application on ESP32](https://williamlai.github.io/2022/11/17/run-wasm-app-on-esp32/), I'm gonna try to run Rust on the ESP32 WASM runtime.

<!-- more -->

# How to run Rust on ESP32 WASM engine

The ESP32 WASM runtime is one of the [WAMR](https://github.com/bytecodealliance/wasm-micro-runtime) implementations. Since it runs on top of MCU, we expect it only to have a few megabytes of memory.

For a Rust application, it has to be:
* No Rust standard library: It's impossible to put all Rust standard libraries into either WAMR or ESP32 native environment.
* No default entry: Like what we did in The C application, the WASM application doesn't have a traditional entry point.

This means we have to make a Rust application small enough, and then we can consider compiling it into WASM byte code.

# Recude footprint in Rust

Let's take an empty main function example. Create a project named `hello` with command `cargo new hello`. The content of `src/main.rs` is as follows.

```Rust
fn main() {
}
```

Make a release build with the command `cargo build --release`. The size of the executable on a Linux x86_64 machine is around 4MB.

```
$ ls -l target/release/hello
-rwxr-xr-x 2 william william 4027256 Nov 22 05:11 target/release/hello
```

And we can see it has the following dependent libraries.

```
$ ldd target/release/hello
        linux-vdso.so.1 (0x00007fffc7ae9000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007ff06db80000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007ff06db5d000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007ff06db50000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ff06d950000)
        /lib64/ld-linux-x86-64.so.2 (0x00007ff06dc04000)
```

## No standard library

By default, a Rust program is built with Rust standard library, which includes `Box`, `String`, `Vec`, `Hashmap`, etc. We can use `#![no_std]` tag to explicitly not use Rust standard library.

```Rust
#![no_std]

fn main() {
}
```

However, it won't build with the following error messages.

```
$ cargo build --release
   Compiling hello v0.1.0 (/mnt/c/workspace/temp/hello)
error: `#[panic_handler]` function required, but not found

error: language item required, but not found: `eh_personality`
  |
  = note: this can occur when a binary crate with `#![no_std]` is compiled for a target where `eh_personality` is defined in the standard library
  = help: you may be able to compile for a target that doesn't need `eh_personality`, specify a target with `--target` or in `.cargo/config`

error: could not compile `hello` due to 2 previous errors
```

So there are two errors to be fixed, "panic handler" and "`eh_personality`".

### Panic handler

The first one is the panic handler. The default panic handler is included in the Rust standard library. Since we don't use it, we must provide our panic handler.

```Rust
#![no_std]

use core::panic::PanicInfo;

#[panic_handler]
fn my_panic_handler(_info: &PanicInfo) -> ! {
    loop {}
}

fn main() {
}
```

### eh_personality

The second error is `eh_personality`. It's a feature to unwind the stack and show the backtrace. The default implementation of `eh_personality` is included in the Rust standard library. There are two approaches to fixing this error.

The first approach is to specify the panic handle to "abort". This makes Rust abort when an error happens. In the `Cargo.toml`, add the following.

```
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

The second approach is adding our own `eh_personality` function as follows. In this approach, it can only build for nightly Rust build instead of stable Rust build.

```Rust
#![no_std]
#![feature(lang_items)]

#[lang = "eh_personality"]

extern "C" fn eh_personality() {}

use core::panic::PanicInfo;

#[panic_handler]
fn my_panic_handler(_info: &PanicInfo) -> ! {
    loop {}
}

fn main() {
}
```

I prefer the first approach.

## Entry point

As we fixed the panic handler and `eh_personality`, we still get the following error in the build.

```
$ cargo build --release
   Compiling hello v0.1.0 (/mnt/c/workspace/temp/hello)
error: requires `start` lang_item

error: could not compile `hello` due to previous error
```

The context of this error is that Rust requires Rust runtime. The Rust runtime is implemented in the `start` language item and is for protecting the stack, storing command line arguments, and stack unwinding. It'll then call the `main` function when these features are ready.

The `start` language item is included in the Rust standard library, which means we don't have a Rust runtime when we don't use the Rust standard library. Therefore, we don't need the Rust entry point `main` by adding `#![no_main]`, and we need to provide an entry point function. It's `_start` in Linux. The content of `src/main.rs` is as follows.

```Rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

#[panic_handler]
fn my_panic_handler(_info: &PanicInfo) -> ! {
    loop {}
}

#[no_mangle]
pub extern fn _start() -> ! {
    loop {}
}
```

The implementation of the `_start` function is an infinite loop because we don't have a proper way to exit the program right now. It requires an `exit` function and implements needed codes.

Now we can build it successfully with the following command. It tells the `ructs` to use the link argument `-nostartfiles`, which means don't use default start files, and we'll provide our own.

```Bash
$ cargo rustc --release -- -C link-arg=-nostartfiles
```

We can put the link arguments in `.cargo/config.toml`, as follows.

```TOML
[target.'cfg(target_os = "linux")']
rustflags = ["-C", "link-arg=-nostartfiles"]
```

And we can build it as usual.

```Bash
$ cargo build --release
```

After being built, the size of the executable is 13KB now. It can be even smaller, but it involves some ELF file surgery, which is out of scope in this article.

```
$ ls -l target/release/hello
-rwxr-xr-x 2 william william 13600 Nov 22 06:32 target/release/hello
```

And it has no dependent libraries.

```
$ ldd target/release/hello
        statically linked
```

We can run this program. But we have to use `Ctrl-C` to exit the program since it only has an infinite loop in the `_start` function.

## Print Hello World

So far, we only have a small program that does nothing. To be able to print "Hello, world!" or other tasks, we can still use C standard library by using [libc crate](https://crates.io/crates/libc). It supports many C standard functions like `printf`, `malloc`, `free`, `memcpy`, etc. To use it, we need to add the `libc` crate in `Cargo.toml`.

```TOML
[dependencies]
libc = "0.2"
```

Modify `src/main.rc` as follows.

```Rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;
use libc::{printf, exit};

#[panic_handler]
fn my_panic_handler(_info: &PanicInfo) -> ! {
    loop {}
}

#[no_mangle]
pub extern fn _start() {
    unsafe {
        printf("Hello, world!\n\0".as_ptr() as *const i8);
        exit(0);
    }
}
```

Please note that there is no EOS (End-Of-String) character `\0` in the string slice in Rust, so we have to add it in case it prints out mojibake.

We also use `exit`. Now our program can exit correctly.

Use the following command to compile it.

```Bash
cargo rustc --release -- -C link-args=-nostartfiles -lc
```

The `-lc` means linking the C library. Now we can check how it runs. It should print the message and exit successfully.

```Bash
$ ./target/release/hello
Hello, world!
```

And the size is 14KB which is slightly bigger than the previous one.

```Bash
$ ls -l target/release/hello
-rwxr-xr-x 2 william william 14224 Nov 22 09:17 target/release/hello
```

It also has C library dependencies.

```
$ ldd target/release/hello
        linux-vdso.so.1 (0x00007fffe0238000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f2f74b30000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f2f74d42000)

```

# Run Rust program in WASM

Refer to [Build WASM application](https://github.com/bytecodealliance/wasm-micro-runtime/blob/main/doc/build_wasm_app.md), we need to install `wasm32-wasi` with the following command.

```Bash
$ rustup target add wasm32-wasi
```

We can build a default "Hello, world!" Rust example with the following command.

```Bash
$ cargo build --release --target wasm32-wasi
```

The size of the WASM byte code is big because it links lots of things. 

```Bash
$ l -l target/wasm32-wasi/release/hello.wasm
-rwxr-xr-x 2 william william 2030863 Nov 22 10:03 target/wasm32-wasi/release/hello.wasm*
```

We can reduce the footprint with the hints from the previous section.

The entry point of the WASM application is the `main` istead of `_start`, so we have to modify our source code as follows.

```Rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;
use libc::{printf, exit};

#[panic_handler]
fn my_panic_handler(_info: &PanicInfo) -> ! {
    loop {}
}

#[no_mangle]
pub extern "C" fn main() {
    unsafe {
        printf("Hello, world!\n\0".as_ptr() as *const i8);
        exit(0);
    }
}
```

The content of `Cargo.toml` doesn't change, as follows.

```TOML
[package]
name = "hello"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"

[dependencies]
libc = "0.2"
```

We can build it with the following commands.

```Bash
$ cargo rustc --release --target wasm32-wasi -- -C link-arg=--initial-memory=65536 -C link-arg=-zstack-size=8192 -C link-arg=--export=__heap_base -C link-arg=--export=__data_end -C link-arg=--strip-all
```

We can also put these link arguments in `.cargo/config.toml`.

```TOML
[build]
rustflags = [
  "-C", "link-arg=--initial-memory=65536",
  "-C", "link-arg=-zstack-size=8192",
  "-C", "link-arg=--export=__heap_base",
  "-C", "link-arg=--export=__data_end",
  "-C", "link-arg=--strip-all",
]
```

And build it with the following command.

```Bash
$ cargo build --release --target wasm32-wasi
```

Now the size of the WASM byte code is only 250 bytes! 

```Bash
$ ls -l target/wasm32-wasi/release/hello.wasm
-rwxr-xr-x 2 william william 250 Nov 22 10:02 target/wasm32-wasi/release/hello.wasm
```

We can put the application on ESP32 WASM as we did in ["Build WASM application"](https://williamlai.github.io/2022/11/17/run-wasm-app-on-esp32/#Build-WASM-application).

```Bash
$ cp target/wasm32-wasi/release/hello.wasm ./test.wasm
$ xxd -i test.wasm <PATH_TO_ESP32_PROJECT>/esp32-wasm-test/main/test_wasm.h
```

Then go to the directory of the ESP32 WASM sample project, build and flash it. (Remember to configure related environment variables.)

```Bash
$ idf.py build
$ idf.py flash monitor
```

We can see the following logs.

```
[00:00:00:659 - 3FFB7620]: warning: failed to link import function (env, __original_main)
Hello, world!
Exception: env.exit(0)
E (10) wamr: Error while app_instance_main
I (397) wamr: Exiting...
```

There is an exception which calling `exit`. So the WASM might have its way of exiting the program, but we don't have to explicitly add one. Modify `src/main.rs` as follows.

```Rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;
use libc::printf;

#[panic_handler]
fn my_panic_handler(_info: &PanicInfo) -> ! {
    loop {}
}

#[no_mangle]
pub extern "C" fn main() {
    unsafe {
        printf("Hello, world!\n\0".as_ptr() as *const i8);
    }
}
```

Then build WASM applications, put the byte code into the ESP32 project, build the ESP32 project, flash and check the logs.

```
[00:00:00:632 - 3FFB7618]: warning: failed to link import function (env, __original_main)
Hello, world!
I (381) wamr: Exiting...
```

Now it exits successfully. Great.

# Why embeded Rust?

At first, it seemed pointless to me to use Rust in a resource-limited environment. To run Rust in either native MCU or WAMR in MCU, Rust won't be able to use the Rust standard library. I guess it's a similar situation as FreeRTOS. Although there is not enough resource or library in FreeRTOS compared to Ubuntu or other powerful platforms, running programs in a small, real-time environment is still essential. Rust can be small and efficient. Moreover, Rust brings safe code and a more productive workflow. So I think Rust does have its place in the embedded MCU environment.