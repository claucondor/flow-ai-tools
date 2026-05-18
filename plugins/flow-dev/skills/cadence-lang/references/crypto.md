# Cadence Crypto API

The `Crypto` contract and the built-in `BLS` module are the security boundary for any
non-trivial Cadence contract. They expose cryptographic primitives that the Flow runtime
executes natively: signature verification, multi-signature key lists, hash computation, and
BLS aggregation. Crypto is concerned with *verification* — proving that a message was signed
by a known key. This is distinct from randomness, which is concerned with *generation*; see
[randomness.md](randomness.md) for `RandomConsumer`, commit-reveal, and beacon history.
Get these primitives wrong and your authorization model breaks silently — `verify()` returns
`false` instead of panicking, so a domain-tag mismatch looks exactly like a forged signature.

---

## Importing

```cadence
import Crypto   // exposes Crypto.KeyList, Crypto.KeyListSignature, Crypto.KeyListEntry
                // also exposes Crypto.hash() and Crypto.hashWithTag() as convenience wrappers
```

`PublicKey`, `SignatureAlgorithm`, and `HashAlgorithm` are built into the Cadence language
— they do not require an import.

```cadence
// No import needed for these:
let pk = PublicKey(publicKey: bytes, signatureAlgorithm: SignatureAlgorithm.ECDSA_P256)
let digest = HashAlgorithm.SHA3_256.hash(data)
```

`BLS` is a built-in contract that does **not** need to be imported. Call its functions
directly as `BLS.aggregateSignatures(...)` and `BLS.aggregatePublicKeys(...)`.

---

## SignatureAlgorithm

```cadence
access(all)
enum SignatureAlgorithm: UInt8 {
    case ECDSA_P256        = 1
    case ECDSA_secp256k1   = 2
    case BLS_BLS12_381     = 3
}
```

| Algorithm | Curve | Raw public key | Signature | Typical use case | Choose when |
|---|---|---|---|---|---|
| `ECDSA_P256` | NIST P-256 | 64 bytes | 64 bytes | Passkeys, WebAuthn, Apple Secure Enclave, on-chain account keys | You integrate with hardware security keys or browser WebAuthn |
| `ECDSA_secp256k1` | secp256k1 | 64 bytes | 64 bytes | Ethereum wallets, MetaMask, WalletConnect, Bitcoin | You verify signatures produced off-chain by an Ethereum wallet |
| `BLS_BLS12_381` | BLS12-381 | 96 bytes | 48 bytes | Aggregated multi-sig, threshold signatures, consensus beacon | You need to aggregate N signatures into one, or integrate with Flow's internal randomness beacon |

Key sizes above are for the **uncompressed raw** encoding passed to `PublicKey(publicKey:...)`.

---

## HashAlgorithm

```cadence
access(all)
enum HashAlgorithm: UInt8 {
    case SHA2_256               = 1
    case SHA2_384               = 2
    case SHA3_256               = 3
    case SHA3_384               = 4
    case KMAC128_BLS_BLS12_381  = 5
    case KECCAK_256             = 6

    access(all) view fun hash(_ data: [UInt8]): [UInt8]
    access(all) view fun hashWithTag(_ data: [UInt8], tag: String): [UInt8]
}
```

| Algorithm | Output | Domain separation | Use case |
|---|---|---|---|
| `SHA2_256` | 32 bytes | Optional tag | General purpose; the most common choice for ECDSA on P-256 |
| `SHA2_384` | 48 bytes | Optional tag | When wider output is required (rare) |
| `SHA3_256` | 32 bytes | Optional tag | Standard SHA-3; NOT Ethereum-compatible (Ethereum uses legacy Keccak padding, not the NIST SHA-3 padding) |
| `SHA3_384` | 48 bytes | Optional tag | Wide-output SHA-3 (rare) |
| `KECCAK_256` | 32 bytes | Optional tag | Ethereum-compatible; used by `eth_sign`, EIP-191, EIP-712. Differs from `SHA3_256` in the padding rule |
| `KMAC128_BLS_BLS12_381` | **128 bytes** (Empirically confirmed on Flow CLI v2.17.1 emulator across multiple input sizes and tag variants — output is exactly 128 bytes.) | Required non-empty tag for BLS-specific uses | BLS hash-to-curve; the same hasher used by Flow's internal consensus protocol. Use only with `BLS_BLS12_381` keys |

### Direct hashing

```cadence
let data: [UInt8] = [1, 2, 3]

// Hash without a tag
let digest = HashAlgorithm.SHA3_256.hash(data)    // 32 bytes

// Hash with a domain separation tag
let tagged = HashAlgorithm.SHA3_256.hashWithTag(data, tag: "my-app-v1")
```

`Crypto.hash(_:algorithm:)` and `Crypto.hashWithTag(_:tag:algorithm:)` are thin wrappers
that delegate to the same enum methods. Either form is acceptable.

---

## PublicKey

```cadence
access(all)
struct PublicKey {
    access(all) let publicKey: [UInt8]
    access(all) let signatureAlgorithm: SignatureAlgorithm

    access(all) view init()

    access(all) view fun verify(
        signature: [UInt8],
        signedData: [UInt8],
        domainSeparationTag: String,
        hashAlgorithm: HashAlgorithm
    ): Bool

    access(all) view fun verifyPoP(_ proof: [UInt8]): Bool
}
```

### Construction

```cadence
let key = PublicKey(
    publicKey: "010203...".decodeHex(),
    signatureAlgorithm: SignatureAlgorithm.ECDSA_P256
)
```

**Validation at construction time.** The runtime validates the key bytes immediately. If the
bytes are malformed (wrong length, point not on curve), the program **aborts** — there is no
`isValid` field and no way to construct an invalid `PublicKey`. All existing `PublicKey`
values are guaranteed valid.

**Immutability.** Fields are `let` and mutations have no effect:

```cadence
key.signatureAlgorithm = SignatureAlgorithm.ECDSA_secp256k1  // ❌ compile error
key.publicKey = []                                            // ❌ compile error
key.publicKey[0] = 99                                         // ❌ no effect (copies a value)
```

### `verify()`

```cadence
let isValid: Bool = key.verify(
    signature: sigBytes,
    signedData: messageBytes,
    domainSeparationTag: "my-app-v1",
    hashAlgorithm: HashAlgorithm.SHA2_256
)
```

- Returns `Bool`. Never panics on bad data — invalid signatures yield `false`.
- The `domainSeparationTag` is prepended to the data before hashing. Pass `""` only when
  the signing side also used an empty tag; otherwise signatures will not match.
- For ECDSA, the tag must not exceed 32 bytes (UTF-8 encoded).
- For BLS, any string length is valid.

### `verifyPoP()`

```cadence
let valid: Bool = blsKey.verifyPoP(popBytes)
```

Verifies a BLS *proof of possession* — a signature by the key over the key's own public key
bytes. Only available for `BLS_BLS12_381` keys. Calling on any other algorithm **aborts**.
Always call `verifyPoP` before including a key in BLS aggregation; see the aggregation
section below.

---

## BLS Aggregation

BLS is a built-in contract — **no import required**.

```cadence
// Aggregate N BLS signatures into one combined signature
view fun BLS.aggregateSignatures(_ signatures: [[UInt8]]): [UInt8]?

// Aggregate N BLS public keys into one combined public key
view fun BLS.aggregatePublicKeys(_ publicKeys: [PublicKey]): PublicKey?
```

Both functions are `view` — safe to call from scripts and view contexts.

### When aggregation returns nil

| Condition | `aggregateSignatures` | `aggregatePublicKeys` |
|---|---|---|
| Empty array | returns `nil` | returns `nil` |
| Decoding failure (malformed bytes) | returns `nil` | — |
| Any key is not `BLS_BLS12_381` | — | **aborts** |

All documented error cases return `nil`. Always nil-check the result with `?? panic(...)` or
a guard rather than force-unwrapping with `!`. Verified empirically on Flow CLI v2.17.1 —
`BLS.aggregate*` is total: every error case returns `nil`, never aborts.

### BLS key generation requires the flow-go SDK

**BLS key generation requires the flow-go SDK, not the Flow CLI.** As of CLI v2.17.1,
`flow keys generate --sig-algo BLS_BLS12_381` fails with "invalid signature algorithm".
JavaScript libraries like Noble/bls12-381 may generate G2 points that are incompatible with
the emulator's gnark-crypto encoding — verify round-trip before relying on a non-flow-go SDK
for production BLS keys.

### Why proof of possession matters

BLS aggregation is vulnerable to the *rogue key attack*: an adversary registers a public key
`pk_attacker = pk_victim * (-1)` (negation of a victim's key). When aggregated with the
victim's key, the attacker's key cancels the victim's, making the aggregated key correspond
to only the attacker's private key. The attacker can then forge signatures that appear to
come from a quorum they do not control.

**Defense**: each participant submits a proof of possession — a BLS signature over their own
public key. `verifyPoP()` confirms the submitter knows the matching private key. Verify PoP
for every key before including it in aggregation.

---

## Worked Example 1 — Verify a secp256k1 Signature (Ethereum Wallet)

This pattern verifies an Ethereum-style signature produced by MetaMask or any secp256k1
signer. Note that Ethereum's `eth_sign` prepends `"\x19Ethereum Signed Message:\n<len>"` to
the data before signing; you must include that prefix in `signedData` if matching an
Ethereum-prefixed signature.

```cadence
// Script: verify_secp256k1.cdc
access(all)
fun main(
    rawPublicKey: String,   // 64-byte hex, no 04 prefix
    signatureHex: String,   // 64-byte hex (r||s)
    message: [UInt8]
): Bool {
    let key = PublicKey(
        publicKey: rawPublicKey.decodeHex(),
        signatureAlgorithm: SignatureAlgorithm.ECDSA_secp256k1
    )

    // Use KECCAK_256 for Ethereum-compatible hashing.
    // Use an empty domainSeparationTag when the signer did not apply one.
    return key.verify(
        signature: signatureHex.decodeHex(),
        signedData: message,
        domainSeparationTag: "",
        hashAlgorithm: HashAlgorithm.KECCAK_256
    )
}
```

**Important**: for `eth_sign`-prefixed messages, pass the already-prefixed bytes:

```cadence
// Build the Ethereum signed-message prefix before calling verify
let prefix = "\u{0019}Ethereum Signed Message:\n".utf8
let lenBytes = message.length.toString().utf8
let prefixed: [UInt8] = prefix.concat(lenBytes).concat(message)

let isValid = key.verify(
    signature: sig,
    signedData: prefixed,
    domainSeparationTag: "",
    hashAlgorithm: HashAlgorithm.KECCAK_256
)
```

---

## Worked Example 2 — Verify a P-256 Signature (WebAuthn / Passkey)

WebAuthn authenticators (Face ID, Touch ID, YubiKey) sign with ECDSA P-256 and SHA-256. The
authenticator returns `clientDataJSON` and `authenticatorData`; the signed bytes are the
SHA-256 hash of `authenticatorData || SHA-256(clientDataJSON)`. Perform that assembly
off-chain and pass the pre-assembled signed bytes to Cadence.

```cadence
// Script: verify_webauthn.cdc
access(all)
fun main(
    rawPublicKey: String,   // 64-byte uncompressed P-256 key (x||y, no 04 prefix)
    signatureHex: String,   // 64-byte DER-decoded r||s
    signedBytes: [UInt8]    // authenticatorData || SHA256(clientDataJSON) — pre-assembled
): Bool {
    let key = PublicKey(
        publicKey: rawPublicKey.decodeHex(),
        signatureAlgorithm: SignatureAlgorithm.ECDSA_P256
    )

    // WebAuthn uses SHA-2 256 and no domain separation tag at the Cadence layer.
    // The domain separation is embedded in the WebAuthn signed structure itself.
    return key.verify(
        signature: signatureHex.decodeHex(),
        signedData: signedBytes,
        domainSeparationTag: "",
        hashAlgorithm: HashAlgorithm.SHA2_256
    )
}
```

---

## Worked Example 3 — Aggregate 3 BLS Signatures and Verify

Use case: threshold oracle — three nodes sign the same message off-chain; on-chain you
verify the single aggregated signature against the aggregated public key, saving computation
versus verifying each signature individually.

```cadence
// Script: verify_bls_aggregate.cdc
access(all)
fun main(
    rawPubKey1: String,   // 96-byte BLS G2 public key, hex-encoded
    rawPubKey2: String,
    rawPubKey3: String,
    popProof1: String,    // proof of possession for key 1
    popProof2: String,
    popProof3: String,
    sig1: String,         // 48-byte BLS G1 signature from signer 1
    sig2: String,
    sig3: String,
    message: [UInt8]
): Bool {
    // Step 1: construct individual public keys
    let pk1 = PublicKey(
        publicKey: rawPubKey1.decodeHex(),
        signatureAlgorithm: SignatureAlgorithm.BLS_BLS12_381
    )
    let pk2 = PublicKey(
        publicKey: rawPubKey2.decodeHex(),
        signatureAlgorithm: SignatureAlgorithm.BLS_BLS12_381
    )
    let pk3 = PublicKey(
        publicKey: rawPubKey3.decodeHex(),
        signatureAlgorithm: SignatureAlgorithm.BLS_BLS12_381
    )

    // Step 2: verify proof of possession before aggregating.
    // Skipping this step opens the rogue key attack.
    assert(pk1.verifyPoP(popProof1.decodeHex()), message: "PoP failed for key 1")
    assert(pk2.verifyPoP(popProof2.decodeHex()), message: "PoP failed for key 2")
    assert(pk3.verifyPoP(popProof3.decodeHex()), message: "PoP failed for key 3")

    // Step 3: aggregate public keys
    let aggregatedKey = BLS.aggregatePublicKeys([pk1, pk2, pk3])
        ?? panic("BLS.aggregatePublicKeys returned nil")

    // Step 4: aggregate signatures
    let aggregatedSig = BLS.aggregateSignatures([
        sig1.decodeHex(),
        sig2.decodeHex(),
        sig3.decodeHex()
    ]) ?? panic("BLS.aggregateSignatures returned nil")

    // Step 5: verify the aggregated signature against the aggregated key.
    // All signers must have signed the same message and used the same domain tag and hash.
    return aggregatedKey.verify(
        signature: aggregatedSig,
        signedData: message,
        domainSeparationTag: "my-oracle-v1",
        hashAlgorithm: HashAlgorithm.KMAC128_BLS_BLS12_381
    )
}
```

Aggregation is order-independent: the order of keys and signatures does not need to match
each other, but signatures and keys must correspond to the same set of signers.

---

## Worked Example 4 — Crypto.KeyList with 2-of-3 Threshold

`Crypto.KeyList` models weighted multi-signature verification. Verification succeeds when the
cumulative weight of valid signatures reaches 1.0. For a 2-of-3 scheme, assign each key a
weight of `0.5` — any two valid signatures sum to 1.0.

```cadence
import Crypto

// Transaction: submit_multisig.cdc
// signerA and signerB each sign `signedData` off-chain.
transaction(
    signedData: [UInt8],
    sigA: [UInt8],
    sigB: [UInt8]
) {
    prepare(admin: auth(BorrowValue) &Account) {
        // Build the key list from stored public keys.
        // In production, load these from contract storage rather than hard-coding.
        let keyList = Crypto.KeyList()

        let pkA = PublicKey(
            publicKey: "db04940e18ec414664ccfd31d5d2d4ece3985acb8cb17a2025b2f1673427267968e52e2bbf3599059649d4b2cce98fdb8a3048e68abf5abe3e710129e90696ca".decodeHex(),
            signatureAlgorithm: SignatureAlgorithm.ECDSA_P256
        )
        let pkB = PublicKey(
            publicKey: "df9609ee588dd4a6f7789df8d56f03f545d4516f0c99b200d73b9a3afafc14de5d21a4fc7a2a2015719dc95c9e756cfa44f2a445151aaf42479e7120d83df956".decodeHex(),
            signatureAlgorithm: SignatureAlgorithm.ECDSA_P256
        )
        let pkC = PublicKey(    // third key — not signing this time
            publicKey: "a3bc940e18ec414664ccfd31d5d2d4ece3985acb8cb17a2025b2f1673427267968e52e2bbf3599059649d4b2cce98fdb8a3048e68abf5abe3e710129e90696ca".decodeHex(),
            signatureAlgorithm: SignatureAlgorithm.ECDSA_P256
        )

        // Each key gets weight 0.5. Any two are sufficient (0.5 + 0.5 = 1.0).
        keyList.add(pkA, hashAlgorithm: HashAlgorithm.SHA3_256, weight: 0.5)
        keyList.add(pkB, hashAlgorithm: HashAlgorithm.SHA3_256, weight: 0.5)
        keyList.add(pkC, hashAlgorithm: HashAlgorithm.SHA3_256, weight: 0.5)

        // Provide signatures for keys A (index 0) and B (index 1).
        // Key C (index 2) does not sign.
        let signatureSet = [
            Crypto.KeyListSignature(keyIndex: 0, signature: sigA),
            Crypto.KeyListSignature(keyIndex: 1, signature: sigB)
        ]

        let isValid = keyList.verify(
            signatureSet: signatureSet,
            signedData: signedData,
            domainSeparationTag: "FLOW-V0.0-user"
        )

        assert(isValid, message: "Multi-sig verification failed")
        // Proceed with authorized action...
    }
}
```

### KeyList verification rules (internal logic)

From the canonical `Crypto.cdc` source, `KeyList.verify()` enforces:

1. Every `keyIndex` in the signature set must be within bounds — otherwise returns `false`.
2. No key index may appear more than once (duplicate key replay) — returns `false` on duplicate.
3. Revoked keys (`isRevoked == true`) invalidate their signature — returns `false`.
4. Each signature is verified against its key's `hashAlgorithm` and the shared `domainSeparationTag`.
5. The function accumulates `validWeights` and returns `true` only when `validWeights >= 1.0`.

### Revoking a key

```cadence
keyList.revoke(keyIndex: 1)
// get(keyIndex: 1) still returns the entry, but entry.isRevoked == true
// Subsequent verify() calls that include keyIndex 1 will return false
```

---

## KMAC128 Deep Dive

`KMAC128_BLS_BLS12_381` is an instance of KMAC128 (KECCAK Message Authentication Code)
customized for BLS signature operations in Flow. Although KMAC is technically a MAC
algorithm, it is used here as a hash function by treating the KMAC key as a non-public
customizer rather than a secret key.

### Internal parameters

| Parameter | `hash()` value | `hashWithTag()` value |
|---|---|---|
| Customizer | UTF-8 of `"H2C"` | UTF-8 of `"H2C"` |
| KMAC key | UTF-8 of `"FLOW--V00-CS00-with-BLS_SIG_BLS12381G1_XOF:KMAC128_SSWU_RO_POP_"` | UTF-8 of `"FLOW-" || tag || "-V00-CS00-with-BLS_SIG_BLS12381G1_XOF:KMAC128_SSWU_RO_POP_"` |
| Output length | 128 bytes | 128 bytes |

The 128-byte output feeds into the hash-to-curve algorithm (simplified SWU mapping per
RFC draft-irtf-cfrg-hash-to-curve-14). It is **not** a general-purpose hash for arbitrary
data. Use it only when implementing BLS-compatible hashing.

### When to provide a domain separation tag

The tag in `hashWithTag` is embedded into the KMAC key string. This means:
- An empty tag and a non-empty tag produce **completely different hash outputs**.
- Within a BLS signing system, all signers and verifiers must agree on the exact tag.
- Flow's internal BLS uses a specific tag embedded in the key constant above. If you are
  interoperating with Flow's beacon or consensus, use `hash()` (empty tag) unless you have
  confirmed the exact tag in use.

### Direct hashing example

```cadence
// Hash data for use as BLS pre-image material
let data: [UInt8] = [0x01, 0x02, 0x03, 0x04]

// No-tag variant — matches Flow's internal BLS hash
let hashNoTag = HashAlgorithm.KMAC128_BLS_BLS12_381.hash(data)
// hashNoTag.length == 128

// Tagged variant — adds domain separation
let hashTagged = HashAlgorithm.KMAC128_BLS_BLS12_381.hashWithTag(
    data,
    tag: "my-bls-app-v1"
)
// hashTagged.length == 128, with different bytes than hashNoTag
```

You rarely call `KMAC128_BLS_BLS12_381.hash()` directly — the runtime invokes it
internally when you call `key.verify(..., hashAlgorithm: HashAlgorithm.KMAC128_BLS_BLS12_381)`.
Use it directly only when you need to compute the pre-image hash outside of a verify call,
for example when implementing custom BLS-based commitment schemes.

---

## Common Pitfalls

### ❌ Pitfall 1 — Wrong or missing `domainSeparationTag`

`verify()` never throws; a tag mismatch looks identical to a forged signature — it simply
returns `false`. If your contract calls `verify()` and always gets `false` despite using
correct keys and data, a tag mismatch is the most common cause.

```cadence
// ❌ Signed off-chain with tag "my-app-v1" but verified with empty tag
let isValid = key.verify(
    signature: sig,
    signedData: data,
    domainSeparationTag: "",       // mismatch — will always return false
    hashAlgorithm: HashAlgorithm.SHA2_256
)

// ✅ Tags must match on both sides
let isValid = key.verify(
    signature: sig,
    signedData: data,
    domainSeparationTag: "my-app-v1",
    hashAlgorithm: HashAlgorithm.SHA2_256
)
```

### ❌ Pitfall 2 — Using SHA3_256 expecting Ethereum-compatible behavior

`SHA3_256` is the NIST-standardized SHA-3 algorithm. Ethereum uses the **original Keccak
submission** (pre-standardization), which has a different padding rule. They produce
different digests for the same input.

```cadence
// ❌ Will not match Ethereum's keccak256() output
let digest = HashAlgorithm.SHA3_256.hash(data)

// ✅ Matches Ethereum's keccak256()
let digest = HashAlgorithm.KECCAK_256.hash(data)
```

This affects any cross-chain flow where you compute a hash on-chain in Cadence and compare
it to a hash from a Solidity contract or Ethereum library.

### ❌ Pitfall 3 — Aggregating BLS signatures without checking proof of possession

Omitting `verifyPoP` before aggregation opens the *rogue key attack*: an attacker can craft
a public key that, when aggregated with legitimate keys, makes the combined key validate
arbitrary messages.

```cadence
// ❌ No PoP check — vulnerable to rogue key attack
let aggKey = BLS.aggregatePublicKeys([pk1, pk2, pk3])
let aggSig = BLS.aggregateSignatures([sig1, sig2, sig3])
let valid = aggKey!.verify(signature: aggSig!, ...)

// ✅ Verify PoP for every key before aggregating
assert(pk1.verifyPoP(pop1), message: "PoP check failed")
assert(pk2.verifyPoP(pop2), message: "PoP check failed")
assert(pk3.verifyPoP(pop3), message: "PoP check failed")
let aggKey = BLS.aggregatePublicKeys([pk1, pk2, pk3])
```

### ❌ Pitfall 4 — Confusing `verify()` returning `false` with a thrown error

`PublicKey.verify()` is a `view` function that returns `Bool`. It does **not** panic on bad
signature bytes, corrupted data, or wrong-length inputs. If you expect an error and do not
check the return value, invalid signatures silently pass authorization checks.

```cadence
// ❌ Return value ignored — authorization bypass
key.verify(signature: sig, signedData: data, domainSeparationTag: "", hashAlgorithm: HashAlgorithm.SHA2_256)
executePrivilegedAction()  // runs unconditionally

// ✅ Always check the return value
let valid = key.verify(signature: sig, signedData: data, domainSeparationTag: "", hashAlgorithm: HashAlgorithm.SHA2_256)
assert(valid, message: "Signature verification failed")
executePrivilegedAction()
```

### ❌ Pitfall 5 — Matching an Ethereum `eth_sign` hash without the prefix

`eth_sign` prepends `"\x19Ethereum Signed Message:\n<message_length>"` to the message before
signing. If you pass the raw message bytes to Cadence without this prefix, verification will
fail even with the correct key and `KECCAK_256`.

```cadence
// ❌ Raw bytes — does not match eth_sign output
let isValid = key.verify(
    signature: sig,
    signedData: rawMessage,
    domainSeparationTag: "",
    hashAlgorithm: HashAlgorithm.KECCAK_256
)

// ✅ Include Ethereum's signing prefix — must be assembled off-chain or in the transaction
// Prefix: 0x19 + "Ethereum Signed Message:\n" + decimal length of message
// The prefix concatenation must be done before calling verify().
```

### ❌ Pitfall 6 — Calling `verifyPoP()` on a non-BLS key

`verifyPoP()` is only valid for `BLS_BLS12_381` keys. Calling it on `ECDSA_P256` or
`ECDSA_secp256k1` causes the program to **abort**.

```cadence
// ❌ Aborts at runtime
let ecdsaKey = PublicKey(
    publicKey: ecdsaBytes,
    signatureAlgorithm: SignatureAlgorithm.ECDSA_P256
)
ecdsaKey.verifyPoP(someBytes)   // program aborts

// ✅ Only call verifyPoP on BLS keys
if key.signatureAlgorithm == SignatureAlgorithm.BLS_BLS12_381 {
    assert(key.verifyPoP(pop))
}
```

---

## Summary: Algorithm Pairing Guide

| Off-chain signer | Recommended `signatureAlgorithm` | Recommended `hashAlgorithm` | `domainSeparationTag` |
|---|---|---|---|
| Ethereum wallet (`eth_sign`) | `ECDSA_secp256k1` | `KECCAK_256` | `""` (Ethereum prefix in data) |
| Ethereum wallet (EIP-712) | `ECDSA_secp256k1` | `KECCAK_256` | `""` (struct hash in data) |
| WebAuthn / passkey / FIDO2 | `ECDSA_P256` | `SHA2_256` | `""` (WebAuthn structure in data) |
| Flow account key (default) | `ECDSA_P256` | `SHA3_256` | `"FLOW-V0.0-user"` (Flow signing domain) |
| Flow account key (secp variant) | `ECDSA_secp256k1` | `SHA3_256` | `"FLOW-V0.0-user"` |
| BLS oracle / threshold | `BLS_BLS12_381` | `KMAC128_BLS_BLS12_381` | Application-specific, agree across all signers |

---

## See Also

- [randomness.md](randomness.md) — BLS is used internally by Flow's randomness beacon;
  `RandomBeaconHistory` and `RandomConsumer` build on the same BLS12-381 infrastructure
- [access-control.md](access-control.md) — Crypto primitives fit into authorization flows
  and capability-based access models
- [../../cadence-audit/references/audit-checklist.md](../../cadence-audit/references/audit-checklist.md) —
  signature verification is an audit hotspot: check tag matching, PoP for BLS, return-value
  handling, and Ethereum encoding compatibility
