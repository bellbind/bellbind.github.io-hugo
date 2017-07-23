---
title: "Implement Hash Functions In WebAssembly Text Format"
slug: "hash-functions-in-wast"
date: 2017-07-23T03:51:27+09:00
categories: ["2017-07"]
tags: ["cryptography", "node.js", "webassembly"]
draft: false
---

[The emscripten module from C SHA256 code](/2017/07/16/hash-functions-with-c/)
is much faster than [the pure ES6 SHA256 code](/2017/07/08/implement-sha256-ecmascript/).
But the emxcripten module required async initialization and explicit cleanup calls.
In this article, Implement SHA256 in WebAssembly directly (with no emscripten).

- [SHA256 in WAST](https://gist.github.com/bellbind/9b73b3965156f167cc0e500cdefb8c8a)
- [SHA512 in WAST](https://gist.github.com/bellbind/825e0b6f6d0f1e428a7fd98346f4ae82)

<!--more-->

## Use `WebAssembly.Instance` as an object (not module)

In node.js, usually modules are provided as  an object factory, not object itself.
In emscripten with WebAssembly, it also uses `WebAssembly.Instance` as an object factory.
A `WebAssembly.Instance` object has a memory(`WebAssembly.Memory`), 
and the memory is shared for objects used the memory of `WebAssembly.Instance` with `_malloc()` and `_free()`. 
It is the reason of explicit cleanup calls required.

For removing the resource managements, use `WebAssembly.Instance` as an object.
make wrapper module of WebAssembly for node.js as:

```js
// sha256.js
const fs = require("fs");

const wasm = fs.readFileSync(`${__dirname}/sha256.wasm`);
const sha256m = new WebAssembly.Module(wasm);

class sha256 {
    constructor() {
        this.hash = new WebAssembly.Instance(sha256m).exports;
    }
    update(buffer) {
        //...
    }
    digest() {
        //...
    }
}
```

The memory in `WebAssembly.Instance` is not shared. It just clean up when the wrapper object disappeared.

Note that the files in the directory as:

- main.js: for command line (do similar outputs as coreutils sha256sum)
- sha256.js: wrapper nodejs module (provide same interface of crypto's hash object)
- sha256.wasm: WebAssembly binary format module


## How to write a WebAssembly Text Format program directly

Error messages of the binaryen's "wasm-as"  are difficult to understand what is wrong.
Some strategies to write the unstable thingare:

- Write each small codes to check everytime
- Write code parts as splitted small `func`s at first, then embed into the bodies.
- Export the splitted `func`s to check them on node.js REPL.

The compiler specification is just WebAssembly's minimum viable products(MVP), not match 
[the online specifications](http://webassembly.org/docs/high-level-goals/).
The limitations of MVP are:

- `global` decls is only for constant numbers, `set_global` operator is disabled. 
- `local` decls are only allowed in `func` head parts
- memory address type is `i32` only
- bit not operator is not existed, use `xor` with `-1`
    - boolean not operator is `eqz`

## Writing a chunk `update()` function

The first version is only `update()` function for a 64byte chunk in a memory.
It required to pass the initial vector values on the memory from JavaScript side.
It can check the result to success pattern (pass a zero data chunk then "hello world" chunk) in each step.

The memory layout is:

- hash vector address: 0-31
- chunk address: 32-95

I wrote the `update()` as loop extracted version from the start.
It is difficult to write with hand, so I write code generator with JavaScript.
The computation functions "to bigendien", "w[i >= 16]" part, "t1", "t2" is made as `func` formerly.

The code generator ("sha256update-call.js") just use them as:

```js
const setW16 = range64.slice(0, 16).map(i => `(set_local $w${i} (call $big32 (i32.load offset=32 align=4 (i32.const ${i * 4}))))`);
```

```js
function genW(i) {
    return `(set_local $w${i} (call $wn (get_local $w${i - 16}) (get_local $w${i - 15} )(get_local $w${i - 7}) (get_local $w${i - 2})))`;
}
```

```js
function genStep(i, a, b, c, d, e, f, g, h) {
    return `;; step ${i}
(set_local $t (call $t1 (get_local $${e}) (get_local $${f}) (get_local $${g}) (get_local $${h}) (get_local $w${i}) (i32.const 0x${K[i].toString(16).padStart(8, "0")})))
(set_local $${d} (i32.add (get_local $${d}) (get_local $t)))
(set_local $${h} (i32.add (get_local $t) (call $t2 (get_local $${a}) (get_local $${b}) (get_local $${c}))))`;
}
```

Don't confuse template string placement's `$` and WAST name's `$`. For example, `$${f}` turns `$a` when `f = "a"`. 

The computation functions are directly written in WAST file "sha256noupdate.wast". 
The "sha256noupdate.wast" also has  `(func $update)` as empty function,
then it can compile with "wasm-as" standaalone and can check sub functions direct via `export`.
The code generator replace the generated `(func $update ...)` to print.

For example `func $wn` as "w[i >= 16]" is 
just `w16 + (rotr(w15, 7) ^ rotr(w15, 18) ^ (w15 >>> 3)) + w7 + (rotr(w2, 17) ^ rotr(w2, 19) ^ (w2 >>> 10))`.
It grows on WAST as:

```scm
 (func $wn (param $w16 i32) (param $w15 i32) (param $w7 i32) (param $w2 i32) (result i32)
       (return (i32.add (i32.add (i32.add (get_local $w16)
                                          (i32.xor (i32.xor (i32.rotr (get_local $w15) (i32.const 7))
                                                            (i32.rotr (get_local $w15) (i32.const 18)))
                                                   (i32.shr_u (get_local $w15) (i32.const 3))))
                                 (get_local $w7))
                        (i32.xor (i32.xor (i32.rotr (get_local $w2) (i32.const 17))
                                          (i32.rotr (get_local $w2) (i32.const 19)))
                                 (i32.shr_u (get_local $w2) (i32.const 10))))))
```

Computation to big endien by filter cascadings as:

```scm
 (func $big32 (param $v i32) (result i32)
       (set_local $v (i32.or
                        (i32.shr_u (i32.and (get_local $v) (i32.const 0xff00ff00)) (i32.const 8))
                        (i32.shl   (i32.and (get_local $v) (i32.const 0x00ff00ff)) (i32.const 8))
                        ))
       (return (i32.or
                        (i32.shr_u (i32.and (get_local $v) (i32.const 0xffff0000)) (i32.const 16))
                        (i32.shl   (i32.and (get_local $v) (i32.const 0x0000ffff)) (i32.const 16))
                        )))
```

Note on generating "sha256.wasm" file on command line as:

```bash
$ node sha256update-embed.js > sha256.wast
$ wasm-as sha256.wast
```

## Add `start` func

Initialization call is exist in WebAssembly as `start` declaration, set initial vector as:

```scm
 (start $init)
 ;; memory layout: h[0..31], chunk[32..95]
 (func $init
       (i32.store align=4 (i32.const 0) (i32.const 0x6a09e667))
       (i32.store align=4 (i32.const 4) (i32.const 0xbb67ae85))
       (i32.store align=4 (i32.const 8) (i32.const 0x3c6ef372))
       (i32.store align=4 (i32.const 12) (i32.const 0xa54ff53a))
       (i32.store align=4 (i32.const 16) (i32.const 0x510e527f))
       (i32.store align=4 (i32.const 20) (i32.const 0x9b05688c))
       (i32.store align=4 (i32.const 24) (i32.const 0x1f83d9ab))
       (i32.store align=4 (i32.const 28) (i32.const 0x5be0cd19))
       )
```

## Add `final(filled, bytesl, bytesh)` to generate result hash value

The finishing process of SHA256 is:

- embedding pad and bit length to chunk
- update last chunk
- turn hash vector as big-endian order

then get memory in JavaScript side.

As SHA256 specification, bit length is 64bit data, so exported funcation would be passed length with two 32bit data.
The other parameter is chunk filled size. WHole code of `final(filled, bytesl, byteh)` is:

```scm
 (func $final (param $filled i32) (param $bytesl i32) (param $bytesh i32)
       (i32.store8_u offset=32 (get_local $filled) (i32.const 0x80))
       (set_local $filled (i32.add (get_local $filled) (i32.const 1)))
       (if (i32.gt_u (get_local $filled) (i32.const 56))
           (block
            (call $memset (get_local $filled) (i32.const 0) (i32.const 64))
            (call $update)
            (set_local $filled (i32.const 0))
            ))
       (call $memset (get_local $filled) (i32.const 0) (i32.const 56))
       (i64.store offset=32 align=8 (i32.const 56) (call $big64
                                       (i64.or (i64.shl (i64.extend_u/i32 (get_local $bytesh)) (i64.const 35))
                                               (i64.shl (i64.extend_u/i32 (get_local $bytesl)) (i64.const 3)))
                                       ))
       (call $update)
       
       (i32.store align=4 (i32.const  0) (call $big32 (i32.load align=4 (i32.const 0))))
       (i32.store align=4 (i32.const  4) (call $big32 (i32.load align=4 (i32.const 4))))
       (i32.store align=4 (i32.const  8) (call $big32 (i32.load align=4 (i32.const 8))))
       (i32.store align=4 (i32.const 12) (call $big32 (i32.load align=4 (i32.const 12))))
       (i32.store align=4 (i32.const 16) (call $big32 (i32.load align=4 (i32.const 16))))
       (i32.store align=4 (i32.const 20) (call $big32 (i32.load align=4 (i32.const 20))))
       (i32.store align=4 (i32.const 24) (call $big32 (i32.load align=4 (i32.const 24))))
       (i32.store align=4 (i32.const 28) (call $big32 (i32.load align=4 (i32.const 28))))
       )
```

`memset(start, value, end)` is simply loops to set each bytes `while (start != end) *start++ = value;` as:

```scm
 ;; start and end is relative address at chunk offset
 (func $memset (param $start i32) (param $v8 i32) (param $end i32)
       (loop $loop (if (i32.eq (get_local $start) (get_local $end)) (return))
              (i32.store8_u offset=32 (get_local $start) (get_local $v8))
              (set_local $start (i32.add (get_local $start) (i32.const 1)))
              (br $loop)))
```

then the WAST program fullly implemented SHA256 functionality.

## Raise more speed

The minimum wasm module become fast from [the pure ES6 code](/2017/07/08/implement-sha256-ecmascript/), 
but yet slower than [the emscripten module](/2017/07/16/hash-functions-with-c/)).
For aquiring more speed,  several changes applied.

### Change function calls to inline code

In WebAssembly, cost of function calls is relatively expensive.
Replacing function calls to their code bodies raise runtime speed.

The "sha267update-embed.js" is changed from the "sha256update-call.js" with no function calls.
That reduce 30% of execution time. But the wasm file size is grown double.

### Reduce local variables

In SHA256, chunk mixed array is not required 64 when calculate mixing in each step.
Set local w0 to w15, then reuse w(i % 16) variables in each steps.

The "sha256update-reduce-call.js" is generator applied the local variables reducing.
But the execution speed is almost same as "sha256update-embed.js" version.

### Move chunking process inside WebAssembly

Not only `update()` 64 bytes in WebAssembly, the process chunking to 64bytes to call `update()` is moved into WebAssembly.
In node.js, buffer size from reading file is 64KB, it exists thousand time JS-to-WAMS overheads.

New `update(byteOffset, bytelength)` to accept any size of buffers is added to "sha256noupdate.wast".
Digest final call changes as `final()`.
That means managing bytesize and chunk filled size inside module.

The memory layout extends as:

- hash: 0-31 (4-uint32)
- chunk: 32-95 (64-byte)
- filled: 96-99 (int32)
- bytes: 104-111 (uint64)
- buffer: 112-...

Note that every memory addresses would be aligned 8bytes; start address with multiple of 8.

`update(offsetm length)` as named `$updateBuf' inside wasm as;

```scm
 (func $updateBuf (param $offs i32) (param $bytes i32)
       (local $filled i32)
       (local $copyend i32)
       (set_local $filled (i32.load offset=96 align=4 (i32.const 0)))
       (if (i32.lt_s (get_local $filled) (i32.const 0)) (return))
       (i64.store offset=104 align=8 (i32.const 0) (i64.add (i64.load offset=104 align=8 (i32.const 0)) (i64.extend_u (get_local $bytes))))
       
       (loop $loop
             (if (i32.lt_u (get_local $bytes) (i32.sub (i32.const 64) (get_local $filled)))
                 (set_local $copyend (i32.add (get_local $filled) (get_local $bytes)))
                 (set_local $copyend (i32.const 64)))
             (call $memcpy (get_local $filled) (get_local $offs) (get_local $copyend))
             (if (i32.ne (get_local $copyend) (i32.const 64))
                 (block
                  (i32.store offset=96 align=4 (i32.const 0) (get_local $copyend))
                  (return)))
             (call $update)
             (set_local $offs (i32.add (get_local $offs) (i32.sub (i32.const 64) (get_local $filled))))
             (set_local $bytes (i32.sub (get_local $bytes) (i32.sub (i32.const 64) (get_local $filled))))
             (set_local $filled (i32.const 0))
             (br $loop)
             )
       )
```

the psudocode of "$updateBuf" is:

```c
void updateBuf(uint32 offs, uint32 bytes) {
  int32 filled = memory[96];
  if (filled < 0) return;
  memory[112] += bytes;
  
  for (;;) {
    uint32 copyend = bytes < 64 - filled ? filled - bytes : 64;
    memcpy(filled, offs, copyend);
    if (copyend < 64) {
       memory[96] = copyend;
       return;
    }
    update();
    offs += 64 - filled;
    bytes -= 64 - filled;
    filled = 0;
  }
}
```

`memcpy(start, src, end)` code is `while (start != end) *start++ = *src++;` as: 

```scm
 ;; simple memcpy(start, src, end)
 (func $memcpy (param $start i32) (param $src i32) (param $end i32)
       (loop $loop (if (i32.eq (get_local $start) (get_local $end)) (return))
              (i32.store8_u offset=32 (get_local $start) (i32.load_u (get_local $src)))
              (set_local $start (i32.add (get_local $start) (i32.const 1)))
              (set_local $src (i32.add (get_local $src) (i32.const 1)))
              (br $loop)))
```

This change drastically reduce execution time for large data.
In this step, the code gets faster than the emscripten version.

### Apply alignment copy

The above `memcpy` is copy in each bytes. 
But in WebAssembly, it can also access as 64bit
that would reduce times of memory access.

To apply them, simply check the both `start` and `src` address started x8, use `i64`  memory access.

```scm
(func $memcpy (param $start i32) (param $src i32) (param $end i32)
       (local $end8 i32)
       (if (i32.and (i32.eqz (i32.and (get_local $start) (i32.const 0x7))) (i32.eqz (i32.and (get_local $src) (i32.const 0x7))))
           (block $loop8-out
                  (set_local $end8 (i32.sub (get_local $end) (i32.const 8)))
                  (loop $loop8 (if (i32.ge_s (get_local $start) (get_local $end8)) (br $loop8-out))
                        (i64.store offset=32 align=8 (get_local $start) (i64.load align=8 (get_local $src)))
                        (set_local $start (i32.add (get_local $start) (i32.const 8)))
                        (set_local $src (i32.add (get_local $src) (i32.const 8)))
                        (br $loop8))
                  ))
       ;; remain as same loop
       (loop $loop (if (i32.eq (get_local $start) (get_local $end)) (return))
              (i32.store8_u offset=32 (get_local $start) (i32.load_u (get_local $src)))
              (set_local $start (i32.add (get_local $start) (i32.const 1)))
              (set_local $src (i32.add (get_local $src) (i32.const 1)))
              (br $loop)))
```

The pseudocode is:

```c
void memcpy(uint32 start, uint32 src, uint32 end) {
  if ((start & 0x7) == 0 && (src & 0x7) == 0) {
    for (int32 end8 = end - 8; start <= end8; start += 8, src += 8) {
      *((uint64) start) = *((uint64) src);
    }
  }
  while (start != end) *start++ = *src++;
}
```

Compare than the simple loop, the alignment access reduced 20% of execution time for large data.

## About SHA512

SHA512 is extended version as SHA256 with  32bit to 64bit.
In WAST file, just replacing `i32` operations to `i64`s and every constants.
The memory of data total bytes splited two 64bit area.

- hash: 0-63 (4-uint64)
- chunk: 64-191 (128-byte)
- filled: 192-195 (int32)
- bytesl: 200-207 (uint64)
- bytesh: 208-215 (uint64)
- buffer: 216-...

## npx friendly module

From node.js-8.2, "npx" command is bundled.
npx can execute npm package "bin" command directly from git urls.

So, the `package.json` to the gist repositories for npx are added:

- SHA256: https://gist.github.com/bellbind/9b73b3965156f167cc0e500cdefb8c8a
- SHA512: https://gist.github.com/bellbind/825e0b6f6d0f1e428a7fd98346f4ae82

Run as:

```bash
$ npx https://gist.github.com/bellbind/9b73b3965156f167cc0e500cdefb8c8a /dev/null
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855 /dev/null

$ npx https://gist.github.com/bellbind/825e0b6f6d0f1e428a7fd98346f4ae82 /dev/null
cf83e1357eefb8bdf1542850d66d8007d620e4050b5715dc83f4a921d36ce9ce47d0d13c5d85f2b0ff8318d2877eec2f63b931bd47417a81a538327af927da3e /dev/null
```

In the `package.json`, generate complete wast file and compile with "wasm-as" as:

```json
{
    "name": "sha256w",
    "version": "1.0.0",
    "main": "sha256.js",
    "bin": "main.js",
    "scripts": {
        "preinstall": "echo 'This module required binaryen wasm-as command'",
        "install": "node sha256update-reduce-local.js > sha256.wast && wasm-as sha256.wast"
    }
}
```
