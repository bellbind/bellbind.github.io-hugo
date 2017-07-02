---
title: "Comparing WebAssembly with asm.js and pure ES6"
slug: "compare-wasm"
date: 2017-07-03T01:45:18+09:00
categories: ["2017-07"]
tags: []
draft: false
---

For comparing the performance of WebAssembly runtimes with others,
I published on-browser programs with the same algorithm 
as "pure ES6", "asm.js" and "WebAssembly" **without emscripten**.

<!--more-->

## The example: Turing-Pattern cell automaton

"Turing Pattern" is a chemical model for forming patterns
as a survival between activators and inhibitators.
A simple generator of the turing pattern is a cell automaton that
each cell is also activator and inhibitator for neighbor cellss.

- cell[t+1] = cell[t] + activators[t] - inhibitators[t]

The effect of activators and inhibitators is just gathering value of neighbor cells,
but the effect as inhibitator is wider than that of activator.

## Active demo

- Source Codes: [gist:dd1c0cd9cbe422caff8dcdae1010ad37](https://gist.github.com/bellbind/dd1c0cd9cbe422caff8dcdae1010ad37)
- Pure ES6 version: [host on githack](https://gist.githack.com/bellbind/dd1c0cd9cbe422caff8dcdae1010ad37/raw/index-es6.html)
- asm.js version: [host on githack](https://gist.githack.com/bellbind/dd1c0cd9cbe422caff8dcdae1010ad37/raw/index-asm.html)
- WebAssembly version: [host on githack](https://gist.githack.com/bellbind/dd1c0cd9cbe422caff8dcdae1010ad37/raw/index-wasm.html)

The programs embed the benchmark with `console.time()/timeEnd()` that prints execution times(ms) on  "Web Console".

## Noten on pure ES6 version

The convolution data is a list of `{x, y, f}` tuple, not an usual number matrix. 
The `f` ia a factor value. The `x` and` y` is a relative offsets as `-r` to `r`.
It can apply linear arrays with `map()` and `reduce()` as:

```js
// mat[y * w + x] = v
function get(mat, x, y) {
    return return mat[((y + w) % w) * w + ((x + w) % w)];
}

const result = mat.map((v, i) => {
    const x = i % w, y = i / w | 0;
    return conv.reduce((s, c) => s + c.f * get(mat, x + c.x, y + c.y), 0);
});
```

Two convolution are used as activator and inhibitator applied in each step.
These convolutions forms as circle effect like a force.

## Noten on asm.js version

For asm.js, the list of tuple of `{x, y, f}`  turns `ArrayBuffer` as strides of `[int32, int32, float64]` as:

```js
function convert(conv) {
    const buf = new ArrayBuffer(conv.length * 16);
    const bufF = new Float64Array(buf);
    const bufI = new Int32Array(buf);
    conv.forEach(({x, y, f}, i) => {
        bufI[i * 4] = x;
        bufI[i * 4 + 1] = y;
        bufF[i * 2 + 1] = f;
    });
    return bufF;
}
```

The hand written asm.js module as:

{{<gist bellbind dd1c0cd9cbe422caff8dcdae1010ad37 "conv.asm.js">}}

The module is loaded with `fetch()` and eval with `Function()`.
Applying convotion with the exported `conv(coffs, clen, soffs, len, w, doffs)` function as:

```js
// cA: [int32, int32, float64] strides as Float64Array
// sF, dA:  mat[y * w + x]  as Float64Array
conv(cA.byteOffset, cA.length / 2, sF.byteOffset, sF.length, w, dA.byteOffset);
```

### Note on asm.js programming

I used the [dherman/asm.js](https://github.com/dherman/asm.js) as the syntax checker.
But the "asmjs" command does not accept bynarien's accpetable asm.js files
(only accpets commonsjs module style files which rejected by binaryen's "asm2wasm" command).

I made a little modification for accpeting non cjs module style as:

- https://github.com/bellbind/asm.js

## Note on WebAssembly version

I use [binaryen](https://github.com/WebAssembly/binaryen)'s "asm2wasm" command for making the WebAssembly version.
The "asm2wams" generated function bodies seems direct translation of the "conv.asm.js" program.
But not directly used the "asm2wasm" generated code, I applyed a little optimization as:

- Remove all `import` section, and add `export` `memory`
- Remove generated `$i32u-rem` `func`, and replace its call part with `i32.rem_u` operation in `$get` and `$conv` bodies.
- Remove generated `$i32u-div` `func`, and replace its call part with `i32.div_u` operation in `$conv` body.

as:

```bash
$ diff conv-asm.wast  conv.wast
2,5c2,3
<  (import "env" "memory" (memory $0 256 256))
<  (import "env" "table" (table 0 0 anyfunc))
<  (import "env" "memoryBase" (global $memoryBase i32))
<  (import "env" "tableBase" (global $tableBase i32))
---
>  (export "heap" (memory $0))
>  (memory $0 1 256)
7,18d4
<  (func $i32u-rem (param $0 i32) (param $1 i32) (result i32)
<   (if (result i32)
<    (i32.eqz
<     (get_local $1)
<    )
<    (i32.const 0)
<    (i32.rem_u
<     (get_local $0)
<     (get_local $1)
<    )
<   )
<  )
24c10
<      (call $i32u-rem
---
>      (i32.rem_u
33c19
<     (call $i32u-rem
---
>     ($i32.rem_u
54,65d39
<  (func $i32u-div (param $0 i32) (param $1 i32) (result i32)
<   (if (result i32)
<    (i32.eqz
<     (get_local $1)
<    )
<    (i32.const 0)
<    (i32.div_u
<     (get_local $0)
<     (get_local $1)
<    )
<   )
<  )
92c66
<       (call $i32u-rem
---
>       ($i32.rem_u
98c72
<       (call $i32u-div
---
>       ($i32.div_u
```

## Benchmark result

- firefox-54.0.1 and chrome-59.0.3071.115 on macOS 10.12.5: MacBook Pro (13-inch, Late 2016, Two Thunderbolt 3 ports)

|      |firfox-54|chrome-59|
|------|--------:|--------:|
|ES6   | 70ms    | 240ms   |
|wasm  | 80ms    | 130ms   |
|asm.js| 80ms    | 210ms   |




