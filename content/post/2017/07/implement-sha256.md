---
title: "Implement SHA256 algorithm in pure ECMAScript"
slug: "implement-sha256-ecmascript"
date: 2017-07-08T12:52:21+09:00
categories: ["2017-07"]
tags: ["ES6", "JavaScript", "algorithm", "cryptography"]
draft: false
---

How to implement SHA256 hash digest algorithmn in **pure ECMAScript**;
with no platform(node.js, modern browsers) dependent APIs.

The running code is in 
[gist:bellbind/1cfcc96514099fcebd455259b7249525](https://gist.github.com/bellbind/1cfcc96514099fcebd455259b7249525).

<!--more-->

## What is pure ECMAScript

The pure ECMAScript is using the functions only in the current 
[ECMAScript specification](https://www.ecma-international.org/ecma-262/8.0/index.html)
which includes 
[TypedArray feature](https://www.ecma-international.org/ecma-262/8.0/index.html#sec-typedarray-objects).

The only exception is "main" entry point for the file.
These dependent scripts put in the guards as node.js module like:

```js
if (typeof require === "function" && require.main === module) {
    // node.js entry point script
} else if (typeof module !== "undefined") {
    // node.js module exports script
}
```

at the bottom of script files.

---

## Overview of SHA256 algorithm

SHA256 is a well-known hash digest algorithm. The wikipedia 
[SHA-2 page](https://en.wikipedia.org/wiki/SHA-2) has entire SHA256 algorithm as 
[a pseudocode](https://en.wikipedia.org/wiki/SHA-2#Pseudocode).

SHA256 is base on
[Merkle–Damgård hash function](https://en.wikipedia.org/wiki/Merkle%E2%80%93Damg%C3%A5rd_construction)
framework as:

- Digest process
    - Start with Initialization Vector(IV)
    - For each fixed length message chunk, mix it into IV
    - Final IV data as a result hash digest
- Padding message with its length value
    - Pad the message as its length become multiple of the chunk size.
    - Mark a bit `1` as a end of the message, then pad `0` with the last chunk tail:
      the mark become `0x80` when the message is a byte stream
    - Embed the message "bit" length value in the end of the last chunk
    - Hashing chunks become: `[message, 0x80, 0, ..., 0, message length as binary data]`

Then the SHA256 applies:

- Unsigned 32-bit integer(= uint32) arithmetics: cyclic shift, logical shift, and or not xor, add
- [Big Endien byte-order](https://en.wikipedia.org/wiki/Endianness#Big-endian):
  between bytes and unsigned integer (bit-order is not applyed)
- Handle 256-bit (= 32-byte = 8-uint32) vector

The constants are 
[no-backdoor random number like values](https://en.wikipedia.org/wiki/Nothing_up_my_sleeve_number) as:

- IV: top 32-bit pattern of fractional part of `sqrt(p)`s on prime number sequence(p=2,3,5,7,11,...)
- Key: top 32-bit pattern of fractional part of `cbrt(p)`s on prime number sequence(p=2,3,5,7,11,...)

---

## Basic Knowledge for JavaScript

### `sha256` API design: as node.js hash functions

I applied similar interface with [node.js's hash object](https://nodejs.org/api/crypto.html#crypto_class_hash) as:

- `hash = new sha256()`: returns the initial hash context.
- `hash.update(fragment)`: process any size of `ArrayBuffer` like data fragment: 
  (also includes node.js's `Buffer`). It returns `this` hash context.
- `digestbuffer = hash.digest()`: finalize processing then return hash digest `ArrayBuffer`
  (node.js's encoding feature is omitted)

The basic example is:

```js
const sha256 = require("./sha256");
const data = Buffer.from("Hello World");
const digest = new sha256().update(data).digest();
console.log(Buffer.from(digest).toString("hex"));
```

or stream style as:

```js
const hash = new sha256();
stream.on("read", (err, buf) => {
  if (!err) hash.update(buf);
});
stream.once("end", () => {
  const digest = hash.digest();
  console.log(Buffer.from(digest).toString("hex"));
});
```

### ECMAScript programming: cast to unsigned 32bit integer

The logical right shift `>>>` returns the value as "uint32" (
[spec](https://www.ecma-international.org/ecma-262/8.0/index.html#sec-unsigned-right-shift-operator)
). Espacially `>>> 0` just convert `uint32`.

```js
-1 >>> 0 //== 4294967295 as UINT32_MAX
```

Put it every point on maybe overflow int32 operations: `a + b`, `~a`, `a << n`, ....

### ECMAScript programming: `TypedArray`

In the program,  `ArrayBuffer`, `Uint8Array`, `Uint32Array` and `DataView` would be used.

`ArrayBuffer` does not directly access its content data.
Accessing the content, use `Uint8Array`, `Uint32Array` and `DataView` as a slice view of `ArrayBuffer`.
They have the `byteOffset`, `byteLength` and `buffer` properties for location the slice.

- `ab = new ArrayBuffer(byteLength=0)`
    - data is initialized `0`
- `u8a = new Uint8Array(arraybuffer, byteOffset=0, length=arraybuffer.byteLength - byteOffset)`
    - note: `length` is a count of numbers (not `byteLength)`
    - note: `byteOffset` should not be negative offset
- `u32a = new Uint32Array(arraybuffer, byteOffset=0, length)`
    - note1: `byteOffset` should be multiply of `4` (== `Uint32Array.BYTES_PER_ELEMENT`)
    - note2: when omit `length` param, `buffer.byteLength - byteOffset` should be multiply of `4`
- `u32a = new Uint32Array(length)`
    - all element is initialized `0`
- `u32a = Uint32Array.from(arrayOrIter, mapfn=v=>v)`
    - note: convert from standard `Array`, `TypedArray.from()` would be prefereable.
- `u8a[i] = n` set the numnber `n` cast as uint8 value (same as `v & 0xff`).
- `u32a[i] = n` set the number `n` cast as uint32 value (same as `v >>> 0`).
- `u8a.set(u8a2, start)`: copy `u8a2` values in `u8a` from `start` offset
    - note: thrown range error when `u8a2.length` is larger than `u8a.length - start`
- `u8a.fill(n, start=0, end=this.length)`: set all element as `n` between from `start` offset to `end - 1` offset
    - note: negative offsets allowed
- `u8a2 = u8a.subarray(start=0, end=this.length)`: make new relative slice view from `start` to `end` offset
    - note: negative offsets allowed
    - usually make the subarray for applying to `set()` above
- `u8a2 = u8a.slice(start=0, end=this.length)`: make new relative slice with copying from `start` to `end` offset
    - note: negative offsets allowed
 
TypedArray is just a slice, don't forget `byteOffset` and `byteLength` to converting the element types:

```
// GOOD
const u32a = new Uint32Array(u8a.buffer, u8a.byteOffset, u8a.byteLength);

// BAD: the byteOffset become 0 and the byteLength become u8a.buffer.byteLength
const u32a = new Uint32Array(u8a.buffer);
```

`Uint32Array` is limited  align x4 slice and can access numbers as "Little Endian" byte-order
(`SetValueInBuffer()`'s last `isLittleEndien` param is `true` at 
[the spec of `IntegerIndexedElementSet()`](https://www.ecma-international.org/ecma-262/8.0/index.html#sec-integerindexedelementset) ).

To use arbitrary byte offsets and **"Big Endian"** byte order, use `DataView` slice.

- `dv = new DataView(arraybuffer, byteOffset=0, byteLengtharraybuffer.byteLength - byteOffset)`
    - note: `byteOffset` should not be negative offset
- `uint32 = dv.getUint32(byteOffset, isLittleEndian=false)`
    - note: thrown range error when `dv.byteLength - byteOffset` less than 4
- `dv.setUint32(byteOffset, uint32, isLittleEndian=false)`

---

## Implementation of SHA256 algorithm

### Prepare the constants

Usually preset the constant values in the source code for runtime performance.

```js
const K = Uint32Array.from([
    0x428a2f98, 0x71374491, 0xb5c0fbcf, 0xe9b5dba5,
    0x3956c25b, 0x59f111f1, 0x923f82a4, 0xab1c5ed5,
    0xd807aa98, 0x12835b01, 0x243185be, 0x550c7dc3,
    0x72be5d74, 0x80deb1fe, 0x9bdc06a7, 0xc19bf174,
    0xe49b69c1, 0xefbe4786, 0x0fc19dc6, 0x240ca1cc,
    0x2de92c6f, 0x4a7484aa, 0x5cb0a9dc, 0x76f988da,
    0x983e5152, 0xa831c66d, 0xb00327c8, 0xbf597fc7,
    0xc6e00bf3, 0xd5a79147, 0x06ca6351, 0x14292967,
    0x27b70a85, 0x2e1b2138, 0x4d2c6dfc, 0x53380d13,
    0x650a7354, 0x766a0abb, 0x81c2c92e, 0x92722c85,
    0xa2bfe8a1, 0xa81a664b, 0xc24b8b70, 0xc76c51a3,
    0xd192e819, 0xd6990624, 0xf40e3585, 0x106aa070,
    0x19a4c116, 0x1e376c08, 0x2748774c, 0x34b0bcb5,
    0x391c0cb3, 0x4ed8aa4a, 0x5b9cca4f, 0x682e6ff3,
    0x748f82ee, 0x78a5636f, 0x84c87814, 0x8cc70208,
    0x90befffa, 0xa4506ceb, 0xbef9a3f7, 0xc67178f2,
]);
const H = Uint32Array.from([
    0x6a09e667, 0xbb67ae85, 0x3c6ef372, 0xa54ff53a,
    0x510e527f, 0x9b05688c, 0x1f83d9ab, 0x5be0cd19,
]);
```

It can also generate in the script as:

```js
function fractop32(v) {
    // split `double` data as little endian 8 bytes 
    const [b0, b1, b2, b3, b4, b5, b6, b7] =
          new Uint8Array(Float64Array.of(v).buffer);
    // exponential part of `double`
    const e = (((b7 & 0x7f) << 4) | (b6 >> 4)) - 1023;
    // upper 32bit of fraction part of `double`
    const upper = (((b6 & 0x0f) << 28) | (b5 << 20) | (b4 << 12) |
                   (b3 << 4) | (b2 >>> 4)) >>> 0;
    // lower 20bit of fraction part of `double`
    const lower = ((b2 & 0x0f) << 16) | (b1 << 8) | b0;
    // slide fraction part with e (as fractional part of an absolute number)
    if (e < 0) return (((1 << 31) >>> (-e - 1)) | (upper >>> -e)) >>> 0;
    return ((upper << e) | (lower >>> (20 - e))) >>> 0;
}
function primes(len) {
    // sieve of eratosthenes
    if (len < 1) return [];
    const ps = [2];
    for (let n = 3; ps.length < len; n += 2) {
        if (ps.every(p => n % p !== 0)) ps.push(n);
    }
    return ps;
}

const K = Uint32Array.from(primes(64), p => fractop32(Math.cbrt(p)));
const H = Uint32Array.from(primes(8), p => fractop32(Math.sqrt(p)));
```

### `sha256` class

For above `sha256` API, the class layout as:

```js
class sha256 {
    constructor() {
        //prepare initial states
    }
    update(buffer) {
        //process buffer as chunks, update states with chunks
        return this;
    }
    digest() {
        //process last chunks, then return the digest buffer
    }
}
```

### `sha256.constructor()`

```js
    constructor() {
        this.hv = H.slice();
        this.total = 0;
        this.chunk = new Uint8Array(64);
        this.filled = 0;
    }
```

Properties of the processing state are:

- `hv`: Initialization Vector (also result digest), copy from constants `H`
- `total`: total **byte length** of processing buffers
     - note: this implementation limited total data byte length < 2^53 (= 8-peta byte), not 2^64-1
- `chunk`: at most 512-bit(= 64-byte) message chunk remainder
- `filled`: remainder byte length (would be processed next)
     - note1: `filled` never reached to `64`
     - note2: negative `filled` means after processing finished

On `update(buffer)`, it may remain a last less than 64-byte chunk.
The remained `chunk` would be processed in `digest()`.

### `sha256.update(buffer)`

```js
    update(buffer) {
        let {filled} = this;
        if (filled < 0) return this;
        const {hv, chunk} = this;
        const buf = new Uint8Array(buffer);
        const bytes = buf.byteLength;
        let offs = 0;
        for (let next = 64 - filled; next <= bytes; next += 64) {
            chunk.set(buf.subarray(offs, next), filled);
            update(hv, chunk);
            filled = 0;
            offs = next;
        }
        chunk.set(buf.subarray(offs), filled);
        this.filled = filled + bytes - offs;
        this.total += bytes;
        return this;
    }
```

Concat `buffer` and `chunk` then split with each 64-byte to apply local function `update(hv, chunk)`:

```js
function update(hv, chunk) {
    const w = new Uint32Array(64);
    const cd = new DataView(chunk.buffer, chunk.byteOffset, chunk.byteLength);
    for (let i = 0; i < 16; i++) {
        w[i] = cd.getUint32(i * 4, false);
    }
    for (let i = 16; i < 64; i++) {
        const w16 = w[i - 16], w15 = w[i - 15], w7 = w[i - 7], w2 = w[i - 2];
        const s0 = (rotr(w15, 7) ^ rotr(w15, 18) ^ (w15 >>> 3)) >>> 0;
        const s1 = (rotr(w2, 17) ^ rotr(w2, 19) ^ (w2 >>> 10)) >>> 0;
        w[i] = (w16 + s0 + w7 + s1) >>> 0;
    }

    const data = step(0, w, ...hv);
    hv.set(hv.map((v, i) => (v + data[i]) >>> 0));
}
```

The `hv` is mixed with a `chunk` in 64 steps, then put back into the `hv`.

Before applying steps, `chunk` transition is prepared as `w[64]`.
The first 16 elements are just chunk data itself **as Big Endian**.
The latter is mixed with former 4 elements.

The `rotr(v, n)` is unsigned 32-bit right cyclic shift operation:

```js
function rotr(v, n) {
    return ((v >>> n) | (v << (32 - n))) >>> 0;
}
```

The `step(n, w, a, b, c, d, e, f, g, h)` is as a tailcall recursive function 
that represents a step of mixing and rotating vector elements:

```js
function step(i, w, a, b, c, d, e, f, g, h) {
    if (i === 64) return [a, b, c, d, e, f, g, h];
    // t1 from e, f, g, h (and K[i], w[i])
    const s1 = (rotr(e, 6) ^ rotr(e, 11) ^ rotr(e, 25)) >>> 0;
    const ch = ((e & f) ^ ((~e >>> 0) & g)) >>> 0;
    const t1 = (h + s1 + ch + K[i] + w[i]) >>> 0;
    // t2 from a, b, c
    const s0 = (rotr(a, 2) ^ rotr(a, 13) ^ rotr(a, 22)) >>> 0;
    const maj = ((a & b) ^ (a & c) ^ (b & c)) >>> 0;
    const t2 = (s0 + maj) >>> 0;
    // rotating vars a,b,c,d,e,f,g,h with mixed d and h
    return step(i + 1, w, (t1 + t2) >>> 0, a, b, c, (d + t1) >>> 0, e, f, g);
}
```

- The `t1` is non-linear **mixing of `e`, `f`, `g`, `h`** and chunk `w[i]` and key constant `K[i]`.
- The `t2` is non-linear **mixing of `a`, `b`, `c`**.

At the step,  `d` turns to `d + t1` and `h` turns to `t1 + t2`.
Then go next step with **rotating variables** 
from `[a, b, c, d, e, f, g, h]` to `[new_h, a, b, c, new_d, e, f, g]`.

For example, to implements with loop as:

```js
function update(hv, chunk) {
    //...
    let [a, b, c, d, e, f, g, h] = hv;
    for (let i = 0; i < 64; i++) {
        const s1 = (rotr(e, 6) ^ rotr(e, 11) ^ rotr(e, 25)) >>> 0;
        const ch = ((e & f) ^ ((~e >>> 0) & g)) >>> 0;
        const t1 = (h + s1 + ch + K[i] + w[i]) >>> 0;
        const s0 = (rotr(a, 2) ^ rotr(a, 13) ^ rotr(a, 22)) >>> 0;
        const maj = ((a & b) ^ (a & c) ^ (b & c)) >>> 0;
        const t2 = (s0 + maj) >>> 0;
        h = g; g = f; f = e; e = (d + t1) >>> 0;
        d = c; c = b; b = a; a = (t1 + t2) >>> 0;
    }
    hv[0] = (hv[0] + a) >>> 0; hv[1] = (hv[1] + b) >>> 0;
    hv[2] = (hv[2] + c) >>> 0; hv[3] = (hv[3] + d) >>> 0;
    hv[4] = (hv[4] + e) >>> 0; hv[5] = (hv[5] + f) >>> 0;
    hv[6] = (hv[6] + g) >>> 0; hv[7] = (hv[7] + h) >>> 0;
}
```

But the loop code is slower than the recursive function code in JavaScript engines.

### `sha256.digest()`

```js
    digest() {
        if (this.filled >= 0) {
            last(this);
            this.filled = -1;
        }
        const digest = new DataView(new ArrayBuffer(32));
        this.hv.forEach((v, i) => {digest.setUint32(i * 4, v, false);});
        return digest.buffer;
    }
```

The `last(this)` process the last remainder chunk and padding data.
For stop invalid processing called  `update()` or `digest()` after the `last()` process,
set `filled` value negative.
Then the result digest value in `hv` convert **Big Endian** byte stream,
and returns it.

The `last(ctx)` marks the bit 1, fills 0, and then embeds length value as a last chunk:

```js
function last(ctx) {
    const {hv, total, chunk, fulled} = ctx;
    chunk[filled] = 0x80;
    let start0 = filled + 1;
    if (start0 + 8 > 64) {
        chunk.fill(0, start0);
        update(hv, chunk);
        start0 = 0;
    }
    chunk.fill(0, start0, -8);
    
    const dv = new DataView(chunk.buffer, chunk.byteLength - 8);
    const b30 = 1 << 29; // note: bits is bytes * 8, lowwer use 29bit
    const low = total % b30;
    const high = (total - low) / b30;
    dv.setUint32(0, high, false);
    dv.setUint32(4, (low << 3) >>> 0, false);
    update(hv, chunk);
}
```

The last padding with reminder is 64-byte or 128-byte.
Inside the `if` block, it makes the first 64-bit as `[message..., 0x80, 0, ..., 0]`
then `update()` it. then the chunk as `[0, ...., 0]`.

latter part is embeding "bit length" as **64-bit big endian** value.
The "left shift 3" means "multiply 8" as byte length to bit length.

## The last part: for node.js entry points

The "sha256.js" has the entry point for "node.js".

```js
if (typeof require === "function" && require.main === module) {
    const fs = require("fs");
    const digestFile = name => new Promise((resolve, reject) => {
        //const hash = require("crypto").createHash("sha256");
        const hash = new sha256();
        const rs = fs.createReadStream(name);
        rs.on("readable", _ => {
            const buf = rs.read();
            if (buf) hash.update(buf);
        });
        rs.on("end", () => {resolve(hash.digest());});
        rs.on("error", error => {reject(error);});
    });
    
    const names = process.argv.slice(2);
    (async function () {
        for (const name of names) {
            const hex = Buffer.from(await digestFile(name)).toString("hex");
            console.log(`${hex}  ${name}`);
        }
    })().catch(console.log);

} else if (typeof module !== "undefined") {
    module.exports = sha256;
}
```

The main code generates hex digest of commandline parameter files.
The output format is compatible with ["sha256sum/gsha256sum"](https://linux.die.net/man/1/sha256sum) 
command of "coreutils".

```bash
$ touch empty.txt
$ node sha256.js empty.txt
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  empty.txt
```

You can compare results or performances with the existing implementations as execubales, e.g.:

```bash
$ dd if=/dev/zero bs=1m count=512 of=512M.data
512+0 records in
512+0 records out
536870912 bytes transferred in 0.458462 secs (1171025978 bytes/sec)
$ time gsha256sum 512M.data 
9acca8e8c22201155389f65abbf6bc9723edc7384ead80503839f49dcc56d767  512M.data

real	0m3.628s
user	0m3.237s
sys	0m0.297s

$ time node sha256.js 512M.data 
9acca8e8c22201155389f65abbf6bc9723edc7384ead80503839f49dcc56d767  512M.data

real	1m9.280s
user	1m8.905s
sys	0m0.776s

```

Note: 512M bytes is 4G bits as **over 32-bit range length**.
It is one of important inputs for testing hash functions.

---

## Appendix: Note on real world implementations of hash functions

### Not use loop for steps in update

When using loop, rotating variables requres n+1 assignments.
But in a step, only few values are updated (e.g. just 2 in 8 vars as SHA256).

So avoiding needless assignments,  manually extracted loops (called "unrolling") on the code
with steos as inline macro, e.g.

```c
#define STEP(a, b, c, d, e, f, g, w, k) {uint32_t t = T1(e,f,g,h,w,k); d += t; h = t + T2(a,b,c);}

STEP(a, b, c, d, e, f, g, h, w[0], K[0]);
STEP(h, a, b, c, d, e, f, g, w[1], K[1]);
STEP(g, h, a, b, c, d, e, f, w[2], K[2]);
STEP(f, g, h, a, b, c, d, e, w[3], K[3]);
//...
STEP(b, c, d, e, f, g, h, a, w[63], K[63]);
```

## Complex byte order handling

Usual hash function embeds byte-order maniputation code.
The code usually complex with many bit-mask, bit shift, bitwise-and/or, operations.
More worsely, these code also mix every host-emndien cases with `#ifdef` macros.

To read the code, it is is important to ignore the complex bit operations seems byte-order translations.

But these are old fasion codes.
To write the modern C99/C11 code, using `inline` functions with `union` variables 
instead of macros and bit shifts as:

```c
static inline uint32_t big32(uint32_t v) {
#ifdef WORDS_BIGENDIAN
  return v;
#else
  union {
    struct {uint8_t b0, b1, b2, b3;};
    uint32_t v;
  } le = {.v = v};
  union {
    struct {uint8_t b3, b2, b1, b0;};
    uint32_t v;
  } be = {.b0 = le.b0, .b1 = le.b1, .b2 = le.b2, .b3 = le.b3};
  return be.v;
#endif
}
```
