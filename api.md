# API reference of bbs-wasm

This document lists the public API of `bbs-wasm` (`src/bbs.ts`, `src/index.ts`). See [readme.md](readme.md) for an overview and examples.

- Node.js : `const bbs = require('bbs-wasm')`
- HTML : `<script src="https://herumi.github.io/bbs-wasm/browser/bbs.js"></script>` (the global object is `bbs`)

The library is experimental and the API may change without backward compatibility.

Functions throw an `Error` on invalid input or internal failure unless noted otherwise. Functions that return `boolean` (`verify`, `proofVerify`, `proofVerifyEx`, `isEqual`) return `false` for an invalid signature or proof instead of throwing.

## Initialization

### `init(maxMsgN = 128): Promise<void>`

Loads the WebAssembly module and initializes the library with the ciphersuite `BLS12381_SHA256`. It must be awaited before any other function is called.

- `maxMsgN` : the maximum number of messages in one signature. The generators for `maxMsgN` messages are precomputed.

### `term(): void`

Frees the memory allocated by `init`.

### `BLS12381_SHA256`

The ciphersuite identifier (`0`). It is the only ciphersuite supported now (BLS12-381-SHA-256 of draft-irtf-cfrg-bbs-signatures-12).

## Types

### `Msg = Uint8Array | bigint`

A message to be signed.

- `Uint8Array` : an octet string. It is hashed to a scalar as `messages_to_scalars` of the spec.
- `bigint` : an integer in `[0, 2^64)`. It is used as a scalar without hashing. This is an extension that is not defined in the spec. A predicate of `proofGenEx` requires an integer message.

### `PredTerm`

```ts
interface PredTerm {
  idx: number  // index of an undisclosed integer message
  coef: number // integer in [1, 2^32)
}
```

### `Predicate`

A range condition on the linear combination `x = sum of coef * msgs[idx]` over `terms`.

```ts
interface Predicate {
  terms: PredTerm[] // 1 to PRED_MAX_TERM terms. idx must be strictly increasing
  type: number      // PRED_GE or PRED_LE
  bound: bigint     // integer in [0, 2^64)
  bitN: number      // 1 to 64. bit length of the range
}
```

- `PRED_GE` (`0`) : `0 <= x - bound < 2^bitN`, i.e. `x >= bound`
- `PRED_LE` (`1`) : `0 <= bound - x < 2^bitN`, i.e. `x <= bound`
- `PRED_MAX_TERM` (`8`) : the maximum number of terms

`x` may exceed `2^64`, but `bound` is a 64-bit integer, so a predicate can only state that `x` is in `[bound, bound + 2^bitN)` or in `(bound - 2^bitN, bound]`. The proof size and the proof generation time grow linearly in `bitN`.

## Classes

`SecretKey`, `PublicKey` and `Signature` share the following methods.

| method | description |
|---|---|
| `serialize(): Uint8Array` | serializes to octets (the format of the spec) |
| `deserialize(buf: Uint8Array): void` | deserializes from octets. Throws if `buf` is invalid |
| `serializeToHexStr(): string` | serializes to a hex string |
| `deserializeHexStr(s: string): void` | deserializes from a hex string |
| `isEqual(rhs): boolean` | returns `true` if the two values are equal |
| `clone()` | returns a copy |
| `clear(): void` | fills the internal buffer with zero |
| `dump(msg = ''): void` | prints `msg` followed by the hex string to the console |

Serialized sizes are 32 bytes for `SecretKey`, 96 bytes for `PublicKey` and 80 bytes for `Signature`.

### `class SecretKey`

| method | description |
|---|---|
| `init(): void` | sets a random secret key |
| `keyGen(keyMaterial: Uint8Array, keyInfo?: Uint8Array, keyDst?: Uint8Array): void` | KeyGen of the spec. `keyMaterial` is a secret octet string of at least 32 bytes. `keyInfo` and `keyDst` are optional. The default dst is used if `keyDst` is not specified |
| `getPublicKey(): PublicKey` | returns the public key corresponding to the secret key |

### `class PublicKey`

Only the common methods.

### `class Signature`

Only the common methods.

### Helper constructors from a hex string

- `deserializeHexStrToSecretKey(s: string): SecretKey`
- `deserializeHexStrToPublicKey(s: string): PublicKey`
- `deserializeHexStrToSignature(s: string): Signature`

Each creates a new object and calls `deserializeHexStr(s)` on it.

## Sign and verify

### `sign(sec: SecretKey, pub: PublicKey, msgs: Msg[], header?: Uint8Array): Signature`

Signs `msgs` (Sign of the spec). `msgs.length` must be at most the `maxMsgN` given to `init`.

- `header` : optional context octets. It must be given to `verify`, `proofGen` and `proofVerify` as well.

### `verify(sig: Signature, pub: PublicKey, msgs: Msg[], header?: Uint8Array): boolean`

Verifies `sig` for `msgs` (Verify of the spec). Returns `true` if the signature is valid.

## Selective disclosure proof

### `getProofSize(undiscN: number): number`

Returns the size in bytes of a proof with `undiscN` undisclosed messages, which is `3 * 48 + (4 + undiscN) * 32`.

### `proofGen(pub: PublicKey, sig: Signature, msgs: Msg[], discIdxs: Uint32Array, header?: Uint8Array, ph?: Uint8Array): Uint8Array`

Creates a proof that discloses only `msgs[discIdxs[i]]` (ProofGen of the spec). Returns the proof.

- `discIdxs` : indices of the messages to be disclosed in strictly ascending order. An empty array discloses nothing
- `header` : the header used in `sign`
- `ph` : optional presentation header. It is bound to the proof and must be given to `proofVerify`

`sig` is not verified here. Call `verify` first if the signature is untrusted.

### `proofVerify(pub: PublicKey, proof: Uint8Array, discMsgs: Msg[], discIdxs: Uint32Array, header?: Uint8Array, ph?: Uint8Array): boolean`

Verifies `proof` (ProofVerify of the spec). Returns `true` if the proof is valid.

- `discMsgs` : the disclosed messages, `discMsgs[i] = msgs[discIdxs[i]]`
- `discIdxs` : the same indices as those of `proofGen`
- `header`, `ph` : the same values as those of `proofGen`

## Range predicates (extension, not defined in the spec)

A proof created by `proofGenEx` is not a proof of the spec and is verified only by `proofVerifyEx`.

### `getProofExSize(undiscN: number, preds: Predicate[]): number`

Returns the size in bytes of a proof of `proofGenEx`, which is `getProofSize(undiscN) + 80 * (number of distinct terms) + sum of (144 * bitN - 48)`. Returns `0` if `preds` is invalid (e.g. not sorted). Throws if `terms.length`, `coef` or `bound` is out of range.

### `proofGenEx(pub: PublicKey, sig: Signature, msgs: Msg[], discIdxs: Uint32Array, preds: Predicate[], header?: Uint8Array, ph?: Uint8Array): Uint8Array`

The same as `proofGen` with the predicates `preds`. Returns the proof.

- `preds` : predicates sorted by `(terms.length, idx, coef, ...)` in lexicographic order. Predicates with the same `terms` share a commitment. Every `idx` must be an index of an undisclosed `bigint` message

Throws if `preds` is invalid or if a predicate does not hold for `msgs`.

### `proofVerifyEx(pub: PublicKey, proof: Uint8Array, discMsgs: Msg[], discIdxs: Uint32Array, preds: Predicate[], header?: Uint8Array, ph?: Uint8Array): boolean`

The same as `proofVerify` with the predicates `preds`, which must be the same as those of `proofGenEx`. Returns `true` if the proof is valid and all the predicates hold.

## Utilities

| function | description |
|---|---|
| `toHexStr(a: Uint8Array): string` | converts octets to a hex string |
| `fromHexStr(s: string): Uint8Array` | converts a hex string to octets. Throws if the length is odd |
| `toHex(a: Uint8Array, start: number, n: number): string` | converts `a[start, start + n)` to a hex string |
| `getRandomValues(buf: Uint8Array): Uint8Array` | fills `buf` with cryptographically random values (`crypto.getRandomValues`) |
