# EVM Call Pattern from Cadence

Crossing the VM boundary from Cadence into Flow EVM is the core of CrossVM development. A single Cadence transaction can invoke an EVM contract atomically — either the whole transaction (Cadence side and EVM side together) commits, or nothing does. The mechanism is the `EVM` standard contract, exposed via `import "EVM"`, which gives Cadence three entry points into the EVM: `coa.call` (state-mutating, signed by the COA), `coa.dryCall` (simulate with a COA, no commit), and `EVM.dryCall` (read-only, no COA needed). All three return an `EVM.Result` struct that you MUST inspect — the Cadence transaction will NOT auto-revert when an EVM call fails.

For the COA itself (creating, borrowing, lifecycle, entitlements), see [coa-lifecycle.md](coa-lifecycle.md). For CU budget caveats when batching many EVM calls, see [cu-ceiling.md](cu-ceiling.md).

---

## API surface

All types live under the `EVM` contract. The signatures below come from the Flow Go runtime stdlib.

```cadence
// Top-level read-only call — no COA, no state change, no signature.
access(all) fun dryCall(
    from: EVM.EVMAddress,
    to: EVM.EVMAddress,
    data: [UInt8],
    gasLimit: UInt64,
    value: EVM.Balance,
): EVM.Result

// COA state-mutating call — requires auth(EVM.Call) on the COA reference.
access(Owner | Call) fun call(
    to: EVM.EVMAddress,
    data: [UInt8],
    gasLimit: UInt64,
    value: EVM.Balance,
): EVM.Result

// COA simulation — same as call but does not commit (uses the COA as msg.sender
// so msg.sender-dependent contracts behave correctly).
access(all) fun dryCall(
    to: EVM.EVMAddress,
    data: [UInt8],
    gasLimit: UInt64,
    value: EVM.Balance,
): EVM.Result

// ABI encoding helpers.
access(all) fun encodeABI(_ values: [AnyStruct]): [UInt8]
access(all) fun decodeABI(types: [Type], data: [UInt8]): [AnyStruct]
access(all) fun encodeABIWithSignature(_ signature: String, _ values: [AnyStruct]): [UInt8]
access(all) fun decodeABIWithSignature(_ signature: String, types: [Type], data: [UInt8]): [AnyStruct]
```

### EVM.Result

```cadence
access(all) struct Result {
    access(all) let status: EVM.Status          // unknown | invalid | failed | successful
    access(all) let errorCode: UInt64           // 0 on success; non-zero VM error codes on failure
    access(all) let errorMessage: String        // human-readable revert string when available
    access(all) let gasUsed: UInt64             // EVM gas consumed
    access(all) let data: [UInt8]               // raw return bytes — decode with EVM.decodeABI
    access(all) let deployedContract: EVMAddress?  // populated only for contract deploys
}
```

### EVM.Status

```cadence
access(all) enum Status: UInt8 {
    case unknown      // status not yet resolved (should not surface in normal flows)
    case invalid      // malformed call (bad calldata, gas below intrinsic, decoding failure)
    case failed       // EVM executed and reverted — EVM state rolled back
    case successful   // EVM executed and committed — read result.data
}
```

### EVM.Balance and EVM.EVMAddress

```cadence
// Balance is denominated in attoflow (1 FLOW = 10^18 attoflow).
access(all) struct Balance {
    access(all) var attoflow: UInt
    access(all) view init(attoflow: UInt)
}

// EVM addresses are wrapped in a struct, NOT raw [UInt8;20].
let addr = EVM.addressFromString("0x1234...abcd")  // strip or include "0x" — both accepted
```

---

## Canonical state-mutating call (`coa.call`)

Full transaction calling `transfer(address,uint256)` on an ERC-20.

```cadence
import "EVM"
import "FungibleToken"

transaction(recipientHex: String, amount: UInt256) {
    prepare(signer: auth(BorrowValue) &Account) {
        // 1. Borrow the COA with the Call entitlement.
        let coa = signer.storage.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(
            from: /storage/evm
        ) ?? panic("No COA at /storage/evm — create one first; see coa-lifecycle.md")

        // 2. Build calldata: 4-byte selector || ABI-encoded args.
        //    keccak256("transfer(address,uint256)")[:4] = 0xa9059cbb
        let selector: [UInt8] = [0xa9, 0x05, 0x9c, 0xbb]
        let recipient = EVM.addressFromString(recipientHex)
        let args = EVM.encodeABI([recipient, amount])
        let calldata = selector.concat(args)

        // 3. Resolve the ERC-20 contract address.
        let token = EVM.addressFromString("0xTokenAddressHere")

        // 4. Call — gasLimit is EVM gas, NOT Cadence CU.
        let result = coa.call(
            to: token,
            data: calldata,
            gasLimit: 100_000,
            value: EVM.Balance(attoflow: 0)
        )

        // 5. MUST check status — Cadence does NOT auto-revert on EVM failure.
        assert(
            result.status == EVM.Status.successful,
            message: "ERC20 transfer failed: code=".concat(result.errorCode.toString())
                .concat(" msg=").concat(result.errorMessage)
        )

        // 6. Decode the return value (ERC-20 transfer returns bool).
        let decoded = EVM.decodeABI(types: [Type<Bool>()], data: result.data)
        let ok = decoded[0] as! Bool
        assert(ok, message: "ERC20 transfer returned false")
    }
}
```

### The result-check rule

```cadence
// ❌ WRONG — the EVM call may have reverted; Cadence will still commit
//    every state change made before and after this line.
coa.call(to: addr, data: calldata, gasLimit: 100_000, value: EVM.Balance(attoflow: 0))

// ❌ WRONG — captured but never inspected; same silent failure.
let result = coa.call(...)
// (no check)

// ✅ CORRECT — explicit status check, panic on any non-success.
let result = coa.call(...)
if result.status != EVM.Status.successful {
    panic("EVM call failed [".concat(result.errorCode.toString()).concat("] ")
        .concat(result.errorMessage))
}
```

The single most common CrossVM bug is omitting the `result.status` check.

---

## View call (`EVM.dryCall`)

`EVM.dryCall` is a top-level function that does NOT require a COA. Use it for read-only queries that don't depend on `msg.sender`.

```cadence
import "EVM"

access(all) fun main(tokenHex: String, ownerHex: String): UInt256 {
    let token = EVM.addressFromString(tokenHex)
    let owner = EVM.addressFromString(ownerHex)

    // balanceOf(address) — selector = keccak256("balanceOf(address)")[:4] = 0x70a08231
    let selector: [UInt8] = [0x70, 0xa0, 0x82, 0x31]
    let calldata = selector.concat(EVM.encodeABI([owner]))

    // `from` is the apparent msg.sender. For view calls that ignore msg.sender,
    // the zero address is conventional.
    let zero = EVM.addressFromString("0x0000000000000000000000000000000000000000")

    let result = EVM.dryCall(
        from: zero,
        to: token,
        data: calldata,
        gasLimit: 50_000,
        value: EVM.Balance(attoflow: 0)
    )
    assert(result.status == EVM.Status.successful, message: "balanceOf failed")

    let decoded = EVM.decodeABI(types: [Type<UInt256>()], data: result.data)
    return decoded[0] as! UInt256
}
```

`EVM.dryCall` can be invoked from a script (`access(all) fun main(...)`) or from inside a transaction. Because it does not commit and does not need a signer, it is the cheapest way to read EVM state from Cadence.

---

## Simulated call (`coa.dryCall`)

`coa.dryCall` runs the EVM call AS the COA's EVM address but discards the resulting state. Use it for gas estimation (`result.gasUsed`), permission preflight on contracts that gate on `msg.sender` (where `EVM.dryCall` with an arbitrary `from` gives the wrong answer), and revert-message extraction without paying gas.

```cadence
import "EVM"

transaction(targetHex: String, amount: UInt256) {
    prepare(signer: auth(BorrowValue) &Account) {
        // Plain reference is enough — dryCall does not require auth(EVM.Call).
        let coa = signer.storage.borrow<&EVM.CadenceOwnedAccount>(from: /storage/evm)
            ?? panic("No COA")

        let target = EVM.addressFromString(targetHex)
        let calldata = EVM.encodeABIWithSignature("transfer(address,uint256)", [target, amount])

        // Estimate with a generous ceiling; result.gasUsed tells us actual cost.
        let estimate = coa.dryCall(
            to: target, data: calldata,
            gasLimit: 1_000_000, value: EVM.Balance(attoflow: 0)
        )
        assert(estimate.status == EVM.Status.successful,
               message: "would revert: ".concat(estimate.errorMessage))

        // Pad ~20% headroom; then perform the real call with auth(EVM.Call).
        let gasLimit = estimate.gasUsed + (estimate.gasUsed / UInt64(5))
        // ... real coa.call(...) with the padded gasLimit
    }
}
```

---

## Calldata encoding

Calldata for a Solidity-style function call is always:

```
[ 4-byte selector ] [ ABI-encoded args ]
```

### Selector

The selector is the first 4 bytes of `keccak256(signature)` where signature is the canonical function form with no parameter names. Compute it off-chain and hardcode it.

| Function | Signature | Selector |
|----------|-----------|----------|
| `balanceOf(address)` | `balanceOf(address)` | `0x70a08231` |
| `transfer(address,uint256)` | `transfer(address,uint256)` | `0xa9059cbb` |
| `approve(address,uint256)` | `approve(address,uint256)` | `0x095ea7b3` |
| `transferFrom(address,address,uint256)` | `transferFrom(address,address,uint256)` | `0x23b872dd` |
| `allowance(address,address)` | `allowance(address,address)` | `0xdd62ed3e` |

In Cadence, write the selector as a `[UInt8]` literal:

```cadence
let selector: [UInt8] = [0xa9, 0x05, 0x9c, 0xbb]
```

### Encoding arguments

`EVM.encodeABI(_ values: [AnyStruct])` handles the common ABI types directly:

| Solidity type | Cadence value | Notes |
|---------------|---------------|-------|
| `address` | `EVM.EVMAddress` | Build with `EVM.addressFromString` |
| `uint8 ... uint256` | `UInt8`, `UInt16`, `UInt32`, `UInt64`, `UInt128`, `UInt256` | Match the width exactly |
| `int8 ... int256` | `Int8`, `Int16`, `Int32`, `Int64`, `Int128`, `Int256` | Match the width exactly |
| `bool` | `Bool` | |
| `string` | `String` | UTF-8 encoded |
| `bytes` | `[UInt8]` | Dynamic byte array |
| `bytesN` | `[UInt8]` with fixed length | Length must match `N` |
| arrays `T[]` | `[T]` | Each `T` must itself be an encodable type |

### Things `encodeABI` does NOT handle out of the box

- **Structs (tuples)** — Cadence struct types are not auto-mapped to Solidity tuples. Encode each field separately and concatenate, or assemble the head/tail layout manually.
- **Dynamic arrays of structs** — same restriction; encode by hand or pre-pack off-chain and pass `bytes`.
- **Nested dynamic types** beyond one level — verify on testnet before relying on it.

When in doubt, round-trip the encoding through `EVM.dryCall` against a reference contract before shipping it in a state-mutating tx.

### Encoding with `encodeABIWithSignature`

`EVM.encodeABIWithSignature` derives the selector AND appends the encoded args:

```cadence
let calldata = EVM.encodeABIWithSignature("transfer(address,uint256)", [recipient, amount])
// Equivalent to: hardcoded4ByteSelector.concat(EVM.encodeABI([recipient, amount]))
```

Use this when the call site is not hot — it does keccak each invocation. For tight loops or batched calls, hardcode the selector.

### Decoding return data

`EVM.decodeABI(types:data:)` returns `[AnyStruct]` parallel to the `types` array:

```cadence
let decoded = EVM.decodeABI(
    types: [Type<UInt256>(), Type<Bool>()],
    data: result.data
)
let amount = decoded[0] as! UInt256
let ok = decoded[1] as! Bool
```

If the function returns nothing (Solidity `function f() external`), skip decoding — `result.data` will be empty.

---

## Gas limits

`gasLimit` in `coa.call` / `EVM.dryCall` / `coa.dryCall` is **EVM gas**, not Cadence CU. Excess gas is refunded; under-provisioning produces `EVM.Status.failed`.

Conservative defaults [UNVERIFIED: actual costs vary by contract; verify per call with `coa.dryCall`]:

| Operation | gasLimit |
|-----------|----------|
| Simple view (`balanceOf`, `allowance`, etc.) | `50_000` |
| Large array read (N ≤ 1024 entries) | `3_000_000` |
| ERC-20 `transfer` / `approve` | `100_000` |
| ERC-20 `transferFrom` | `120_000` |
| Uniswap-v2-style swap | `250_000` |
| Uniswap-v3-style single-pool swap | `200_000` |
| Contract deploy (medium) | `2_000_000` |

When in doubt, run `coa.dryCall` once with a high ceiling (e.g. `5_000_000`), read `result.gasUsed`, and pad ~20%.

---

## EVM.Status — what each value means

| Status | Meaning | Typical cause |
|--------|---------|---------------|
| `successful` | EVM executed and committed (or simulated cleanly for dry calls) | Normal happy path |
| `failed` | EVM executed and reverted; state rolled back | `require`/`revert` in Solidity, OOG, bad math |
| `invalid` | The call never executed — input was malformed | Bad ABI encoding, gas below intrinsic 21k, unparseable calldata |
| `unknown` | Result not yet resolved | Should not surface in user code — treat as failure |

For `failed`, inspect `errorMessage` (the Solidity revert string when one was emitted) and `errorCode` (numeric VM error). For `invalid`, the error is on the Cadence side of the encoding — fix your calldata, not the contract.

```cadence
let result = coa.call(...)
if result.status == EVM.Status.successful { /* happy path */ }
else if result.status == EVM.Status.failed { panic("EVM revert: ".concat(result.errorMessage)) }
else { panic("Invalid call [".concat(result.errorCode.toString()).concat("]")) }
```

---

## When to use which

| Concern | `EVM.dryCall` | `coa.dryCall` | `coa.call` |
|---------|---------------|---------------|------------|
| Needs a COA in storage | No | Yes (plain `&` ref) | Yes (`auth(EVM.Call)` ref) |
| Commits EVM state | No | No | Yes |
| `msg.sender` inside the EVM call | The `from` you pass | The COA's EVM address | The COA's EVM address |
| Usable in a script | Yes | No (storage borrow) | No (storage borrow) |
| Usable in a transaction | Yes | Yes | Yes |
| Pays EVM gas (FLOW) | No | No | Yes (consumed `gasUsed * gasPrice`) |
| Cadence CU overhead | Low [UNVERIFIED: ~50–100 CU; see T12] | Similar to `coa.call` minus commit [UNVERIFIED: see T12] | Higher; scales with EVM gas used [UNVERIFIED: ~200–500 CU baseline plus gas-derived CU; see T12] |
| Use case | Read-only view, public state | Gas estimation, sender-gated preflight | Any state change, value transfer |

For CU planning and how EVM gas converts into the per-transaction CU ceiling, see [cu-ceiling.md](cu-ceiling.md).

---

## Worked example: bridge-and-call (ERC-20 approve + stake)

Two state-mutating calls in one atomic transaction. If either reverts, the whole tx reverts so long as you `panic` on a non-success status.

```cadence
import "EVM"

transaction(stakingHex: String, tokenHex: String, amount: UInt256) {
    prepare(signer: auth(BorrowValue) &Account) {
        let coa = signer.storage.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(
            from: /storage/evm
        ) ?? panic("No COA")

        let staking = EVM.addressFromString(stakingHex)
        let token = EVM.addressFromString(tokenHex)

        // 1) approve(stakingContract, amount)
        let approveSel: [UInt8] = [0x09, 0x5e, 0xa7, 0xb3]
        let approveData = approveSel.concat(EVM.encodeABI([staking, amount]))
        let r1 = coa.call(to: token, data: approveData, gasLimit: 100_000,
                          value: EVM.Balance(attoflow: 0))
        assert(r1.status == EVM.Status.successful, message: "approve failed: ".concat(r1.errorMessage))

        // 2) stake(amount)
        // selector = keccak256("stake(uint256)")[:4] = 0xa694fc3a
        let stakeSel: [UInt8] = [0xa6, 0x94, 0xfc, 0x3a]
        let stakeData = stakeSel.concat(EVM.encodeABI([amount]))
        let r2 = coa.call(to: staking, data: stakeData, gasLimit: 300_000,
                          value: EVM.Balance(attoflow: 0))
        assert(r2.status == EVM.Status.successful, message: "stake failed: ".concat(r2.errorMessage))
    }
}
```

Because every call status is asserted, a revert on either step propagates as a Cadence panic and the whole transaction (including any Cadence-side state changes the prepare block made) is rolled back.

---

## Common pitfalls

1. **Not checking `result.status`.** Cadence does NOT revert on EVM failure. Every `coa.call` / `EVM.dryCall` / `coa.dryCall` must be followed by an explicit `result.status == EVM.Status.successful` check, otherwise EVM failures silently no-op while the Cadence side commits.
2. **Wrong entitlement on the COA borrow.** `coa.call` requires `auth(EVM.Call) &EVM.CadenceOwnedAccount`. A plain `&` reference cannot call. `coa.dryCall` works with a plain reference. For value transfer out of the COA you additionally need `auth(EVM.Withdraw)`.
3. **Forgetting the selector.** `EVM.encodeABI([...])` encodes ARGUMENTS only — it does not prepend the 4-byte selector. Either prepend manually or use `EVM.encodeABIWithSignature(sig, [...])`.
4. **Wrong integer width.** Solidity `uint256` must be `UInt256` in Cadence, `uint8` must be `UInt8`, etc. Passing `UInt64` where the contract expects `uint256` produces an `invalid` status or an off-by-zero-padding bug.
5. **Treating `gasLimit` as a CU budget.** `gasLimit` is EVM gas. Cadence CU is a separate budget; both must fit. See [cu-ceiling.md](cu-ceiling.md).
6. **Using `EVM.dryCall` for `msg.sender`-gated reads.** `EVM.dryCall` lets you pass any `from`. If the contract gates on `msg.sender` (e.g. `nonces[msg.sender]`), the answer is meaningless. Use `coa.dryCall` instead.
7. **Encoding structs or arrays-of-structs with `encodeABI`.** Plain `encodeABI` does not round-trip arbitrary tuples. For complex types, encode off-chain and pass the resulting `bytes`, or assemble the head/tail layout by hand.
8. **Reading `EVM.Status.invalid` as a contract rejection.** `invalid` means the input never reached the contract — your encoding is wrong. Fix the calldata, not the contract.
9. **Hardcoding the storage path.** `/storage/evm` is a convention, not a protocol rule. Verify the path your project uses before borrowing.

---

## See also

- [coa-lifecycle.md](coa-lifecycle.md) — creating, borrowing, and destroying COAs.
- [cu-ceiling.md](cu-ceiling.md) — how EVM gas interacts with the Cadence CU budget.
- `cadence-lang/references/accounts.md` — entitlement model for `auth(...) &Account`.
- `flow-react-sdk/references/cross-vm.md` — equivalent flows from a React frontend.
- `cadence-audit/references/audit-checklist.md` — audit rule: every EVM call must check `result.status`.
