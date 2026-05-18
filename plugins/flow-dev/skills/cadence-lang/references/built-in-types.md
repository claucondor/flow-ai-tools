# Built-in Types — No Import Required

Several cryptographic and account types are built directly into the Cadence language runtime.
They are always in scope without any import statement, the same way `Int` or `String` are.
Attempting to import them as if they were contracts causes a compile error with a confusing
message that mentions an invalid location, leading developers to hunt for a non-existent
package. The fix is always to delete the import line.

---

## Types That Are Built-In (No Import)

The following are native Cadence types. They require no `import` statement in any context —
contracts, transactions, scripts, or test files.

### Cryptographic types

| Type | Kind | Notes |
|---|---|---|
| `HashAlgorithm` | Built-in enum | Cases: `SHA2_256`, `SHA2_384`, `SHA3_256`, `SHA3_384`, `KECCAK_256`, `KMAC128_BLS_BLS12_381` |
| `SignatureAlgorithm` | Built-in enum | Cases: `ECDSA_P256`, `ECDSA_secp256k1`, `BLS_BLS12_381` |
| `PublicKey` | Built-in struct | Constructed with `publicKey: [UInt8]` and `signatureAlgorithm:` |
| `AccountKey` | Built-in struct | Fields: `keyIndex`, `publicKey`, `hashAlgorithm`, `weight`, `isRevoked` |

### Language types

| Type | Kind | Notes |
|---|---|---|
| `Address` | Built-in type | 8-byte account address |
| `Block` | Built-in struct | Fields: `id`, `height`, `view`, `timestamp`; returned by `getCurrentBlock()` |
| `Type` | Built-in metatype | Represents a Cadence type at runtime; constructed with `Type<T>()` |
| `Capability` | Built-in generic struct | `Capability<&T>` — no import required |
| `Path` (and `StoragePath`, `PublicPath`, `PrivatePath`) | Built-in types | Path literals like `/storage/foo` are `StoragePath` |
| `Account` | Built-in struct | The full account type returned by `getAccount()` and available in `prepare` |

`BLS` is a **built-in contract** — it does not need to be imported. Call its functions
directly as `BLS.aggregateSignatures(...)` and `BLS.aggregatePublicKeys(...)`.

---

## The Mistake: Importing a Built-In Type

```cadence
// ❌ FAILS — HashAlgorithm is not a deployable contract
import "HashAlgorithm"

access(all) fun main(): [UInt8] {
    return HashAlgorithm.SHA3_256.hash([1, 2, 3])
}
```

The exact error produced on v2.17.1 is a configuration-check error fired before any Cadence
compilation:

```
error: cannot find contract with location 'HashAlgorithm' in configuration
--> HashAlgorithm
```

The same form applies to `SignatureAlgorithm` and `PublicKey`:

```
error: cannot find contract with location 'SignatureAlgorithm' in configuration
--> SignatureAlgorithm
```

```
error: cannot find contract with location 'PublicKey' in configuration
--> PublicKey
```

This is a pre-compile Flow CLI error, not a Cadence type or location error. The CLI fails
at configuration checking — it looks for a contract named `HashAlgorithm` in `flow.json`
or the dependency graph, finds none, and stops. The root cause is that `"HashAlgorithm"` is
an identifier import — the CLI tries to resolve it as a deployable contract and cannot find
any entry at that name.

```cadence
// ✅ CORRECT — no import, use directly
access(all) fun main(): [UInt8] {
    return HashAlgorithm.SHA3_256.hash([1, 2, 3])
}
```

The same mistake applies to `SignatureAlgorithm`, `PublicKey`, `AccountKey`, and `BLS`:

```cadence
// ❌ All of these are wrong
import "SignatureAlgorithm"
import "PublicKey"
import "AccountKey"
import "BLS"

// ✅ None of these need an import
let algo = SignatureAlgorithm.ECDSA_P256
let key  = PublicKey(publicKey: bytes, signatureAlgorithm: algo)
let hash = HashAlgorithm.SHA2_256.hash(data)
let agg  = BLS.aggregateSignatures([sig1, sig2])
```

---

## What DOES Require an Import

`Crypto` is the one cryptography-related import that IS required. It provides `Crypto.KeyList`,
`Crypto.KeyListSignature`, `Crypto.KeyListEntry`, and the `Crypto.hash()` / `Crypto.hashWithTag()`
convenience wrappers.

```cadence
// ✅ Crypto must be imported explicitly
import Crypto

access(all) fun verifyMultiSig(
    signatureSet: [Crypto.KeyListSignature],
    signedData: [UInt8]
): Bool {
    let keyList = Crypto.KeyList()
    // ... add keys ...
    return keyList.verify(signatureSet: signatureSet, signedData: signedData, domainSeparationTag: "")
}
```

The distinction: `HashAlgorithm` and `SignatureAlgorithm` are enum types that live in the
language runtime. `Crypto` is a contract deployed to the service account that provides
higher-level abstractions (`KeyList`, multi-sig verification) built on top of those enums.

---

## Using Built-In Types in Tests

The same rules apply inside test files and in transactions executed via `Test.executeTransaction`.
No import is needed:

```cadence
// In a _test.cdc file — no import for cryptographic types
import Test

access(all) fun testHashOutput() {
    let data: [UInt8] = [0x01, 0x02, 0x03]
    let digest = HashAlgorithm.SHA3_256.hash(data)
    Test.assertEqual(32, digest.length)
}

access(all) fun testPublicKeyConstruction() {
    // Bytes below are illustrative; real keys must be valid curve points.
    let algo = SignatureAlgorithm.ECDSA_P256
    // PublicKey construction panics on invalid bytes — use valid keys in tests.
    // let key = PublicKey(publicKey: validBytes, signatureAlgorithm: algo)
}
```

---

## Diagnosing "Cannot Find Declaration" on a Built-In Type

If you see `cannot find declaration HashAlgorithm` (or any of the other built-in types) in a
script or contract, the most common causes are:

1. **Import statement present** — Delete the import line. The type is always in scope.
2. **Typo in the type name** — `HashAlgorythm`, `SignatureAlgo`, etc. Cadence is case-sensitive.
3. **Cadence 0.x → 1.0 migration** — In Cadence 0.x some of these types had different names
   or required different syntax. If you are migrating older code, check the migration guide
   at `cadence-lang.org`.

---

## See Also

- [`crypto.md`](crypto.md) — Full Cadence Crypto API reference: the `Crypto` contract
  (`KeyList`, multi-sig, `KeyListSignature`), worked examples for secp256k1 / P-256 / BLS
  aggregation, and common pitfalls
- [`accounts.md`](accounts.md) — `Account` type, storage, capabilities, and key management
- [`randomness.md`](randomness.md) — `HashAlgorithm` is used internally by `RandomConsumer`
  and `Xorshift128plus`; see this reference for the randomness APIs
