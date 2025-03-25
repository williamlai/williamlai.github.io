---
title: Rust and C coexist
date: 2022-11-14 12:10:23
categories: tech
tags:
- Rust
- C
---

To call a C function in Rust or to call a Rust function in C is similar to how it works in C & C++. In the following example, I'm gonna build Rust as a static library and use C as the main executable application.

<!-- more -->

# Create a static Rust library

Use the following command to create a Rust library `rustlib`.

```Rust
cargo new rustlib --lib
```

Modify `Cargo.toml` file and add the following setting.

```Toml
[lib]
crate-type=["staticlib"]
```

With this setting, it'll build a static library. Please refer to [Linkage](https://doc.rust-lang.org/reference/linkage.html) for more details on this setting.

# Add code

In this example, the C code will call Rust to add one on a number and get the result. The Rust code will do 1-second sleep, add 1 to a number and return the result.

## Add C code

Create a file `adder.c` with the following content in the root folder of the Rust library.

```C
#include <stdio.h>
#include <unistd.h>
#include <stdint.h>

int32_t rust_add_one(int32_t num);

void c_do_sleep(int32_t seconds) {
    printf("sleep %d seconds\n", seconds);
    sleep(seconds);
}

int main() {
    int32_t x = 0;

    while (x < 10) {
        x = rust_add_one(x);
        printf("%d\n", x);
    }

    return 0;
}
```

The code includes a function prototype `rust_add_one` that is supposed to be implemented in Rust code.

Then it implements a function named `c_do_sleep` for Rust to call it for sleeping.

The main function keeps adding one to x by calling Rust function `rust_add_one` until it's larger or equal to 10.

## Add Rust code

Here is the content of `lib.rs`

```Rust
extern {
    fn c_do_sleep(seconds: i32);
}

#[no_mangle]
pub extern "C" fn rust_add_one(num: i32) -> i32 {
    unsafe {
        c_do_sleep(1);
    }
    num + 1
}
```

At first, it externs a C function `c_do_sleep` for later usage.

Then it implements a function named `rust_add_one` with an `extern "C"` marker. The marker makes it callable from C code. However, the name is mangled, so that the function name won't be `rust_add_one` in the symbol table. To demangle it, add `#[no_mangle]` marker on function `rust_add_one` so that C code can call it with the exact name.

The implementation of `rust_add_one` calls `c_do_sleep` for sleeping one second then adds one to the number and returns. The `c_do_sleep` function call is wrapped within an `unsafe` section because it's an extern call, and it's regarded as an unsafe code in Rust.

## Link C code and Rust library, then run it

Run the following command to build the Rust static library.

```Bash
cargo build --release
```

After built, there will be a static library in `target/release/librustlib.a`.

Then use the following command to compile and link the C code.

```Bash
gcc -o adder adder.c target/release/librustlib.a
```

It should be able to link everything. Now run it and check the result.

```Bash
./adder
sleep 1 seconds
1
sleep 1 seconds
2
sleep 1 seconds
3
......
```

# Conclusion

One reason to make C and Rust coexist is that sometimes we have a legacy C code and want to re-write it into Rust code. One strategy is porting components one by one. While porting, adding some glue layers is necessary. So this example shows the idea of it.
