# EVM Precompiles in Flow EVM

Flow EVM inherits the full Ethereum precompile set from the underlying go-ethereum implementation, giving every Solidity or Yul contract access to cryptographic primitives at fixed addresses without deploying bytecode. On top of the standard set, Flow EVM injects a Flow-specific extended precompile — the **Cadence Arch** contract — at address `0x0000000000000000000000010000000000000001`. Cadence Arch is the unique differentiator: a Solidity contract (or a Cadence transaction via `coa.call`) can call it to read the current Flow block height, draw a revertible random number, query a historical randomness seed, or verify that an EVM address is controlled by a particular Flow account (COA ownership proof). All without a bridge, an oracle, or an off-chain relayer. When you call a precompile from Cadence, use exactly the same `coa.call` / `EVM.dryCall` mechanics documented in [evm-call.md](evm-call.md); the only difference is the target address and the ABI-encoded input.

For questions about the Cadence-side cryptographic equivalents (native SHA-2, signature verification), see [../../cadence-lang/references/crypto.md](../../cadence-lang/references/crypto.md). For CU budget implications of repeated precompile calls, see [cu-ceiling.md](cu-ceiling.md).

---

## Flow EVM Fork Status

Flow EVM runs on go-ethereum's chain configuration with all forks from Homestead through Osaka applied. The activation schedule below comes directly from `fvm/evm/emulator/config.go` in the flow-go repository (source: commit read on 2026-05-17).

| Ethereum Fork | Flow EVM Status | Activation notes |
|---|---|---|
| Homestead | Active (genesis) | Block 0 |
| Tangerine Whistle (EIP-150) | Active (genesis) | Block 0 |
| Spurious Dragon (EIP-155/158) | Active (genesis) | Block 0 |
| Byzantium | Active (genesis) | Block 0 — adds 0x05–0x08 precompiles |
| Constantinople / Petersburg | Active (genesis) | Block 0 |
| Istanbul | Active (genesis) | Block 0 — adds 0x09 BLAKE2F precompile |
| Berlin | Active (genesis) | Block 0 — all precompile addresses "warm" |
| London | Active (genesis) | Block 0 |
| Shanghai | Active (genesis) | Timestamp 0 |
| Cancun | Active (genesis) | Timestamp 0 — adds 0x0A KZG point evaluation |
| Prague (Pectra) | Active on all networks | Mainnet 2025-05-15; Testnet 2025-05-08; Previewnet genesis — adds 0x0B–0x11 BLS12-381 precompiles |
| Osaka | Active on all networks | Mainnet 2025-12-03; Testnet 2025-11-19; Previewnet genesis |
| Verkle | Not scheduled (`VerkleTime: nil`) | |

Implications: as of 2026, every standard Ethereum precompile from 0x01 through 0x11 is available on all Flow EVM networks. The extended Cadence Arch precompile is always present at all fork levels.

---

## Ethereum Standard Precompiles

All addresses below are canonical Ethereum precompile addresses. They use the `FlowEVMNativePrecompileAddressPrefix` (12 leading zero bytes) in Flow EVM's internal address scheme. Reserved address range: **0x01–0xFF** is reserved for native precompiles — do not deploy user contracts to addresses in this range.

### Summary Table

| Address | Name | Fork | Gas |
|---|---|---|---|
| `0x01` | ECRECOVER | Frontier | 3,000 |
| `0x02` | SHA256 | Frontier | 60 + 12/word |
| `0x03` | RIPEMD-160 | Frontier | 600 + 120/word |
| `0x04` | IDENTITY (datacopy) | Frontier | 15 + 3/word |
| `0x05` | MODEXP | Byzantium (EIP-198) | formula (see below) |
| `0x06` | ECADD (bn256) | Byzantium (EIP-196) | 150 (post-Istanbul) |
| `0x07` | ECMUL (bn256) | Byzantium (EIP-196) | 6,000 (post-Istanbul) |
| `0x08` | ECPAIRING (bn256) | Byzantium (EIP-197) | 45,000·k + 34,000 |
| `0x09` | BLAKE2F | Istanbul (EIP-152) | 1 per round |
| `0x0A` | POINT_EVALUATION (KZG) | Cancun (EIP-4844) | 50,000 |
| `0x0B` | BLS12_G1ADD | Prague (EIP-2537) | 375 |
| `0x0C` | BLS12_G1MSM | Prague (EIP-2537) | variable |
| `0x0D` | BLS12_G2ADD | Prague (EIP-2537) | 600 |
| `0x0E` | BLS12_G2MSM | Prague (EIP-2537) | variable |
| `0x0F` | BLS12_PAIRING_CHECK | Prague (EIP-2537) | 32,600·k + 37,700 |
| `0x10` | BLS12_MAP_FP_TO_G1 | Prague (EIP-2537) | 5,500 |
| `0x11` | BLS12_MAP_FP2_TO_G2 | Prague (EIP-2537) | 23,800 |

Gas costs are EVM gas — not Cadence CU. A `coa.call` to a precompile costs Cadence CU for the bridge overhead plus the EVM gas is consumed from the COA's FLOW balance. See [cu-ceiling.md](cu-ceiling.md) for the CU-per-hop baseline.

### `0x01` — ECRECOVER

Recovers the Ethereum address that signed a message hash using an ECDSA signature (secp256k1 curve). Returns the signer address as a 32-byte right-aligned value, or 32 zero bytes on failure (never reverts).

| Field | Details |
|---|---|
| Input | 128 bytes: `hash` (32) + `v` (32, must be 27 or 28) + `r` (32) + `s` (32) |
| Output | 32 bytes: right-aligned 20-byte Ethereum address, or zero on failure |
| Gas | 3,000 (flat) |
| Fork | Frontier |
| Flow EVM | Available at genesis |

### `0x02` — SHA256

Computes the SHA-256 hash of arbitrary input data.

| Field | Details |
|---|---|
| Input | arbitrary bytes |
| Output | 32 bytes: SHA-256 hash |
| Gas | 60 + 12 per 32-byte word (rounded up) |
| Fork | Frontier |
| Flow EVM | Available at genesis |

### `0x03` — RIPEMD-160

Computes the RIPEMD-160 hash of arbitrary input data.

| Field | Details |
|---|---|
| Input | arbitrary bytes |
| Output | 32 bytes: right-aligned 20-byte RIPEMD-160 hash |
| Gas | 600 + 120 per 32-byte word (rounded up) |
| Fork | Frontier |
| Flow EVM | Available at genesis |

### `0x04` — IDENTITY (datacopy)

Returns its input unchanged. Used in ABI-level data manipulation (cheap memory copy via `staticcall`).

| Field | Details |
|---|---|
| Input | arbitrary bytes |
| Output | same bytes |
| Gas | 15 + 3 per 32-byte word (rounded up) |
| Fork | Frontier |
| Flow EVM | Available at genesis |

### `0x05` — MODEXP (EIP-198)

Modular exponentiation: computes `(BASE ** EXPONENT) % MODULUS`.

| Field | Details |
|---|---|
| Input | 3×32-byte length headers (Blen, Elen, Mlen) + BASE + EXPONENT + MODULUS (each right-padded to declared length) |
| Output | `(BASE**EXPONENT) % MODULUS` as bytes, same length as MODULUS |
| Gas | `floor(mult_complexity(max(Mlen,Blen)) * max(adjExpLen,1) / 3)` (EIP-2565 formula) |
| Fork | Byzantium (EIP-198), gas repriced at Berlin (EIP-2565) |
| Flow EVM | Available at genesis |

### `0x06` — ECADD (EIP-196)

Point addition on the alt_bn128 (bn256) curve, used in zero-knowledge proofs.

| Field | Details |
|---|---|
| Input | 128 bytes: two G1 points `(x1,y1,x2,y2)` as 32-byte big-endian field elements |
| Output | 64 bytes: resulting G1 point `(x,y)` |
| Gas | 150 (post-Istanbul repricing from EIP-1108) |
| Fork | Byzantium (EIP-196) |
| Flow EVM | Available at genesis |

### `0x07` — ECMUL (EIP-196)

Scalar multiplication on the alt_bn128 (bn256) curve.

| Field | Details |
|---|---|
| Input | 96 bytes: G1 point `(x,y)` (64 bytes) + scalar `s` (32 bytes big-endian) |
| Output | 64 bytes: resulting G1 point `(x,y)` |
| Gas | 6,000 (post-Istanbul repricing from EIP-1108) |
| Fork | Byzantium (EIP-196) |
| Flow EVM | Available at genesis |

### `0x08` — ECPAIRING (EIP-197)

Bilinear pairing check on alt_bn128. Inputs an array of (G1, G2) pairs and returns 1 if the pairing product equals zero in the field, 0 otherwise.

| Field | Details |
|---|---|
| Input | 192·k bytes: k pairs of `(G1_point, G2_point)` — empty input is valid and returns 1 |
| Output | 32 bytes: `1` (true) or `0` (false) |
| Gas | 45,000·k + 34,000 (post-Istanbul repricing from EIP-1108) |
| Fork | Byzantium (EIP-197) |
| Flow EVM | Available at genesis |

### `0x09` — BLAKE2F (EIP-152)

Executes rounds of the BLAKE2b compression function F. Enables Zcash and BLAKE2-based interop.

| Field | Details |
|---|---|
| Input | exactly 213 bytes: `rounds` (4 bytes big-endian u32) + state `h` (64 bytes, 8×u64 LE) + message `m` (128 bytes, 16×u64 LE) + offset counters `t` (16 bytes, 2×u64 LE) + final flag `f` (1 byte, 0 or 1) |
| Output | 64 bytes: updated state `h` (same LE encoding) |
| Gas | 1 per round |
| Fork | Istanbul (EIP-152) |
| Flow EVM | Available at genesis |

### `0x0A` — POINT_EVALUATION (EIP-4844)

Verifies a KZG proof that a blob (represented by a commitment) evaluates to a specific value at a given point. Used for EIP-4844 blob transaction data availability proofs.

| Field | Details |
|---|---|
| Input | 192 bytes: `versioned_hash` (32) + evaluation point `z` (32, big-endian) + claimed value `y` (32, big-endian) + KZG commitment (48) + KZG proof (48) |
| Output | 64 bytes: `FIELD_ELEMENTS_PER_BLOB` (32 bytes) + `BLS_MODULUS` (32 bytes) |
| Gas | 50,000 (flat) |
| Fork | Cancun (EIP-4844) |
| Flow EVM | Available at genesis (Cancun is applied at timestamp 0) |

Note: Flow EVM does not include a blob mempool or data availability layer — there are no blob-carrying transactions in Flow EVM. The POINT_EVALUATION precompile is present (because go-ethereum includes it at the Cancun level), but there are no network-level blobs to verify. A direct call to `0x0A` with a synthetically constructed valid KZG proof returns the correct 64-byte response (`FIELD_ELEMENTS_PER_BLOB || BLS_MODULUS`). Invalid proofs return `status == .failed` with `'mismatched versioned hash'` or `'invalid input length'`. Normal EIP-4844 use cases (blob sidecar verification) do not apply.

### BLS12-381 Precompiles (EIP-2537) — Prague

All seven BLS12-381 precompiles added by EIP-2537 are active on all Flow EVM networks. These precompiles enable BLS signature verification and zero-knowledge circuit operations over the BLS12-381 curve, which is used by Ethereum's consensus layer (Beacon Chain).

| Address | Name | Input | Output | Gas |
|---|---|---|---|---|
| `0x0B` | BLS12_G1ADD | 256 bytes (two G1 points) | 128 bytes (G1 result) | 375 |
| `0x0C` | BLS12_G1MSM | 160·k bytes (k point-scalar pairs) | 128 bytes (G1 result) | variable (k-dependent discount table) |
| `0x0D` | BLS12_G2ADD | 512 bytes (two G2 points) | 256 bytes (G2 result) | 600 |
| `0x0E` | BLS12_G2MSM | 288·k bytes (k point-scalar pairs) | 256 bytes (G2 result) | variable (k-dependent discount table) |
| `0x0F` | BLS12_PAIRING_CHECK | 384·k bytes (k G1+G2 pairs) | 32 bytes (1 or 0) | 32,600·k + 37,700 |
| `0x10` | BLS12_MAP_FP_TO_G1 | 64 bytes (base field element) | 128 bytes (G1 point) | 5,500 |
| `0x11` | BLS12_MAP_FP2_TO_G2 | 128 bytes (extension field element) | 256 bytes (G2 point) | 23,800 |

BLS12-381 G1/G2 point encoding follows EIP-2537's 48-byte compressed / 96-byte uncompressed conventions. For MSM operations the discount table from EIP-2537 applies — gas decreases per-pair as k grows.

---

## The Cadence Arch Precompile

Cadence Arch is Flow EVM's **extended precompile** — not part of the Ethereum standard, deployed only on Flow EVM. It is the most important feature in this document for Flow developers: it lets a Solidity contract or a Cadence transaction read live Cadence state (block height, randomness, COA identity) from inside the EVM without any oracle, relayer, or bridge. The call is synchronous, atomic, and costs only a small fixed amount of EVM gas.

### Address

```
0x0000000000000000000000010000000000000001
```

This address is derived deterministically by the Flow EVM runtime's address allocator:
- Bytes 0–11: `FlowEVMExtendedPrecompileAddressPrefix` = `0x000000000000000000000001`
- Bytes 12–19: big-endian `uint64(1)` (index 1, the first extended precompile)

User contracts should never be deployed to addresses with the prefix `0x000000000000000000000001` — these are reserved for Flow-extended precompiles. Addresses `0x0000000000000000000000020000000000000000` and above are COA addresses.

**Source**: `fvm/evm/handler/precompiles.go` (flow-go), `fvm/evm/precompiles/arch.go` (flow-go), confirmed in Flow developer documentation.

### Methods

#### `flowBlockHeight() returns (uint64)`

Returns the current Flow (Cadence) block height at the time the transaction executes.

| Field | Details |
|---|---|
| Selector | `0x53e87d66` |
| Input | none (0 bytes after selector) |
| Output | 32 bytes: right-aligned `uint64` block height (EVM-standard 256-bit word) |
| Gas | 2 (matches the EVM `NUMBER` opcode cost) |

The returned value is the **Cadence block height**, not an EVM block number. EVM block numbers on Flow EVM are separate; use `block.number` inside Solidity for the EVM block. Use `flowBlockHeight()` when your contract needs to key on the underlying Flow consensus block.

#### `getRandomSource(uint64 height) returns (bytes32)`

Returns the 32-byte random source (from Flow's Random Beacon history) for the given Flow block height. The source is committed before the block is produced, making it safe for commit-reveal schemes.

| Field | Details |
|---|---|
| Selector | `0x78a75fbe` |
| Input | 32 bytes: ABI-encoded `uint64` block height (right-aligned in a 256-bit word) |
| Output | 32 bytes: `bytes32` random source |
| Gas | 1,000 |

The `height` argument must be a block that has already been finalized. Future heights and the current block return `status == .failed` with `errorMessage` containing `'Source of randomness not yet recorded'`. Block 0 (before genesis) returns `'Requested block height precedes recorded history'`. Both errors embed the full Cadence panic trace in `errorMessage`.

#### `revertibleRandom() returns (uint64)`

Returns a pseudo-random `uint64` derived from the transaction's execution context. Unlike `getRandomSource`, this value can be influenced by the transaction sequence within a block (miners/validators can reorder). It is suitable for applications that only need statistical randomness and can tolerate revert-based manipulation (the caller can revert if the value is unfavorable).

| Field | Details |
|---|---|
| Selector | `0x705fab20` |
| Input | none (0 bytes after selector) |
| Output | 32 bytes: right-aligned `uint64` random value |
| Gas | 1,000 |

For security-sensitive randomness (lotteries, NFT reveal), prefer `getRandomSource` with a commit-reveal pattern. Use `revertibleRandom` for non-critical shuffles or distributions where the stakes are low.

#### `verifyCOAOwnershipProof(address coaAddress, bytes32 signedData, bytes encodedProof) returns (bool)`

Verifies that a given EVM address is a COA controlled by a specific Flow account. This is the cross-VM identity bridge: a Solidity contract can confirm that the EOA or COA calling it is actually backed by a particular Flow account without any off-chain oracle.

| Field | Details |
|---|---|
| Selector | `0x5ee837e7` |
| Input (after selector) | ABI-encoded `(address coaAddress, bytes32 signedData, bytes encodedProof)` — the `encodedProof` is a variable-length byte array containing the COA ownership proof as produced by `EVM.validateCOAOwnershipProof` |
| Output | 32 bytes: `bool` — `1` for valid, `0` for invalid |
| Gas | 1,000 base + 3,000 per signature in the proof |

The gas formula for `verifyCOAOwnershipProof` scales with the number of Flow account keys included in the proof, matching the `ECRECOVER` cost (3,000 gas) per signature. For a single-key Flow account the cost is 4,000 gas total.

Generating the `encodedProof` bytes happens on the Cadence side before calling this precompile — the Flow account owner calls an EVM contract (or a `coa.call` path) that invokes `verifyCOAOwnershipProof`, passing a proof bundle produced by their wallet.

---

## Worked Examples

### Example 1 — ECRECOVER from a Cadence Transaction

Use case: verify an off-chain ECDSA signature against a message hash before proceeding with on-chain logic. The signature was created by an Ethereum EOA.

```cadence
import "EVM"

// Verifies an ECDSA secp256k1 signature and returns the recovered signer address.
// Parameters are in the raw-byte form that ECRECOVER expects.
transaction(
    msgHash: [UInt8],   // 32 bytes — keccak256 of the signed message
    v: UInt8,           // 27 or 28
    r: [UInt8],         // 32 bytes
    s: [UInt8]          // 32 bytes
) {
    prepare(signer: auth(BorrowValue) &Account) {
        let coa = signer.storage.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(
            from: /storage/evm
        ) ?? panic("No COA at /storage/evm")

        // Build the 128-byte ECRECOVER input.
        // The precompile is a raw-byte interface — no function selector.
        // Layout: hash(32) | v_padded(32) | r(32) | s(32)
        var input: [UInt8] = []
        // Pad msgHash to 32 bytes
        assert(msgHash.length == 32, message: "msgHash must be exactly 32 bytes")
        input = input.concat(msgHash)
        // v is right-padded in a 32-byte word
        var vPadded = [UInt8](repeating: 0, count: 32)
        vPadded[31] = v
        input = input.concat(vPadded)
        input = input.concat(r)
        input = input.concat(s)
        assert(input.length == 128, message: "ECRECOVER input must be 128 bytes")

        let ecRecoverAddr = EVM.addressFromString("0x0000000000000000000000000000000000000001")

        // ECRECOVER is read-only; EVM.dryCall is sufficient and requires no COA auth.
        // Use the COA address as `from` to avoid msg.sender zero-address quirks.
        // Minimum gasLimit on Flow EVM is 21,064 (the EIP-3860 intrinsic minimum).
        // Use 100,000 as a safe default for precompile calls.
        // (ECRECOVER's own precompile charge is 3,000 gas; the extra covers the EVM tx envelope.)
        let result = EVM.dryCall(
            from: coa.address(),
            to: ecRecoverAddr,
            data: input,
            gasLimit: 100_000,
            value: EVM.Balance(attoflow: 0)
        )

        assert(
            result.status == EVM.Status.successful,
            message: "ECRECOVER failed: ".concat(result.errorMessage)
        )

        // On invalid signature, ECRECOVER returns empty data (length 0), not zeros.
        // (This diverges from the "32 zero bytes" description in the Solidity yellow paper;
        //  Flow EVM v2.17.1 returns 0 bytes — data.length == 0 — on any invalid signature.)
        if result.data.length == 0 {
            // Invalid signature — no address recovered.
            return
        }

        // Valid signature: output is 32 bytes, right-aligned 20-byte address.
        assert(result.data.length == 32, message: "Unexpected ECRECOVER output length")

        let recoveredBytes = result.data.slice(from: 12, upTo: 32) // last 20 bytes
        let recovered = EVM.EVMAddress(bytes: recoveredBytes.toConstantSized<[UInt8; 20]>()!)

        log("Recovered signer: ".concat(recovered.toString()))
        // Compare recovered against expected address to verify the signature.
    }
}
```

**When to use vs Cadence native crypto**: `HashAlgorithm.KECCAK_256` and `PublicKey.verify` in Cadence can verify secp256k1 signatures natively — no EVM call needed. Use `0x01` ECRECOVER when the hash was produced by EVM-standard keccak and you need the address format, or when calling from inside a Solidity contract.

---

### Example 2 — Cadence Arch `flowBlockHeight` from Cadence

Use case: a Cadence script reads the current Cadence block height via the EVM precompile path. This verifies the precompile is wired correctly and demonstrates the pattern any Solidity contract would use internally.

```cadence
import "EVM"

// Queries the Cadence Arch precompile to get the current Flow block height.
// This is a read-only view — EVM.dryCall with no COA required.
access(all) fun main(): UInt64 {
    let archAddr = EVM.addressFromString("0x0000000000000000000000010000000000000001")
    let zero = EVM.addressFromString("0x0000000000000000000000000000000000000000")

    // flowBlockHeight() — selector 0x53e87d66, no arguments.
    let selector: [UInt8] = [0x53, 0xe8, 0x7d, 0x66]

    let result = EVM.dryCall(
        from: zero,
        to: archAddr,
        data: selector,
        gasLimit: 100_000,   // 2 gas precompile cost; 100k covers the 21,064 EVM intrinsic floor
        value: EVM.Balance(attoflow: 0)
    )

    assert(
        result.status == EVM.Status.successful,
        message: "Cadence Arch flowBlockHeight call failed: ".concat(result.errorMessage)
    )

    // Return value is a 32-byte word (EVM 256-bit) containing the uint64 height.
    assert(result.data.length == 32, message: "Unexpected return length from flowBlockHeight")

    let decoded = EVM.decodeABI(types: [Type<UInt256>()], data: result.data)
    let heightU256 = decoded[0] as! UInt256
    // Safe cast: Flow block height fits in UInt64
    return UInt64(heightU256)
}
```

You can also call `flowBlockHeight()` from inside a Solidity contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface ICadenceArch {
    function flowBlockHeight() external view returns (uint64);
    function getRandomSource(uint64 height) external view returns (bytes32);
    function revertibleRandom() external view returns (uint64);
    function verifyCOAOwnershipProof(
        address coaAddress,
        bytes32 signedData,
        bytes memory encodedProof
    ) external view returns (bool);
}

contract FlowBlockReader {
    ICadenceArch constant ARCH = ICadenceArch(0x0000000000000000000000010000000000000001);

    function currentFlowBlock() external view returns (uint64) {
        return ARCH.flowBlockHeight();
    }

    function randomAtBlock(uint64 height) external view returns (bytes32) {
        return ARCH.getRandomSource(height);
    }
}
```

To call this deployed `FlowBlockReader` from Cadence, use the standard `coa.call` or `EVM.dryCall` pattern from [evm-call.md](evm-call.md). The Cadence Arch call is abstracted inside the Solidity layer; from Cadence's perspective it is just a call to the deployed contract address.

---

### Example 3 — SHA256 via Precompile vs Native Cadence

Both options hash data with SHA-256. Choose based on context.

#### Option A — 0x02 SHA256 precompile from Cadence (EVM path)

```cadence
import "EVM"

// Returns SHA-256 hash of `data` by calling the EVM SHA256 precompile.
// ✅ Use when: computing hash inside EVM for interop with Solidity contracts
//             that rely on the same hash.
// ❌ Avoid when: hashing purely Cadence data — use Cadence native crypto instead.
access(all) fun sha256ViaPrecompile(data: [UInt8]): [UInt8] {
    let sha256Addr = EVM.addressFromString("0x0000000000000000000000000000000000000002")
    let zero = EVM.addressFromString("0x0000000000000000000000000000000000000000")

    // SHA256 precompile: no function selector — raw input only.
    // Precompile cost: 60 + 12 * ceil(len/32). For 128 bytes: 60 + 12*4 = 108 gas.
    // However, the EVM intrinsic gas floor is 21,064 — values below that trigger
    // "intrinsic gas too low: have N, want 21064". Use a flat 100,000 as a safe default.
    let result = EVM.dryCall(
        from: zero,
        to: sha256Addr,
        data: data,
        gasLimit: 100_000,
        value: EVM.Balance(attoflow: 0)
    )

    assert(result.status == EVM.Status.successful, message: "SHA256 precompile failed")
    assert(result.data.length == 32, message: "SHA256 must return 32 bytes")
    return result.data
}
```

#### Option B — Native Cadence hash (no EVM hop)

```cadence
// ✅ Preferred for pure Cadence logic — no EVM overhead, no CU for bridge layer.
let hash: [UInt8] = HashAlgorithm.SHA2_256.hash(data)
```

#### Comparison

| Criterion | `0x02` Precompile (EVM) | `HashAlgorithm.SHA2_256` (Cadence) |
|---|---|---|
| Output | identical 32-byte SHA-256 digest | identical 32-byte SHA-256 digest |
| Cadence CU | ~16 CU for EVM.dryCall overhead | ~5 CU (native op) |
| EVM gas charged | no (dryCall; gas consumed but not paid) | N/A |
| Needs COA | no (`EVM.dryCall`) | no |
| Use from script | yes | yes |
| Use inside Solidity | yes (via `0x02` call) | no |

✅ Use `HashAlgorithm.SHA2_256` for anything that stays in Cadence.  
✅ Use `0x02` when you need the same hash result that an EVM-side `sha256(data)` call would produce — for example, verifying a hash committed inside a Solidity contract.  
❌ Do not use the precompile path just to hash data in a Cadence script — the EVM bridge overhead is wasteful.

---

## Reserved Address Ranges

Flow EVM reserves specific address ranges for its runtime infrastructure. Never deploy user contracts to these ranges.

| Range | Prefix (bytes 0–11) | Use |
|---|---|---|
| `0x0000...0001` – `0x0000...00FF` | all zeros | Ethereum native precompiles (0x01–0x11 currently used, 0x12–0xFF reserved) |
| `0x000000000000000000000001_00000000` – `0x000000000000000000000001_FFFFFFFF...` | `000000000000000000000001` | Flow extended precompiles (Cadence Arch is index 1; future precompiles use higher indices) |
| `0x000000000000000000000002_xxxxxxxx` | `000000000000000000000002` | COA addresses (deterministically allocated by the EVM runtime) |
| `0x000000000000000000000003_00000000` | `000000000000000000000003` | Coinbase address |

**Safe deployment range**: addresses that do not begin with any of the above prefixes. In practice, any EVM address generated by normal contract deployment (via `CREATE` from a funded EOA or via a factory contract) will fall outside these ranges.

---

## Common Pitfalls

1. **Calling precompiles with a function selector when none is expected.** Native precompiles (0x01–0x11) do NOT use ABI selectors — they consume raw calldata. If you build calldata with `EVM.encodeABIWithSignature("sha256(bytes)", [...])` for the SHA256 precompile, the first 4 bytes will be treated as input data, not a dispatcher — the result will be wrong. Build raw bytes only, or call via a Solidity wrapper that does the same.

2. **Calling Cadence Arch without a selector.** The reverse of the above: Cadence Arch IS a multi-function precompile and DOES require the 4-byte function selector (e.g. `[0x53, 0xe8, 0x7d, 0x66]` for `flowBlockHeight`). Missing the selector returns an `ErrInvalidMethodCall` result.

3. **Expecting ECRECOVER to revert on bad signatures.** ECRECOVER never reverts. If the signature is invalid, it returns **empty data** (`result.data.length == 0`), not 32 zero bytes. Always check `result.data.length == 0` before slicing the result — do not rely on `result.status == .failed` or assume the output is 32 bytes.

4. **Treating `0x0A` POINT_EVALUATION as a blob availability check for user data.** Flow EVM does not have a blob mempool. POINT_EVALUATION can be called with a crafted proof, and a synthetically valid KZG proof returns the correct 64-byte response on the emulator (confirmed). However, there is no mechanism to submit actual EIP-4844 blobs to Flow EVM.

5. **Using `revertibleRandom` for high-stakes randomness.** `revertibleRandom` produces a value that is determined at execution time — a miner or validator choosing transaction ordering can influence it. For lotteries or NFT minting, use `getRandomSource(height)` with a commit-reveal scheme to obtain the random seed from a committed beacon source.

6. **Querying `getRandomSource` with the current block height.** The random source for the current Cadence block has not yet been committed. Querying the current block or any future block returns `status == .failed` with `'Source of randomness not yet recorded'`. Always pass `height - 1` or earlier. Block 0 returns `'Requested block height precedes recorded history'`.

7. **Gas estimation for `verifyCOAOwnershipProof` with multi-key accounts.** The gas cost scales as `1,000 + 3,000 * numSignatures`. A 3-of-5 multi-key Flow account proof costs `1,000 + 3 * 3,000 = 10,000` gas. Use `coa.dryCall` to estimate before the live call.

8. **Deploying a contract at an extended precompile address.** The EVM runtime injects Cadence Arch at its address during environment setup; it is not a deployed contract in the EVM state trie. Attempting to `eth_getCode` at `0x00...010000000000000001` will return empty bytes. This is expected — the precompile exists at the runtime level, not the storage level.

---

## See Also

- [evm-call.md](evm-call.md) — canonical `coa.call` / `EVM.dryCall` mechanics, calldata layout, `result.status` discipline.
- [cu-ceiling.md](cu-ceiling.md) — how EVM gas interacts with the Cadence CU budget; precompile calls still consume bridge-layer CU.
- [../../cadence-lang/references/crypto.md](../../cadence-lang/references/crypto.md) — Cadence-native SHA-2, SHA-3, RIPEMD-160, and ECDSA verification without going through the EVM.
