[![Build Status](https://github.com/herumi/bbs-wasm/actions/workflows/main.yml/badge.svg)](https://github.com/herumi/bbs-wasm/actions/workflows/main.yml)

# BBS signature for Node.js and browsers by WebAssembly

# Abstract

This library implements the BBS signature scheme of [draft-irtf-cfrg-bbs-signatures](https://datatracker.ietf.org/doc/draft-irtf-cfrg-bbs-signatures/) (draft 12, ciphersuite BLS12-381-SHA-256).

- A signer signs a list of messages with one signature.
- A holder of the signature creates a proof that discloses only a subset of the messages.
- A verifier checks the proof with the public key without learning the undisclosed messages.
- As an extension that is not defined in the spec, a proof can also show that an undisclosed integer message satisfies range predicates such as `18 <= age <= 65`.

The C++ implementation is in [mcl](https://github.com/herumi/mcl) (`include/mcl/bbs.h`, `include/mcl/bbs.hpp`, `src/bbs.cpp`).
This repository builds it to WebAssembly and provides a TypeScript API.

This library is experimental. The API is not fixed yet and may change without backward compatibility.

# News
- 2026/Oct/07 initial version

# Demo

* [selective disclosure demo](https://herumi.github.io/bbs-wasm/browser/index.html) (switch en/ja by the top-right button)

# for Node.js

```
pnpm test
```

# How to use

- Node.js : `const bbs = require('bbs-wasm')`
- HTML : `<script src="https://herumi.github.io/bbs-wasm/browser/bbs.js"></script>` (the global object is `bbs`)

```js
const bbs = require('bbs-wasm')

async function main () {
  await bbs.init()

  // key generation
  const sec = new bbs.SecretKey()
  sec.init() // random secret key
  // or sec.keyGen(keyMaterial, keyInfo, keyDst) as the KeyGen of the spec
  const pub = sec.getPublicKey()

  // sign messages (Uint8Array or bigint)
  const msgs = [new Uint8Array([1, 2, 3]), 19960320n, new Uint8Array([10, 11, 12, 13])]
  const header = new Uint8Array([0x11, 0x22]) // optional
  const sig = bbs.sign(sec, pub, msgs, header)
  console.log(bbs.verify(sig, pub, msgs, header)) // true

  // disclose msgs[0] and msgs[2] only
  const discIdxs = new Uint32Array([0, 2])
  const discMsgs = [msgs[0], msgs[2]]
  const ph = new Uint8Array([1, 2, 3]) // presentation header (optional)
  const proof = bbs.proofGen(pub, sig, msgs, discIdxs, header, ph)
  console.log(bbs.proofVerify(pub, proof, discMsgs, discIdxs, header, ph)) // true

  bbs.term()
}

main()
```

The serialized forms are `Uint8Array` (`serialize()` / `deserialize()`) or hex strings (`serializeToHexStr()` / `deserializeHexStr()`, `bbs.deserializeHexStrToSecretKey()` etc.).

## Range predicates (extension)

`proofGenEx` / `proofVerifyEx` take an array of predicates for undisclosed integer messages (`bigint` messages in `[0, 2^64)`).

```js
// msgs = [name, birthday (YYYYMMDD), age, address]
const msgs = [name, 19960320n, 30n, address]
const sig = bbs.sign(sec, pub, msgs, header)

// disclose the name and show that birthday <= 20081001 and 18 <= age <= 65
const discIdxs = new Uint32Array([0])
const preds = [
  { idx: 1, type: bbs.PRED_LE, bound: 20081001n, bitN: 25 },
  { idx: 2, type: bbs.PRED_GE, bound: 18n, bitN: 8 },
  { idx: 2, type: bbs.PRED_LE, bound: 65n, bitN: 8 }
]
const proof = bbs.proofGenEx(pub, sig, msgs, discIdxs, preds, header, ph)
console.log(bbs.proofVerifyEx(pub, proof, [msgs[0]], discIdxs, preds, header, ph)) // true
```

- `idx` : index of the (undisclosed) message
- `type` : `bbs.PRED_GE` (message >= bound) or `bbs.PRED_LE` (message <= bound)
- `bound` : `bigint` in `[0, 2^64)`
- `bitN` : bit length of the range (`|message - bound| < 2^bitN`); the proof size grows linearly in the total of `bitN`

A proof created by `proofGenEx` is not a proof of the spec and is verified only by `proofVerifyEx`.

# How to build
Install clang, lld and binaryen (`wasm-opt`). The Makefile uses `clang-21` by default; override it with `make wasm LLVM_VER=-20`.

```
sudo apt install clang lld binaryen
git submodule update --init --recursive
pnpm install
pnpm run build       # make wasm (src/bbs_c.js) and tsc (dist/)
pnpm run build:browser # browser/bbs.js
pnpm run build:demo    # browser/demo.js
```

`src/bbs_c.js` is generated from `src/mcl` and is not committed.

# License

modified new BSD License
http://opensource.org/licenses/BSD-3-Clause

# Author

MITSUNARI Shigeo(herumi@nifty.com)

# Sponsors welcome
[GitHub Sponsor](https://github.com/sponsors/herumi)
