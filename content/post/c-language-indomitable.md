---
title: "C Language indomitable"
date: 2024-01-30T17:37:23.254Z
Description: ""
Tags: []
Categories: []
DisableComments: false
---

>     Nothing is faster than C.
>                   A. Einstein

The C programming language is old, outdated, hardly expressive, error prone and beaten. Few want to use it.
Yet, it always seems to find a way to stay relevant to modern technologies.

In this instance we are going to use C to create a minimal Wasm module to be used in cutting edge web applications. In the process we will learn a bit of C, the compiler Clang, WebAssembly, memory allocation fro dummies, utf-8, The Ryu algorithm, SIMD instructions and linear algebra. As almost everything I write this is a _learning_ and a _teaching_ opportunity.

## How to create a Wasm module with C

Probably the simplest way to produce a Wasm module is to use C program without the standard library compiled with Clang.

The reason why we do not want to use the C standard library is because it interacts too much with the OS and it is very OS dependent
In the browser many things are very different from a regular OS, files and memory being two outstanding examples.

If you need to use a more full featured version that includes some code from runtime you will be better off using emscripten.
That being said there are a very large class of cases in which no std C plus a simple memory allocator is all you need.

## Simplest example

Link to the github folder

The simplest example consist of only two files:

* main.c
* index.html

It is so small that I will list the contents here.

```c
__attribute__((visibility("default")))
double multiply_add(double a, double b) {
  return a*b + a;
}
```

You need to compile the c code with:

```bash
jsmith~:/$ clang --target=wasm32 -Oz -nostdlib -Wl,--no-entry -Wl,--export-dynamic main.c -o main.wasm
```

If you have make installed in your system you can just run `make`.

The compiled Wasm code should be very small, around 245 bytes. Yes, that is correct bytes.

The Wasm file produced is a binary file. Meaning it is not designed for human consumption.
But we can very easily convert that into a human readable wat file by either installing [wasm2wat][7] in your system or using [this][6].

{{< highlight wasm >}}
(module
  (type (;0;) (func (param f64 f64) (result f64)))
  (func $multiply_add (type 0) (param f64 f64) (result f64)
    local.get 0
    local.get 1
    f64.mul
    local.get 0
    f64.add)
  (memory (;0;) 2)
  (global (;0;) (mut i32) (i32.const 66560))
  (export "memory" (memory 0))
  (export "multiply_add" (func $multiply_add)))
{{< / highlight >}}

I will stop for a couple of minutes to contemplate that code. WebAssembly is a stack machine with linear memory model.
this module exports a function called `multiply_add` it's signature is `(f64, f64) => f64` meaning it gets two doubles (float 64) and it returns a double. The body of the function is very easy to interpret:

* put the first parameter the stack
* put the second parameter the stack
* call f64.mul. That is an assembly instruction that multiplies the last two things on the stack and puts the result in the stack
* put the first parameter on the stack (again)
* call f64.add. This is an instruction that adds the last two entries in the stack
* there is only one thing left in the stack and that is what the function returns.

The minimal bare bones, all the necessary boilerplate included, HTML file reads:

{{< highlight html >}}
<!DOCTYPE html>
<html lang="en" >
<head>
  <meta charset="UTF-8">
  <title>Simplest wasm example</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <script type="module">
    // Get the wasm and instantiate it.
    const memory = new WebAssembly.Memory({ initial: 1 });
    const env = { memory: memory };
    const { instance } = await WebAssembly.instantiateStreaming(fetch("./main.wasm"), { env });
    
    // Setup the inputs and event listeners
    const input1 = document.getElementById('input-1');
    const input2 = document.getElementById('input-2');
    const update = () => {
        const value1 = Number(input1.value);
        const value2 = Number(input2.value);
        const result = instance.exports.multiply_add(value1, value2);
        document.getElementById('result').innerText = `${result}`;
    };
    input1.addEventListener('change', update);
    input2.addEventListener('change', update);

    // first time load
    update();
  </script>
</head>
<body>
<div class="title">A simple Wasm module!</div>
<div><span>Input 1:</span><input id="input-1" type="number" value="5"></div>
<div><span>Input 2:</span><input id="input-2" type="number" value="6"></div>
<div><span>Result of multiply-add:</span><span id="result"></span></div>
</div>
{{< / highlight >}}

You can find it working here, or you can spin up your own server and try it locally:

```bash
jsmith:~/ python -m http.server
```

And open your browser at <http://localhost:8000>

#### Exercises

1. Try to compile the program without the optimization -Oz. Convert the code to human readable wat. What do you observe?
2. Change the function multiply_add to something else, like the norm of a vector (a, b) => a^2 + b^2
3. Add a second function (for instance the function you created in exercise 2) and inspect the wat file, what can you learn from it?

## Bump allocator

This section in based on ideas from [Surma][surma].

We are going to add to vectors of length `n`. In order to do this we need to allocate memory for the resulting vector.
Note that in this particular case we could use one of the vectors and modify it's content. 

The example in the previous section, while certainly useful for many use cases, lacks some basic features.
First we cannot allocate memory on the heap and second we cannot pass or return strings. In this and the next sections we will remedy the first issue.

If you have not heard about it before this is an excellent moment for you to learn the difference between the stack and the heap.
The stack is a part of the memory that is known at compile time. The numbers you define inside a function fall in this class.

The heap is different. Objects allocated on the heap are not known at compile time. For instance an array of numbers or a list of strings whose length is not known before hand. In general the program does not have enough memory to allocate for this kind of objects and needs to ask for memory to the underlying OS. If the OS does have memory available will give a pointer with the address to the program. When the program is done using the object it should return the memory to the OS. Failing to do that will result in a memory leak.


One of the best things of not being able to use the C standard library is that we need to roll out our own implementation for a memory allocator. A C memory allocator (malloc/free) is one of those things people do in university to learn how things work and never use them again. Here, we have to


#### Exercises

1. Study the wat file produced by `main.wasm`. Can you understand what the Wasm instructions are doing?
You will want to consult the [Wasm specification][wasm-spec]

## Memory alignment and a walloc

## String

Most programs are inevitably going to end up using strings. Today this means unicode and utf-8 encoded strings.
WebAssembly does not have a concept of strings (yet) so we are bound to learn something new.
In this section we will build code that calculates the length of a string, reverses a string and says "Hello" + your name.
This should be enough for you to build anything you like with strings.

### The basics of utf-8

[Wikipedia][wikipedia-utf-8] has a good explanation on the subject that we follow. Brian Kernighan's explanation in the [Go programming Language][go-programming-language] is also excellent.

A string is composed out of chars. A char is technically a _unicode scalar value_ an in the `UTF-8` standard is encoded with 1, 2, 3 or 4 bytes.
Most used chars in the English language are encoded with just one byte. The first significant digit of this byte being 0: `b0xxxxxxx`. there ar 128 different chars. Chars that need to bytes for the encoding use bytes of the form [`b110xxxxx`, `b10xxxxxx`]. There are 1920 chars in this batch (see Exercise 2). Most letters like `à` or `ñ` are in this list. The Euro character `€` is encoded with three bytes because it was created long after unicode. In JavaScript you can:

```javascript
const encoder = new TextEncoder();
console.log(encoder.encode('€'));
>>> Uint8Array(3) [ 226, 130, 172 ]
```

To see the result of the encoding of any string into bytes.

### Passing string to WebAssembly

The previous example actually shows us the first bit, to pass a JavaScript string to WebAssembly we need to convert it into an `Uint8Array`. We can already write the JavaScript code for the wrapper function:

```javascript
function stringLength(str) {
    const encoder = new TextEncoder();
    const v = encoder.encode(str);
    const n = v.length;
    const pointer = malloc(n);
    
    const c = new Uint8Array(memory.buffer, pointer, n);
    c.set(v);

    const l = string_length(pointer, n);
    free(pointer);
    return l;
}
```

Where `string_length` is the Wasm function that computes the utf-8 length of the string. In the code above `n` would be the length of the byte array which is always `>=l`. Note that this is a especially dumb example. In JavaScript you can always use `str.length` to compute the char length of a string.

### Getting Strings from WebAssembly

How do we get a string back from WebAssembly, well, actually how do we get a `Uint8Array` back from WebAssembly.
We already did encounter this problem before when we were adding two vectors. In that situation we allocated some memory for the C program and said "hey pal could you put the result of the addition in that memory address?".
But we have an slightly more difficult problem here because, in general, we do not know the size of the resulting string.
We could, of course, make a guess that is enough for the result or resort to some other hack but the common pattern is to send to the C program a pointer to an int. Call it `p`. The C function will allocate the necessary memory and set the memory address of the resulting `Uint8Array` in that pointer. That is `p` wind up being a pointer to a pointer.
There is yet another catch. We know now the position in memory of the resulting string, but we do not know how long it is. There are two traditional ways to solving this issue. One is to set a null terminating character at the end of the string. That is the C approach.
The other is to have pass yet another pointer to an int that will hold the 

See it in action:
```javascript
// An 32 bit int is four bytes 
const SIZE_OF_INT = 4;

function sayHello(name) {
    // Encode the string we are passing in an Uint8Array
    const encoder = new TextEncoder();
    const v = encoder.encode(name);
    const n = v.length;
    const pointerName = malloc(n);
    const c = new Uint8Array(memory.buffer, pointerName, n);
    c.set(v);

    // This pointer will hold a pointer to the memory position of the result 
    const pResult = malloc(1 * SIZE_OF_INT);

    // this pointer will hold the byte length of the resulting string
    const pSize = malloc(1 * SIZE_OF_INT);
    const cResult = new Int32Array(memory.buffer, pResult, 1);
    const cSize = new Int32Array(memory.buffer, pSize, 1);

    // Make the call
    say_hello(pointerName, n, pResult, pSize);
    
    // Again: the value of cResult is not the result but a pointer to the result
    const result = new Uint8Array(memory.buffer, cResult[0], cSize[0]);

    // Convert the byte array into a string
    const decoder = new TextDecoder();
    const say =  decoder.decode(result);

    // free everything
    free(cResult[0]);
    free(pResult);
    free(pSize);

    return say;
}
```

### Excercises

1. Let's learn some C advanced concepts and check how tey translate into WebAssembly. In the function `string_reverse` when we wanted to copy 4 bytes we did:

```c
dest[di] = src[si];
dest[di+1] = src[si+1];
dest[di+2] = src[si+2];
dest[di+3] = src[si+3];
```

You can shorten this by:

```c
*((int32_t*)&dest[di]) = *((int32_t*)&src[si]);
```

In C a pointer is a position in memory alongside with he type it is pointing towards. For example `&src[si]` is pointing towards the byte position of memory where `src[si]` is located. If we do `(int32_t*)&src[si]` that means covert (cast) that pointer into a pointer to a 32 bit number. The next step is `*(int32_t*)&src[si]` which means take the content of the pointer.
This kind of thing is the bread and butter of the _advanced_ C programmer.

Can you see if there is any difference in the Wasm code between one and the other.
NOTE: The original code is adapted from [QuicJs][quick-js] implementation.

2. How many chars can we encode with 2 bytes, 3 bytes and 4 bytes? 
Could we find an alternative unicode encoding with only three bytes for all those characters? 

## The turn of the screw. Importing methods. Painting in the canvas

Here we describe how to add some elements from the C standard library (libc) 

Ryū algorithm?
// https://www.youtube.com/watch?v=kw-U6smcLzk
// Maybe: https://blog.benoitblanchon.fr/lightweight-float-to-string/

## SIMD instructions

In this section we will add SIMD instructions (Single Instruction, Multiple Data) to our toolbox and use them to implement matrix multiplication.

I will be honest, I was not planning on writing this post. A few weeks ago someone asked me "Hi Nico, what are you doing over the weekend?". I thought "oh!, I have a free weekend, it might be a good idea to write about Rust, Fast Fourier Transforms and SIMD instructions in Wasm". Naturally all that became this post you are reading now. So I guess SIMD instructions is where I draw the line. They are in.

They are also a nice addition in the new standard WebAssembly 2.0.

In short with a single SIMD you can operate in batches of numbers. In the following diagram a 4 lane a `i32x4.add` instruction is shown:

![SIMD instructions](/images/SIMD.svg)

WebAssembly SIMDs operate on 4 lanes of 32 bit numbers or 2 lanes of 64 bit numbers, for instance. In short they perform operations on 128 bits at a time.
In theory then they can double or quadruple the speed of your computations. Remember that Wasm is no a _real_ assembly language, but mimics many properties of some.
Real machines do have SIMD instructions and even if you have never heard about SIMD or assembler you might have heard the marketing of it:

![MMX](/images/PentiumMMX-presslogo.jpg)

Although SIMD instructions where used by a computer as early as 1965 in the [ILLIAC IV](https://en.wikipedia.org/wiki/ILLIAC_IV) supercomputer it was not until the late 90's that Intel was to put SIMD instructions on a personal computer. Those were branded MMX instructions and made marketing history. They operated in lanes of 128 bits. Later Intel came up with the _Streaming SIMD Extensions_ (SSE) and the continuing Advanced Vector Extensions (AVX). That last family has 256 bit lanes and in the more powerful computers 512 lanes (unless you know, it is unlikely that you have a processor that supports AVX-512).

More than a quarter of a century after after personal computers all over the world could enjoy SIMD instruction we can today use them in the browser (actually the Dart programming language, designed by no other than Lars Bak, could target the Web in the Dart VM back in 2013!).

This kind of parallel computing cannot help with every kind of task, of course. But there are quite a few tasks for with SIMD instructions can speed up the game quite a lot: 3D graphics and physics, image and signal processing, numerical computing.... Not a silver bullet though, as Daniel Lemire famously [showed](https://lemire.me/blog/2018/09/07/avx-512-when-and-how-to-use-these-new-instructions/) AVX-512 instructions can damage the speed of your application. Those instructions are so power hungry that the processor is forced to operate a lower frequencies (a technique known as throttling). This alongside many other complications made a grumpy Linux Torvalds [declare](https://www.phoronix.com/scan.php?page=news_item&px=Linus-Torvalds-On-AVX-512) "I Hope AVX-512 Dies A Painful Death".

That being said, when used right SIMD instructions can be a huge win.
For instance the above mentioned Daniel Lemire has been leading the way using SIMD to a [variety](https://lemire.me/en/publication/arxiv1910/)-[of](https://lemire.me/en/publication/arxiv190208318/)-[algorithms](https://lemire.me/en/publication/arxiv210910433/) that already made their way into the standard libraries of modern programming languages like go or Rust. See [Lemire's publications](https://lemire.me/en/#publications) for the entire list.

Before we actually use SIMD in WebAssembly I would like to tell you how it is used on a regular C program. C, like many other programming languages does not have primitives to talk about SIMD instructions. In place of that uses something called _[intrinsics](https://www.intel.com/content/www/us/en/develop/documentation/cpp-compiler-developer-guide-and-reference/top/compiler-reference/intrinsics.html)_. Quoting Intel: those are assembly coded functions that let you use C function call and variables in place of the assembly instructions. They are always expanded inline providing the same benefit as using inline assembly.

Thus we will be basically programming in assembly with C looking touch. At the bare minimum we basically need:

* A way to combine several numbers (could be 32 bit floats, 64 bit doubles, ints, uints, ...) into a group of them to form a SIMD register
* Instructions to operate those registers (add, sub, mult, div)
* A way to extract numbers form a SIMD register

Let us assume our registers are of 128 bits length and that we are working with 32 bit floats.
a extremely simple instance would be:

```C
// program: simple_128.c
// compile with
// clang -O3 -march=native -o simple_128 simple_128.c
// run: ./simple_128 12

#include <immintrin.h>
#include <stdio.h>

void print_vector(__m128 vector) {
    float *vector_f = (float *)&vector;
    for (int i=0; i < 4; i++) {
        printf("%f ", vector_f[i]);
    }
    printf("\n");
}

int main(int argc, char **argv) {
    // Get a number from the command line
    // This is to prevent the compiler from optimizing too much
    if (argc < 2) {
        printf("Usage: %s <number>\n", argv[0]);
        exit(1);
    }
    float a = atof(argv[1]);

    __m128 first_row = _mm_set_ps(2.0*a, 4.0, 6.0*a, 8.0);
    __m128 second_row = _mm_set_ps(1.0, 3.0*a, 5.0, 7.0*a);

    __m128 result = _mm_sub_ps(first_row, second_row);

    print_vector(first_row);
    print_vector(second_row);
    print_vector(result);

    return 0;
}
```

Let's walk through that example. For the SIMD instructions it wil be useful to have the [intel guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#techs=SSE) at hand.

The first intrinsic we find in a new data type `__m128` that will host 4 32-bit numbers, 8 64-bit numbers, etc.
In our case we will use it to host 4 32-bit floats. The two intrinsics are `_mm_set_ps` and `_mm_sub_ps` where ps stands for _packed single-precision_.
Perhaps of interest is also how we extract the float values from the lines. We will see that in the WebAssembly case there are instructions for that but here the trick is:

```C
float *vector_f = (float *)&vector;
```

This is actually a trick that we have seen before in this post when talking about utf-8. Here `_m128 *p = &vector` is a pointer that points to a type with 128 bits. Meaning that `p[1]` will point to a position of memory 128 bits after `p`. In `float *vector_f = (float *)&vector;`, `vector_f` points to the _same_ position of memory but `vector_f[1]` will increment the memory position only by 32 bits.

And that's all there is to it. You can find instructions to multiplicate `__m128_mul_ps`, divide and so forth. If you want to add, multiply, etc two double number numbers instead you will need _SSE3_ instructions provided by the intrinsics `_mm_add_pd`, `_mm_mul_pd`, (where pd stands for _packed double_), etc.

Let's turn our eyes to WebAssembly now.

A similar program written in WebAssembly:

```C
```







## Should I use pure Clang for my project or emscripten

Chances are you will be needing emscripten nad it is a great tool.

However if you have a compact C module that does not use the standard library beyond malloc/free and a couple of other methods that you can replicate you might consider this approach to have a simple lean and straightforward way to produce your Wasm module.

The emscripten toolchain is rather big and complex and it would be easy for you to include code you do not need.

### Exercise

1. Try zig!

## Installing Clang

To follow the code in this post you need to have clang installed in your system. I am using clang version 13.
What I do is I download the binaries and copy them to `/opt/clang/` then export `EXPORT CLANG=/opt/bin/clang`

## Appendix: The basics of WebAssembly

I think is extremely useful if you at least know a couple of things about WebAssembly.

## References:


* [Walloc][walloc] is a bare bones malloc implementation for WebAssembly.
* The idea of the bump allocator was suggested in the beautiful post of [Surma][surma]
* If you want to learn the internals of Wasm no better place than the [WebAssembly specification][wasm-spec]
* The llvm Wasm [linker options][wasm-ld]
* Understanding [Wasm text format][wat-mozilla]
* Many other languages to target WebAssembly today, zig [zig][zig-wasm], [go][go-wasm] or my personal favourite [rust][rust-wasm]
* Fabrice Bellard's [QuickJs][quick-js] is a good place to look for inspiration.


[walloc]: https://github.com/wingo/walloc
[tinyalloc]: https://github.com/thi-ng/tinyalloc
[surma]: https://surma.dev/things/c-to-webassembly/
[wasm-spec]: https://webassembly.github.io/spec/core/
[wasm-ld]: https://lld.llvm.org/WebAssembly.html
[wat-mozilla]: https://developer.mozilla.org/en-US/docs/WebAssembly/Understanding_the_text_format
[wasm2wat-demo]: https://webassembly.github.io/wabt/demo/wasm2wat/
[wabt]: https://github.com/WebAssembly/wabt
[zig-wasm]: https://ziglang.org/documentation/master/#WebAssembly
[go-wasm]: https://github.com/golang/go/wiki/WebAssembly
[rust-wasm]: https://rustwasm.github.io/book/
[quick-js]: https://bellard.org/quickjs/
[wikipedia-utf-8]: https://en.wikipedia.org/wiki/UTF-8
[go-programming-language]: https://www.gopl.io/


## NOTES:

Title stolen without permission from [Raoul Bott's](http://www.numdam.org/article/PMIHES_1988__68__99_0.pdf) formidable rendition to Ed Witten's Interpretation of Morse Theory
