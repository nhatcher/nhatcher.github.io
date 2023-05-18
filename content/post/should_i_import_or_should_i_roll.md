---
title: "Trigonometric functions in WebAssembly"
date: 2023-05-19T11:41:55+02:00
Description: ""
Tags: [WebAssembly, Rust]
Categories: []
DisableComments: true
---

All the code in this blog post is available at [github](https://github.com/nhatcher/import_or_roll). I also setup a site to make some tests of your own in the [github pages](https://www.nhatcher.com/import_or_roll/).

When computing elementary transcendental functions in WebAssembly (like trigonometric functions) we have two options, we can either import the functions for the runtime or roll our own.

The second option doesn't really mean that we have two write our own implementation but that we can use the `libc` or whatever implementation we have from our standard library.

As we shall see we probably want to import and use the implementation from the host runtime, in our case the browser, and not the one given by our standard library. This is good news because it means we need to ship less code _and_ be more performant.

Think about the following problem, how many integer numbers `x` in `[0, N]` are such that `sin(x) < T`?

A simple JavaScript function would look like:

```javascript
function count_numbers(N, T) {
    let count = 0;
    for (let i = 0; i < N; i += 1) {
        if (Math.sin(i) < T) {
            count++;
        }
    }
    return count;
}
```

For instance: `count_numbers(100_000_000, 0.223) => 57_158_497`. In C:

```c
// gcc -o test tets.c -O3 -lm

#include <stdio.h>
#include <math.h>

int count_numbers(double N, double T) {
    int count = 0;
    for (double i = 0.0; i < N; i += 1.0) {
        if (sin(i) < T) {
            count++;
        }
    }
    return count;
}


int main() {
    int count = count_numbers(100000000.0, 0.223);
    printf("%i", count);
}
```

Now let's run this in WebAssembly. Let's use Rust first. Create a project like:

```bash
$ cargo new --lib test_wasm
```

Modify `cargo.toml` to contain the `crate-type = ["cdylib"]` lib:

```toml
[package]
name = "test_wasm"
version = "0.1.0"
edition = "2021"

[dependencies]

[lib]
crate-type = ["cdylib"]
```

In `lib.rs` you will need:

```Rust
#[link(wasm_import_module = "Math")]
extern "C" {
    fn sin(x: f64) -> f64;
}

#[no_mangle]
pub fn count_js_import(n: f64, x: f64) -> i32 {
    let mut count = 0;
    let mut i: f64 = 0.0;
    while i < n {
        if unsafe { sin(i) } < x {
            count += 1;
        }
        i += 1.0;
    }
    count
}

#[no_mangle]
pub fn count_rust(n: f64, x: f64) -> i32 {
    let mut count = 0;
    let mut i: f64 = 0.0;
    while i < n {
        if f64::sin(i) < x {
            count += 1;
        }

        i += 1.0;
    }
    count
}
```

The `#[no_mangle]` macros just ensure the functions are properly exported. The first lines import a function `Math.sin` from the host environment.

To compile it to WebAssembly run:

```bash
$ cargo build --release --target wasm32-unknown-unknown
```

The `wasm` compiled file will be at: `target/wasm32-unknown-unknown/release/test_wasm.wasm`. Note that wasm files produced by the Rust compiler are normally very large. You may want to remove the garbage running:

```bash
$ wasm-gc test_wasm.wasm
```

Get that file in the same folder with the driver program:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=300, initial-scale=1.0">
    <title>Should I import or should I roll?</title>
    <script>
        function count_numbers(n, t) {
            let count = 0;
            for (let x = 0; x < t; x += 1) {
                if (Math.sin(x) < t) {
                    count++;
                }
            }
            return count;
        }
        const N = 100_000_000;
        const T = 0.223;
        WebAssembly.instantiateStreaming(fetch("main.wasm"), { Math }).then((obj) => {
            const {count_rust_roll, count_js_import} = obj.instance.exports;

            console.time('count_numbers');
            console.log(count_numbers(N, T));
            console.timeEnd('count_numbers');

            console.time('count_js_import');
            console.log(count_js_import(N, T));
            console.timeEnd('count_js_import');

            console.time('count_rust_roll');
            console.log(count_rust_roll(N, T));
            console.timeEnd('count_rust_roll');
        });
    </script>
</head>
</html>
```

Finally run a local server, for instance:

```bash
$ python -m http.server
```

A version of that be found in my [github pages](https://www.nhatcher.com/import_or_roll/).

## Some numbers

As we see on the table bellow, rolling our own version of `sin` (or using one from the standard library) is somewhat slower, so we should definitely import.

We don't need to make a proper benchmark here to make a decision. Rolling your own implementation of `sin` is worse than using the browser's implementation:


|            | N=100_000   | N=10_000_000 | N=100_000_000 |
|------------|:-----------:|:------------:|:-------------:|
|JavaScript  |  0.048      |  0.173       |  1.374        |
|C Metal     |  0.020      |  0.130       |  1.100        |
|Rust Metal  |  0.020      |  0.130       |  1.100        |
|Wasm Import |  0.019      |  0.163       |  1.633        |
|Wasm Roll   |  0.048      |  1.362       | 16.146        |

Again, this is not a benchmark, so don't read too much into "pure javascript being faster than WebAssembly". This may well true in this particular case because the js engine is able to compile the function into assembly code on the spot (that is a JIT, Jut In Time compiler). That said you might want to run this in Chrome or Firefox or even your phone and see some differences.

The only takeaway here is that folks targeting WebAssembly in the browser should use the browser's `Math.sin` and not use your language's implementation.

Thanks mainly it! Thanks for reading.

## Let's also do it with C

Just because it is fun, let's do the same thing in C.

```c
// clang --target=wasm32 --no-standard-libraries -Wl,--export-all -Wl,--no-entry -o main.wasm main.c

__attribute__((import_module("Math"), import_name("sin")))
double Sin(double x);

__attribute__((visibility("default")))
int count_numbers(double N, double T) {
    int count = 0;
    for (double i = 0.0; i < N; i += 1.0) {
        if (Sin(i) < T) {
            count++;
        }
    }
    return count;
}
```

The two _attributes_ are a way to tell the clang compiler to export the function `count_numbers` and to import the function `Math.sin` as `Sin`. See, for instance (Se, for instance, (https://clang.llvm.org/docs/AttributeReference.html#import-module))

To compile it we can use (clang can compile directly to wasm we do not need emscripten):

```bash
 $ clang --target=wasm32 --no-standard-libraries -Wl,--export-all -Wl,--no-entry -o main.wasm main.c
```

To run and test the code:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=300, initial-scale=1.0">
    <title>Testing WebAssembly</title>
    <script>
        WebAssembly.instantiateStreaming(fetch("main.wasm"), { Math }).then((obj) => {
            console.time('compute');
            console.log(obj.instance.exports.count_numbers(100_000_000, 0.223));
            console.timeEnd('compute');
        });
    </script>
</head>
</html>
```

## Exercises

1. Does importing a function from the host environment incur in a performance penalty? The answer is no, you can do a similar test we did with the function `sin` with the function `abs`. Go ahead and do that!
2. Use emscripten to compile a C file that uses the C implementation for sin, maybe emscripten is smart enough, I have not tried myself!. Maybe you want to use `[musl libc](https://musl.libc.org/)`
3. Roll your own implementation of `sin` using Taylor series or any other approximation you think it is reasonable. What happens?