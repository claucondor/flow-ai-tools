# Reading ERC20 Storage from Cadence

Reading ERC20 storage from Cadence is the cheapest, most common cross-VM operation: a Cadence script encodes a 4-byte function selector plus ABI-encoded arguments, hands them to `EVM.dryCall`, and decodes the returned `[UInt8]` back into a Cadence value. Nothing crosses the VM boundary except calldata in and return-data out; no state changes, no COA, no signature, no transaction fee. This is the canonical pattern any wallet, DeFi UI, or Cadence-side analytics script uses to display EVM token balances, allowances, and metadata. Compared to mutating calls (`coa.call`), `dryCall` is cheap (~50–100 CU for a simple view) because it neither persists EVM state nor requires the caller to own a `CadenceOwnedAccount` resource.

See [evm-call.md](evm-call.md) for the full mechanics of `EVM.dryCall`, calldata layout, and revert decoding. See [coa-lifecycle.md](coa-lifecycle.md) for why reads do NOT need a COA. See [cu-ceiling.md](cu-ceiling.md) for per-call CU budgets when batching many reads.

---

## API Surface (recap)

```cadence
// Contract-level dryCall — no COA required, accepts any EVMAddress as `from`
access(all) fun dryCall(
    from: EVMAddress,
    to: EVMAddress,
    data: [UInt8],
    gasLimit: UInt64,
    value: EVM.Balance
): EVM.Result

// ABI helpers
EVM.encodeABI(_ values: [AnyStruct]): [UInt8]
EVM.encodeABIWithSignature(_ signature: String, _ values: [AnyStruct]): [UInt8]
EVM.decodeABI(types: [Type], data: [UInt8]): [AnyStruct]
EVM.decodeABIWithSignature(_ signature: String, types: [Type], data: [UInt8]): [AnyStruct]

// Address construction
EVM.addressFromString(_ asHex: String): EVMAddress   // accepts "0x..." or "..."
EVM.EVMAddress(bytes: [UInt8; 20])                   // raw 20-byte form
```

`EVM.Result.status` is `EVM.Status.successful` on success. On failure (revert, OOG, non-contract address), it is `EVM.Status.failed` and `data` may be empty or contain a Solidity revert reason.

---

## Canonical ERC20 Selectors

These never change — they are derived from the function signature via `keccak256(signature)[:4]`. Cadence does not need to compute them; you can pass them as literal `[UInt8]` or rely on `encodeABIWithSignature` to prepend them for you.

| Function | Signature | Selector |
|---|---|---|
| `balanceOf(address)` | `balanceOf(address)` | `0x70a08231` |
| `allowance(address,address)` | `allowance(address,address)` | `0xdd62ed3e` |
| `decimals()` | `decimals()` | `0x313ce567` |
| `symbol()` | `symbol()` | `0x95d89b41` |
| `name()` | `name()` | `0x06fdde03` |
| `totalSupply()` | `totalSupply()` | `0x18160ddd` |

All examples below prefer `encodeABIWithSignature` (slightly more verbose but self-documenting) over manual selector concatenation.

---

## 1. `balanceOf(address)` — Read Raw Balance

```cadence
import "EVM"

/// Returns the raw ERC20 balance (UInt256, no decimal adjustment) for `holder`
/// on the token at `tokenHex`. Returns nil if the EVM call reverts.
access(all) fun main(tokenHex: String, holderHex: String): UInt256? {
    let token = EVM.addressFromString(tokenHex)
    let holder = EVM.addressFromString(holderHex)

    let calldata = EVM.encodeABIWithSignature("balanceOf(address)", [holder])

    let result = EVM.dryCall(
        from: EVM.addressFromString("0x0000000000000000000000000000000000000000"),
        to: token,
        data: calldata,
        gasLimit: 100_000,
        value: EVM.Balance(attoflow: 0)
    )

    if result.status != EVM.Status.successful {
        return nil
    }

    let decoded = EVM.decodeABI(types: [Type<UInt256>()], data: result.data)
    return decoded[0] as! UInt256
}
```

### Converting raw balance to a display value

`UInt256` is the on-chain unit (wei-like). For UI display, divide by `10^decimals`. Cadence has no native `UInt256 → UFix64` cast; you must:

```cadence
// raw is the UInt256 from balanceOf, decimals is from decimals() (typically 18 for ERC20)
// UFix64 has 8 fractional digits — for >8 decimal tokens, divide first to avoid overflow

let scale = UInt256(10) ** UInt256(decimals - 8)  // safe only if decimals >= 8
let scaled = raw / scale                          // now fits in UFix64 up to ~1.8e11
let displayUFix = UFix64(scaled) / 100_000_000.0  // shift remaining 8 decimals
```

✅ Always check `decimals >= 8` before this pattern. For decimals < 8 (rare), shift the other way.
❌ Never do `UFix64(raw) / UFix64(10**18)` — `UInt256(10**18)` overflows `UFix64`'s range above ~1.8e11.

---

## 2. `allowance(owner, spender)` — Read Approval Amount

```cadence
import "EVM"

access(all) fun main(
    tokenHex: String,
    ownerHex: String,
    spenderHex: String
): UInt256? {
    let token = EVM.addressFromString(tokenHex)
    let owner = EVM.addressFromString(ownerHex)
    let spender = EVM.addressFromString(spenderHex)

    let calldata = EVM.encodeABIWithSignature("allowance(address,address)", [owner, spender])

    let result = EVM.dryCall(
        from: EVM.addressFromString("0x0000000000000000000000000000000000000000"),
        to: token,
        data: calldata,
        gasLimit: 100_000,
        value: EVM.Balance(attoflow: 0)
    )

    if result.status != EVM.Status.successful {
        return nil
    }

    let decoded = EVM.decodeABI(types: [Type<UInt256>()], data: result.data)
    return decoded[0] as! UInt256
}
```

---

## 3. `decimals()`, `totalSupply()` — No-Arg, Scalar Return

```cadence
import "EVM"

access(all) fun main(tokenHex: String): UInt8? {
    let result = EVM.dryCall(
        from: EVM.addressFromString("0x0000000000000000000000000000000000000000"),
        to: EVM.addressFromString(tokenHex),
        data: EVM.encodeABIWithSignature("decimals()", []),
        gasLimit: 50_000,
        value: EVM.Balance(attoflow: 0)
    )
    if result.status != EVM.Status.successful { return nil }
    // ERC20 returns `uint8` ABI-encoded as a right-aligned 32-byte word.
    return EVM.decodeABI(types: [Type<UInt8>()], data: result.data)[0] as! UInt8
}
```

`totalSupply()` follows the same shape — swap the signature to `"totalSupply()"` and the decoded type to `Type<UInt256>()`.

Note: some legacy tokens (MakerDAO MKR is the classic case) return `bytes32` instead of `uint8` for `decimals()`. If `dryCall` succeeds but decoding fails, fall back to `Type<UInt256>()` and cast manually.

---

## 4. `symbol()`, `name()` — Dynamic-Length String

Strings in Solidity ABI are encoded as `(offset, length, data)`. `EVM.decodeABI` handles the offset/length unpacking automatically when you request `Type<String>()`.

```cadence
import "EVM"

access(all) fun main(tokenHex: String): String? {
    let result = EVM.dryCall(
        from: EVM.addressFromString("0x0000000000000000000000000000000000000000"),
        to: EVM.addressFromString(tokenHex),
        data: EVM.encodeABIWithSignature("symbol()", []),
        gasLimit: 100_000,
        value: EVM.Balance(attoflow: 0)
    )
    if result.status != EVM.Status.successful { return nil }
    return EVM.decodeABI(types: [Type<String>()], data: result.data)[0] as! String
}
```

`name()` is identical — swap the signature to `"name()"`.

Gotcha: some early tokens (the original MKR, SAI) encode `symbol()`/`name()` as `bytes32` instead of `string`. Detect by inspecting `result.data.length`:
- 32 bytes → `bytes32` (right-padded ASCII; trim trailing `0x00`)
- ≥ 96 bytes → standard `string` encoding (32-byte offset + 32-byte length + padded data)

✅ Robust reader pattern:
```cadence
if result.data.length == 32 {
    // bytes32 fallback — decode as UInt256, convert bytes to ASCII string manually
} else {
    return EVM.decodeABI(types: [Type<String>()], data: result.data)[0] as! String
}
```

---

## 5. Batching: One Script, Multiple Reads

Each `dryCall` costs `~50–100 CU` `[UNVERIFIED: T12 must measure exact per-call overhead]` plus return-data decoding. The 9999 CU script ceiling allows roughly 50–100 reads per script — far more than RPC-side batching on most chains.

```cadence
import "EVM"

access(all) struct TokenInfo {
    access(all) let symbol: String?
    access(all) let decimals: UInt8?
    access(all) let totalSupply: UInt256?
    access(all) let userBalance: UInt256?

    init(symbol: String?, decimals: UInt8?, totalSupply: UInt256?, userBalance: UInt256?) {
        self.symbol = symbol
        self.decimals = decimals
        self.totalSupply = totalSupply
        self.userBalance = userBalance
    }
}

access(all) fun main(tokenHex: String, holderHex: String): TokenInfo {
    let token = EVM.addressFromString(tokenHex)
    let holder = EVM.addressFromString(holderHex)
    let zero = EVM.addressFromString("0x0000000000000000000000000000000000000000")
    let zeroBal = EVM.Balance(attoflow: 0)

    // symbol()
    var symbol: String? = nil
    let symRes = EVM.dryCall(from: zero, to: token,
        data: EVM.encodeABIWithSignature("symbol()", []),
        gasLimit: 100_000, value: zeroBal)
    if symRes.status == EVM.Status.successful && symRes.data.length >= 96 {
        symbol = EVM.decodeABI(types: [Type<String>()], data: symRes.data)[0] as? String
    }

    // decimals()
    var decimals: UInt8? = nil
    let decRes = EVM.dryCall(from: zero, to: token,
        data: EVM.encodeABIWithSignature("decimals()", []),
        gasLimit: 50_000, value: zeroBal)
    if decRes.status == EVM.Status.successful {
        decimals = EVM.decodeABI(types: [Type<UInt8>()], data: decRes.data)[0] as? UInt8
    }

    // totalSupply()
    var totalSupply: UInt256? = nil
    let tsRes = EVM.dryCall(from: zero, to: token,
        data: EVM.encodeABIWithSignature("totalSupply()", []),
        gasLimit: 50_000, value: zeroBal)
    if tsRes.status == EVM.Status.successful {
        totalSupply = EVM.decodeABI(types: [Type<UInt256>()], data: tsRes.data)[0] as? UInt256
    }

    // balanceOf(holder)
    var userBalance: UInt256? = nil
    let balRes = EVM.dryCall(from: zero, to: token,
        data: EVM.encodeABIWithSignature("balanceOf(address)", [holder]),
        gasLimit: 100_000, value: zeroBal)
    if balRes.status == EVM.Status.successful {
        userBalance = EVM.decodeABI(types: [Type<UInt256>()], data: balRes.data)[0] as? UInt256
    }

    return TokenInfo(
        symbol: symbol,
        decimals: decimals,
        totalSupply: totalSupply,
        userBalance: userBalance
    )
}
```

Total cost: `[UNVERIFIED: ~200–500 CU for 4 reads, T12 must measure]`. Well under the 9999 CU script ceiling.

---

## Edge Cases

### Zero address holder
`balanceOf(0x0)` on a standard ERC20 returns the amount of tokens that have been irrecoverably burned via transfer-to-zero. The call succeeds; it does NOT revert. Treat the result like any other balance.

### Non-standard ERC20s (USDT, BNB)
`USDT.transfer()` and `USDT.approve()` return nothing instead of `bool` — this matters for **writes** (`coa.call`) but is irrelevant for the read methods covered here. `balanceOf`, `allowance`, `decimals`, `symbol`, `name`, and `totalSupply` all follow the standard on USDT, USDC, and every major token.

### Proxies (EIP-1967, transparent, UUPS)
`EVM.dryCall` transparently follows the proxy's `delegatecall` chain. No special handling is needed in Cadence — pass the proxy address as `to` and decode the result normally. The proxy contract's `fallback()` forwards the call to the implementation, which executes against the proxy's storage. From Cadence's perspective, the proxy *is* the token.

### Contract does not implement ERC20 (or address has no contract)
`dryCall` returns `Status.failed` with empty `data`. Return `nil` from the script — do not `panic`.

```cadence
✅ if result.status != EVM.Status.successful { return nil }
❌ assert(result.status == EVM.Status.successful, message: "call failed")  // panics, kills the whole script
```

### Reentrancy and stale reads
Scripts execute against a fixed block snapshot. Two reads in the same script see the same state. There is no reentrancy risk on read-only scripts.

---

## CU Cost Reference

| Operation | Approximate CU |
|---|---|
| `EVM.encodeABIWithSignature(...)` with 0–2 args | `[UNVERIFIED: ~5–20 CU, T12 must measure]` |
| `EVM.dryCall` simple `uint256` return | `[UNVERIFIED: ~50–100 CU, T12 must measure]` |
| `EVM.dryCall` dynamic `string` return | `[UNVERIFIED: ~80–150 CU, T12 must measure]` |
| `EVM.decodeABI` to `UInt256` | `[UNVERIFIED: ~10–30 CU, T12 must measure]` |
| `EVM.decodeABI` to `String` (short) | `[UNVERIFIED: ~30–80 CU, T12 must measure]` |

See [cu-ceiling.md](cu-ceiling.md) for the methodology and ceiling implications when batching.

The EVM `gasLimit` parameter on a `dryCall` is consumed but not charged in FLOW (no fee is paid for view operations). It still must be high enough to cover the EVM-side execution — `100_000` is generous for any standard ERC20 read.

---

## Common Pitfalls

### Confusing `EVMAddress` with Cadence `Address`
- Cadence `Address` is the 8-byte Flow account address (`0x1cf0e2f2f715450`).
- `EVM.EVMAddress` is the 20-byte EVM address (`0xA0Cf798816D4b9b9866b5330EEa46a18382f251e`).
- Do not pass a `Address` where an `EVMAddress` is expected. They are entirely different types and the compiler will reject the substitution — but if you carry the wrong format as a `String` argument, `EVM.addressFromString` will silently accept malformed input. Validate hex length is 40 characters (or 42 with `0x` prefix).

### Using `coa.call` for reads
`coa.call` requires a `CadenceOwnedAccount` resource, mutates EVM state if the function is non-view, and charges full EVM gas in FLOW. For pure reads, `EVM.dryCall` is strictly better: no COA needed, no fee, ~3–5× cheaper in Cadence CU.
- ✅ Read: `EVM.dryCall`
- ❌ Read: `coa.call` (wastes CU, requires a COA, and you cannot run it from a script — only a transaction)

### Assuming all ERC20s strictly implement the standard
The read methods are well-behaved on every major token, but tail-end edge cases exist:
- MKR / SAI: `symbol()` / `name()` return `bytes32` instead of `string`. Inspect `result.data.length` and branch.
- USDT et al: write-side `transfer` returns nothing instead of `bool`. Irrelevant for reads, but relevant if you later want to `coa.call` `transfer`.
- Tokens with no `name()` or `symbol()` (rare, mostly defunct): `dryCall` returns `Status.failed`. Return `nil`.

### Unit confusion: raw `UInt256` vs decimal-adjusted `UFix64`
The `UInt256` from `balanceOf` is in the token's smallest unit (typically `10^18` per whole token). Always pair with a `decimals()` lookup before displaying. Mixing raw and adjusted values in math is the most common cross-VM display bug.

### Forgetting the `from` argument
The contract-level `EVM.dryCall` takes `from`, `to`, `data`, `gasLimit`, `value`. The COA-level `dryCall` (called on a `&EVM.CadenceOwnedAccount` reference) omits `from` because it is implicit. From a script, you must use the contract-level version and pass `EVM.addressFromString("0x0...0")` (or any well-known sentinel) as `from`. The token contract sees this as `msg.sender`, which for `view` reads typically does not matter.

### Panicking on failure
Scripts that `panic` on `Status.failed` cannot be used safely as building blocks for UIs — a single bad token kills a batched read. Always return `nil` or an explicit error variant. Reserve `panic` for invariant violations, never for upstream failures.

---

## Related References

- [evm-call.md](evm-call.md) — Canonical mechanics of `EVM.dryCall` and `coa.call`, calldata layout, `result.status` discipline, revert reason decoding.
- [coa-lifecycle.md](coa-lifecycle.md) — ERC20 reads do NOT need a COA. This is the primary reason to prefer `dryCall` over `coa.call` for view methods.
- [cu-ceiling.md](cu-ceiling.md) — The 9999 CU shared budget. ERC20 reads are the cheapest cross-VM operation; you can batch 50–100 per script.
- [solidity-fixtures.md](solidity-fixtures.md) — Minimal ERC20 Solidity fixtures for local emulator tests of these read patterns.
