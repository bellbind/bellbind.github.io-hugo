---
title: "Implement Hash functions with C"
slug: "hash-functions-with-c"
date: 2017-07-16T21:24:40+09:00
categories: ["2017-07"]
tags: ["c","algorithm","cryptography", "emscripten"]
draft: false
---

Note about imprementing major cryptographic Hash functions(SHA256, SHA512, SHA1, MD5, RIPEMD169)  with standard C11.
Entire codes in gist are:

- [SHA256](https://gist.github.com/bellbind/2fd66a09340942b9845ef432a5c8c419)
- [SHA1](https://gist.github.com/bellbind/ff05971591d4fca28f001c143d46ae5f)
- [MD5](https://gist.github.com/bellbind/6cbaa8023e7588c6cbd17756f7c3ba42)
- [RIPEMD160](https://gist.github.com/bellbind/d9368aafc1d9ee958f1d09c9305f48a9)
- [SHA512](https://gist.github.com/bellbind/030c8786e53af6c763fdf0e3fdbecbcc)

These hash are similar algorithms. Once writing one of them, writing others would be easy.

<!--more-->

----
## Port from SHA256 ECMAScript implementation

At first, I wrote the [SHA256 function with pure ECMAScript](https://bellbind.github.io/2017/07/08/implement-sha256-ecmascript/).
The implementation is very slower than other implementation: coreutils `sha256sum` and `crypto` in node.js.

So I want to comparing them with WebAssembly based implementations.
Before the writing pure WebAssembly program, I use the emscripten to generate from C implementations.
Then I make the C11 SHA256 implementation from experiments of ECMAScript version.

Implementing SHA256 with C11 is easier than with ECMAScript,  because:

- 32bit unsigned integer operations exists: no overflow/minus sign processing required
- SHA256 not required memory management: no `malloc()` and `free()`

Used standard library functions are just `memcpy()` and `memset()` in `<string.h>`. 
They are easy to implement yourself (when no platform dependent speed up codes).

The export API's are similar with ECMAScript versions: `sha256_init()`, `sha256_update()` and `sha256_digest()`.
These arguments are pointers for memories allocated on the stack (local variables) as:

```c
#include <stdio.h>
#include "sha256.h"

int main() {
  struct sha256 ctx;
  unsigned char digest[SHA256_HASH_BYTES];
  char* buffer = "Hello World!";

  sha256_init(&ctx);
  sha256_update(&ctx, buffer, 12);
  sha256_digest(&ctx, digest);

  for (size_t i = 0; i < SHA256_HASH_BYTES; i++) printf("%02x", digest[i]);
  printf("  \"%s\"\n", buffer);
  return 0;
}
```

### About function to big endian

I prefer to use `union` to convert on memory layout than bit operations as:

```c
static inline uint32_t big32(uint32_t v) {
  union {
    struct {uint8_t b0, b1, b2, b3;};
    uint32_t v;
  } le = {.v = v};
  union {
    struct {uint8_t b3, b2, b1, b0;};
    uint32_t v;
  } be = {.b0 = le.b0, .b1 = le.b1, .b2 = le.b2, .b3 = le.b3};
  return be.v;
}
```

When using bit operations to convert endian, I prefer filter cascading:

```c
static inline uint32_t big32(uint32_t v) {
   v = ((v & 0xff00ff00) >> 8) | ((v & 00ff00ff) << 8);
   return ((v & 0xffff0000) >> 16) | ((v & 0000ffff) << 16));
}
```

In standard C, there are no method to judge host endians **at compile time**.
But gcc and clang has 
[predefined macros: `__BYTE_ORDER__`, `__ORDER_BIG_ENDIAN__`](https://gcc.gnu.org/onlinedocs/cpp/Common-Predefined-Macros.html).

It can print as:

```bash
$ cc -dMN -E - < /dev/null | grep BYTE_ORDER
#define __BYTE_ORDER__ __ORDER_LITTLE_ENDIAN__
```

### Manual loop extraction

In invertal `update()` function, the loops for mxing chunks to the vector is not effective
because all variable assigned at last but only few variables(for example, `d` and `h` in SHA256) 
updated in each step, other assignments are just rotating.

```c
  // loop version
  uint32_t a = sh[0], b = sh[1], c = sh[2], d = sh[3],
    e = sh[4], f = sh[5], g = sh[6], h = sh[7];
  for (int i = 0; i < 64; i++) {
    uint32_t s0 = rotr(a, 2) ^ rotr(a, 13) ^ rotr(a, 22);
    uint32_t s1 = rotr(e, 6) ^ rotr(e, 11) ^ rotr(e, 25);
    uint32_t ch = (e & f) ^ (~e & g);
    uint32_t maj = (a & b) ^ (a & c) ^ (b & c);
    uint32_t t1 = h + s1 + ch + K[i] + w[i];
    uint32_t t2 = s0 + maj;
    h = g; g = f; f = e; e = d + t1;
    d = c; c = b; b = a; a = t1 + t2;
  }
```

When extracted loop, variable assignments are turned to parameter rotation.
The step updates the variables only, others are just referred.
The execution time would be reduced for the loop extraction,
called [loop unrolling](https://en.wikipedia.org/wiki/Loop_unrolling) technique.

```c
// step as inline function
static inline void
step(uint32_t a, uint32_t b, uint32_t c, uint32_t* d,
     uint32_t e, uint32_t f, uint32_t g, uint32_t* h, uint32_t w, uint32_t k) {
  *h += t1(e, f, g, w, k);
  *d += *h;
  *h += t2(a, b, c);
}
```

```c
  // extracted loop with inline function
  uint32_t a = sh[0], b = sh[1], c = sh[2], d = sh[3],
    e = sh[4], f = sh[5], g = sh[6], h = sh[7];
  step(a, b, c, &d, e, f, g, &h, w[0], K[0]);
  step(h, a, b, &c, d, e, f, &g, w[1], K[1]);
  step(g, h, a, &b, c, d, e, &f, w[2], K[2]);
  step(f, g, h, &a, b, c, d, &e, w[3], K[3]);
  step(e, f, g, &h, a, b, c, &d, w[4], K[4]);
  step(d, e, f, &g, h, a, b, &c, w[5], K[5]);
  step(c, d, e, &f, g, h, a, &b, w[6], K[6]);
  step(b, c, d, &e, f, g, h, &a, w[7], K[7]);
  step(a, b, c, &d, e, f, g, &h, w[8], K[8]);
  ....
  step(b, c, d, &e, f, g, h, &a, w[63], K[63]);
```

FYI: execution times for 512MB data as:


```bash
$ dd if=/dev/zero bs=1m count=512 of=512M.data
$ time ./sha256 512M.data
```

|                    | real time |
|--------------------|----------:|
|loop      (clang-4) |3.684s|
|extracted (clang-4) |3.205s|
|loop      (gcc-7)   |3.052s|
|extracted (gcc-7)   |3.063s|
|(coreutil sha256sum)|3.207s|



----
## About main function: emulating coreutils hash commands

At first, I only implemented output to coreutils compatible format as:

```
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  /dev/null
```

It only uses standard C file functions: `fopen()`, `fread()` and `fclose()`.

### note on libc `getline()`

Then I implement checker option "-c" at next. 
It required **text file line reading**.
I choose the [`getline()`](http://man7.org/linux/man-pages/man3/getline.3.html) function in `<stdio.h>`.
But it is functions in **POSIX extension**; not available in some environments.

So I just try to implement `getline()` compatible functions at:

- [getline() with `fgets()` or with `fgetc()`](https://gist.github.com/bellbind/966082b01fc39fffc8853b668942c36c)

I tried both with `fgets()` version and with `fgetc()` version. 
The `fgets()` version would be faster than the `fgetc()` version but the code is tricky a little.

```c
static ssize_t
getline(char** restrict pline, size_t* restrict plsize, FILE* restrict fp) {
  if (*plsize < 2) *pline = realloc(*pline, *plsize = 128);
  if (feof(fp)) return -1;
  char* cur = *pline;
  size_t cursize = *plsize;
  for (;;) {
    char* ret = fgets(cur, cursize, fp);
    if (ret == NULL && !feof(fp)) return -1; //=> read error
    if (ret == NULL && cur == *pline) return -1; //=> read empty
    char* eod = memchr(cur, '\0', cursize);
    if (feof(fp) || eod[-1] == '\n') return eod - *pline;
    // line continued
    cursize = *plsize + 1; // last of *pline is '\0'
    *pline = realloc(*pline, *plsize *= 2);
    cur = *pline + *plsize - cursize;
  }
}
```

(I think `fgets()` would be better if returned the position of `'\0'`).

### note on the line format for sum file 

The code of parsing sum lines is complex for emulating coreutils error/warning messages.
The splitter of hex digest and file name is at most two, the first character allow  SPACE and TAB, but second one accepts only SPACE.
The file names started with SPACE areallowed for the splitter.

```
# blank lines and # comment lines allowed
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  /dev/null

# single space silitter is allowed
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855 /dev/null

# space allowed fron t of lines
    e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  /dev/null
```

Even if invalid formatted lines are exist, they do not effect the exit result value.


## CMakeFiles.txt for C programs

I added to a [cmake](https://cmake.org/) build script. To build binaries and libraries as:

```bash
$ git clone https://gist.github.com/bellbind/2fd66a09340942b9845ef432a5c8c419 sha256
$ cd sha256
$ mkdir build
$ cd build
$ cmake ..
$ make
$ ./sha256e /dev/null
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  /dev/null
```

To switch the compiler with cmake options as:

```bash
$ make clean
$ cmake -DCMAKE_C_COMPILER=gcc-7 ..
$ make
```

----
## Difference of other hash functions from SHA256

### SHA1 from SHA256

[SHA1](https://en.wikipedia.org/wiki/SHA-1) is:

- left cycric shift
- hash size: 5 x uint32
- loop steps: 80
- only 4 keys used used for each 20 loop steps
- loop step uses 4 diffrent non-linear function `f(b, c, d)` each 20 steps

### MD5 from SHA256

[MD5](https://en.wikipedia.org/wiki/MD5) is:

- little endian byte order
- left cycric shift
- hash size: 4 x uint32
- loop steps: 64
- chunk words are not mixed for values, indexes are mixed
- loop step uses 4 diffrent non-linear function `f(b, c, d)`, word index function `g(i)` and cycric shift amount `S[i]`for each 16 steps


### RIPEMD160 from SHA256

[RIPEMD160(aka RMD160)](https://en.wikipedia.org/wiki/RIPEMD) is:

- little endian byte order
- left cycric shift
- hash size: 5 x uint32
- chunk words are not mixed for values, indexes are mixed
- loop steps: 80
- two variable rotations in loop step: left side and right side
- only 5 keys used for each 16 loop steps
- on both side, loop step uses 5 diffrent non-linear function `f(b, c, d)`, word index function `g(i)` and cycric shift amount `S[i]`for each 16 steps
- hash vector also rotated for each chunk: `h[i] = h[(i + 1) % 5] + l[(i + 2) % 5] + r[(i + 3) % 5]`

### SHA512 from SHA256

[SHA512](https://en.wikipedia.org/wiki/SHA-2) is:

- expand uint64 operators
    - hash size: 8 x uint64
    - chunk size: 128 bytes
    - bit size: uint128 
    - bit shift amount is diffrent 
    - (value of initial vector and keys are different as uint64)
- loop steps: 80

----

## Use sha256 C implementation for emscripten (with WebAssemby)

The SHA256 C codes are also acceptable for [emscripten](https://github.com/kripken/emscripten) compile without any modifications.
After set up emscripten with [bynaryen](https://github.com/WebAssembly/binaryen),  for compiling with WebAssembly target as:

```bash
emcc -std=c11 -pedantic -O3 -Wall -Wextra sha256extracted.c -o sha256.js -s WASM=1 -s EXPORTED_FUNCTIONS='["_sha256_init","_sha256_update","_sha256_digest"]'
```

It generates as emscripten module "sha256.js" and "sha256.wasm". 
But it is not friendly for usual JavaScript programming because of:

- asynchronous module initialization existed (it would not use module synchromously as in node.js)
- memory allocation and releasing required: use `let ptr = m._malloc(s)` and `m._free(ptr)`
- function params only integer. arrays and structs put in memory and pass its address values
- total memory size is fixed at compile time: default is 256 * 64k = 16M

So usually, the emscripten module is wrapped with more user friendly module code.
(But it cannot avoid async initialization and resource releasing entirely).

The wrapper module: `sha256mod.js` is:

- [repoistory on gist](https://gist.github.com/bellbind/de02a751e52ab3a3d35e843221d3d37d)

It is a basic example for using emscripten modules.

The repository includes:

- main code "sha256main.js": outputs hash sums and checking sum files with "-c" option.
- package.json for building with `npm install`: emscripten embironments required
- run with [`npx`](https://www.npmjs.com/package/npx) as:

```bash
$ npx https://gist.github.com/bellbind/de02a751e52ab3a3d35e843221d3d37d /dev/null
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  /dev/null
```

Execution time for 512MB data as:

|                       | time |
|-----------------------|-----:|
|emscripten module      |4.854s|
|nodejs crypto module   |1.777s|
|pure ECMAScript module |1m10.004s|
|(coreutils sha256sum)  |3.2007s|


