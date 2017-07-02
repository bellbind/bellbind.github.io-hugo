---
title: "Comparing WebAssembly performance with asm.js and plain ES6"
slug: "compare-wasm"
date: 2017-07-03T01:45:18+09:00
categories: ["2017-07"]
tags: ["WebAssemby", "asm.js", "ES6"]
draft: false
---

For comparing the performance of WebAssembly runtimes with others,
I published on-browser programs with the same algorithm 
as "plain ES6", "asm.js" and "WebAssembly" **without emscripten**.

<!--more-->

## The example: Turing-Pattern cell automaton

["Turing Pattern"](https://www.google.com/search?q=Turing+Pattern) is a chemical model for forming patterns of animals
as a survival between activators and inhibitators.
A simple generator of the turing pattern is a cell automaton that
each cell is also activator and inhibitator for neighbor cellss.

- cell[t+1] = cell[t] + activators[t] - inhibitators[t]

The effect of activators and inhibitators is just gathering value of neighbor cells,
but the effect as inhibitator is wider than that of activator.

## Demo

| version      | link                                                                                                       |
|:------------:|:-----------------------------------------------------------------------------------------------------------|
| Plain ES6    | [host on githack](https://gist.githack.com/bellbind/dd1c0cd9cbe422caff8dcdae1010ad37/raw/index-es6.html)   |
| asm.js       | [host on githack](https://gist.githack.com/bellbind/dd1c0cd9cbe422caff8dcdae1010ad37/raw/index-asm.html)   |
| WebAssembly  | [host on githack](https://gist.githack.com/bellbind/dd1c0cd9cbe422caff8dcdae1010ad37/raw/index-wasm.html)  |
|              |                                                                                                            |
|(Source Codes)| [gist:dd1c0cd9cbe422caff8dcdae1010ad37](https://gist.github.com/bellbind/dd1c0cd9cbe422caff8dcdae1010ad37) |

The programs embed the benchmark with `console.time()/timeEnd()` that prints execution times(ms) on  "Web Console".

## Notes on plain ES6 version

- [script-es6.js](https://gist.github.com/bellbind/dd1c0cd9cbe422caff8dcdae1010ad37#file-script-es6-js)

The convolution representation is a list of `{x, y, f}` tuple, not an usual number matrix. 
The `f` ia a factor value. The `x` and` y` is a relative offsets as `-r` to `r`.
It can apply to linear arrays with `map()` and `reduce()` as:

```js
// mat[y * w + x] = v
function get(mat, x, y) {
    return return mat[((y + w) % w) * w + ((x + w) % w)];
}

const result = mat.map((v, i) => {
    const mx = i % w, my = i / w | 0;
    return conv.reduce((s, {x, y, f}) => s + f * get(mat, mx + x, my + y), 0);
});
```

Two convolution are used as activator and inhibitator applied in each step.
These convolutions forms as circle effect like a force.

## Notes on asm.js version

- [conv.asm.js](https://gist.github.com/bellbind/dd1c0cd9cbe422caff8dcdae1010ad37#file-conv-asm-js)
- [script-asm.js](https://gist.github.com/bellbind/dd1c0cd9cbe422caff8dcdae1010ad37#file-script-asm-js)

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

The asm.js module is loaded with `fetch()` and eval with `Function()`.
Applying convolution with the exported `conv(coffs, clen, soffs, len, w, doffs)` function as:

```js
// cpnv: [int32, int32, float64] strides as Float64Array
// cells, result:  mat[y * w + x] as Float64Array
conv(conv.byteOffset, conv.length / 2, cells.byteOffset, cells.length, w, result.byteOffset);
```

### Notes on asm.js programming

I used the [dherman/asm.js](https://github.com/dherman/asm.js) as the syntax checker.
But the "asmjs" command does not accept bynarien's accpetable asm.js files
(only accpets commonsjs module style files which rejected by binaryen's "asm2wasm" command).

I made a little modification for accepting non cjs module style:

- https://github.com/bellbind/asm.js

## Notes on WebAssembly version

- [conv.wast](https://gist.github.com/bellbind/dd1c0cd9cbe422caff8dcdae1010ad37#file-conv-wast)
- [script-wasm.js](https://gist.github.com/bellbind/dd1c0cd9cbe422caff8dcdae1010ad37#file-script-wasm-js)

I use [binaryen](https://github.com/WebAssembly/binaryen)'s "asm2wasm" command for making the WebAssembly version.
The "asm2wams" generated function bodies seem as direct translation of the "conv.asm.js" program.
But not directly used the "asm2wasm" generated code, I applyed a little optimization as:

- Remove all `import` section, and add `export` `memory`
- Remove generated `$i32u-rem` `func`, and replace its call part with `i32.rem_u` operation in `$get` and `$conv` bodies.
- Remove generated `$i32u-div` `func`, and replace its call part with `i32.div_u` operation in `$conv` body.

Complete diff:

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
>     (i32.rem_u
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
>       (i32.rem_u
98c72
<       (call $i32u-div
---
>       (i32.div_u
```

## Benchmark result

- Browser: firefox-54.0.1 and chrome-59.0.3071.115 on macOS 10.12.5
- Hardware: MacBook Pro (13-inch, Late 2016, Two Thunderbolt 3 ports)

| version     |firefox-54|chrome-59|
|-------------|---------:|--------:|
| Plain ES6   | 70ms     | 240ms   |
| WebAssembly | 80ms     | 130ms   |
| asm.js      | 80ms     | 210ms   |

