# Native FLOW Bridge — Cadence ↔ EVM

Native FLOW is the only token on Flow that crosses the Cadence/EVM boundary without an ERC20
wrapping step or a bridge contract — moving it is two primitive methods on the COA resource:
`coa.deposit(from: <-vault)` and `coa.withdraw(balance: ...)`. Both run inside a single Cadence
transaction, and the transaction itself is atomic at the Cadence layer — but the EVM side is
*not* automatically atomic with it: a reverted `coa.call` does not abort the surrounding Cadence
tx unless you explicitly check `result.status`. The conversion math is also subtle: FLOW is
`UFix64` (8 decimal digits) in Cadence and `UInt` attoflow (18 decimal digits) on EVM, so a
factor of `10^10` separates the two.

> **Scope.** This reference covers **native FLOW only**. ERC20 tokens on Flow EVM are bridged
> through a separate contract system (`onflow/flow-evm-bridge`) that wraps Cadence FTs into
> ERC20s and vice versa — that surface is covered in the cross-VM tokens reference, not here.
> See also [`coa-lifecycle.md`](coa-lifecycle.md) for COA creation and entitlements, and
> [`evm-call.md`](evm-call.md) for the EVM call result handling that underpins true round-trip
> atomicity.

---

## API Surface

### Cadence → EVM: `coa.deposit`

```cadence
access(all)
fun deposit(from: @FlowToken.Vault)
```

Takes a `@FlowToken.Vault` resource, moves its full balance into the COA's EVM balance, and
destroys the input vault. **No entitlement required** — anyone with a reference to the COA can
deposit into it. The increase is reflected immediately in `coa.balance()`.

### EVM → Cadence: `coa.withdraw`

```cadence
access(EVM.Owner | EVM.Withdraw)
fun withdraw(balance: EVM.Balance): @FlowToken.Vault
```

Takes an `EVM.Balance`, decrements the COA's EVM-side balance by that amount, and returns a
freshly-minted `@FlowToken.Vault` containing the withdrawn FLOW. **Requires either `EVM.Owner`
or `EVM.Withdraw` entitlement** on the COA reference — borrowing with `&EVM.CadenceOwnedAccount`
alone is not sufficient.

### Read EVM balance: `coa.balance`

```cadence
access(all)
view fun balance(): EVM.Balance
```

Returns the COA's EVM-side FLOW balance. Use `.inFLOW()` to project to `UFix64`, or read the
`.attoflow` field for the full 18-digit precision.

### The `EVM.Balance` struct

```cadence
access(all) struct Balance {
    access(all) var attoflow: UInt          // 18-decimal "wei-style" FLOW
    init(attoflow: UInt)                    // construct from atto units
    access(all) fun setFLOW(flow: UFix64)   // set from 8-decimal Cadence FLOW
    access(all) view fun inFLOW(): UFix64   // project to 8-decimal Cadence FLOW (lossy)
    access(all) view fun inAttoFLOW(): UInt // redundant alias for the .attoflow field
}
```

Verified against emulator EVM contract `A.f8d6e0586b0a20c7.EVM` on CLI v2.17.1 (Crescendo era).

---

## Decimal Reconciliation: UFix64 ↔ attoflow

FLOW is stored two different ways on the two sides of the VM boundary:

| Side    | Type    | Decimal digits | Smallest unit               |
|---------|---------|----------------|-----------------------------|
| Cadence | `UFix64`| 8              | `0.00000001 FLOW` (10 nF)   |
| EVM     | `UInt`  | 18             | `1 attoflow` (10^-18 FLOW)  |

The two formats differ by exactly `10^10`. Conversions:

```
attoflow  =  UInt(flow_ufix64 * 10.0^8) * 10^10
flow_ufix =  UFix64(attoflow / 10^10) / 10^8       // lossy when attoflow % 10^10 != 0
```

Three idioms for constructing a balance:

```cadence
// 1. From attoflow directly (full precision, EVM-native amounts)
let b1 = EVM.Balance(attoflow: UInt(1_500_000_000_000_000_000))   // 1.5 FLOW

// 2. From UFix64 via setFLOW (recommended when starting from Cadence-side amount)
let b2 = EVM.Balance(attoflow: 0)
b2.setFLOW(flow: 1.5)                                              // 1.5 FLOW

// 3. From a coa.balance() readout
let coaBal = coa.balance()                                         // EVM.Balance
let inUFix = coaBal.inFLOW()                                       // possibly lossy
```

> ❌ **Anti-pattern.** Do NOT compute attoflow by hand with `UInt(x) * 1_000_000_000_000_000_000`
> if your source is a `UFix64` — you will lose the fractional part of the UFix64. Use
> `setFLOW(flow:)`.
>
> ❌ **Anti-pattern.** Do NOT round-trip via `inFLOW()` when the EVM-side amount has more than
> 8 decimal digits of precision (e.g. swap output dust). You will silently truncate sub-10nF
> remainders, which is the same trap WFLOW unwrapping has.

---

## Atomicity Guarantees — The Important Part

The whole Cadence transaction (Cadence statements **and** every embedded EVM call) is atomic
at the **Cadence** layer: if the Cadence tx aborts, every state change reverts on both sides.
But the *converse* is not true: an EVM call that reverts does **not** abort the Cadence tx.

| Operation                          | EVM revert ⇒ Cadence tx aborts? |
|------------------------------------|---------------------------------|
| `coa.deposit(from: <-vault)`       | N/A — never reverts on EVM side beyond the deposit credit itself |
| `coa.withdraw(balance: ...)`       | Panics in Cadence if balance insufficient — Cadence-layer abort |
| `coa.call(to:..., data:..., ...)`  | **No** — must check `result.status` and `panic` manually |

✅ **Atomic by construction**: `coa.deposit` and `coa.withdraw` cannot leave inconsistent state
between Cadence and EVM. They are direct balance moves at the FVM layer, not EVM transactions.

❌ **Not atomic without a status check**: any `coa.call` that touches the deposited FLOW. If
the EVM contract reverts, the deposited FLOW *remains in the COA* and the Cadence tx commits
anyway. Funds aren't lost, but they're stuck in the COA unless you have a withdraw path.

Always wrap deposit + call + withdraw in a tx that panics on EVM failure:

```cadence
let result = coa.call(to: target, data: calldata, gasLimit: 200_000,
                      value: EVM.Balance(attoflow: 0))
if result.status != EVM.Status.successful {
    panic("EVM call failed (errorCode=".concat(result.errorCode.toString()).concat(")"))
}
```

See [`evm-call.md`](evm-call.md) for the full `EVM.Status` taxonomy and gas-exhaustion handling.

---

## Cost / Gas Accounting

Native FLOW bridging crosses two distinct accounting systems. Both apply to the same tx.

### Cadence side: Computation Units (CU)

| Operation                          | Cadence CU (empirical, CLI v2.17.1) |
|------------------------------------|-------------------------------------|
| `coa.deposit(from: <-vault)`       | ≈ 8.5 CU per call (+ 6 CU intercept) |
| `coa.withdraw(balance: ...)`       | ≈ 9.2 CU per call (+ 7 CU intercept) |
| `coa.balance()`                    | ≈ 1.5 CU per call (+ 5 CU intercept) |
| `EVM.Balance(...)` / `setFLOW`     | < 1 CU |

The Flow CLI's default `--compute-limit` is 9,999 CU. The protocol-level cap is set by network parameters and is materially higher (the emulator accepts ≥ 1,000,000). For batched bridge operations exceeding 9,999 CU, pass an explicit `--compute-limit` and ensure the access node accepts it.

### EVM side: gas

`coa.deposit` / `coa.withdraw` do **not** execute EVM contract code — they are balance writes
at the FVM layer — so they do not consume the COA's `gasLimit`. But once FLOW is in the COA,
a subsequent `coa.call(..., value: EVM.Balance(attoflow: N))` *does* spend EVM gas like a
normal EVM transaction (21000 base + intrinsic + execution cost). Empirically, `coa.call` with
`value > 0` does **not** add the ~9,000 gas Ethereum surcharge for non-zero CALL value — the gas
used is the base 21,000 + intrinsic + execution, regardless of whether value is transferred.

### Inclusion fee

Standard Flow transaction inclusion + execution fee applies on top of both. The payer of the
Cadence transaction is charged in FLOW from their account vault (not from the COA).

---

## Failure Modes

### 1. EVM call reverts after Cadence-credit moved (the trapped-funds scenario)

Without a `result.status` check, this sequence leaves the deposited FLOW stranded in the COA:

```cadence
// ❌ DON'T — partial commit on EVM failure
transaction(amount: UFix64) {
    prepare(signer: auth(BorrowValue) &Account) {
        let coa = signer.storage.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(
            from: /storage/evm) ?? panic("no COA")
        let vRef = signer.storage.borrow<auth(FungibleToken.Withdraw) &FlowToken.Vault>(
            from: /storage/flowTokenVault) ?? panic("no vault")
        let v <- vRef.withdraw(amount: amount)
        coa.deposit(from: <-v)                                  // FLOW now in COA
        let _ = coa.call(to: target, data: cd, gasLimit: 200_000,
                          value: EVM.Balance(attoflow: 0))      // EVM may revert
        // Cadence proceeds. FLOW sits in COA. If signer lacks withdraw entitlement, trapped.
    }
}
```

The fix is twofold: borrow the COA with the additional `EVM.Withdraw` entitlement so a recovery
path exists, AND panic on `result.status != successful`.

### 2. Insufficient COA balance on withdraw

`coa.withdraw(balance:)` panics inside Cadence if the requested amount exceeds the COA's EVM
balance. **Pre-check** with `coa.balance()`:

```cadence
let have = coa.balance().attoflow
let want: UInt = ...
assert(have >= want, message: "COA balance too low: have=".concat(have.toString())
       .concat(" want=").concat(want.toString()))
```

### 3. Sub-10nF dust loss on withdraw

If `coa.balance().attoflow` is not divisible by `10^10`, the residual atto-FLOW under 10nF
is **not withdrawable** through the `FlowToken.Vault` round trip (the Cadence vault simply
cannot represent it). The dust stays in the COA. Use `inFLOW()` to discover the safely-withdrawable
amount.

### 4. Insufficient signer-vault balance on deposit

`vault.withdraw(amount:)` panics in Cadence. This is a vault-level failure, not bridge-specific.

### 5. Bridging to a non-Flow chain by mistake

A COA's EVM address has the prefix `0x000000000000000000000002` (12 leading bytes). This prefix
is protocol-defined by the FVM `AllocateCOAAddress` function and is **identical on emulator,
testnet, and mainnet**. Sending FLOW to that address *from another L1 or L2* results in permanent
loss — COA addresses only exist on Flow.

---

## Worked Example: Full Round-Trip

A complete transaction that:

1. Withdraws 1.5 FLOW from the signer's vault.
2. Deposits it into the signer's COA (Cadence → EVM).
3. Calls an EVM contract (e.g. a Uniswap swap router).
4. Checks `result.status` and panics on failure.
5. Withdraws the remaining COA balance back to a Cadence vault.
6. Deposits that vault back into the signer's FlowToken vault.

```cadence
import "EVM"
import "FungibleToken"
import "FlowToken"

transaction(swapAmount: UFix64, target: String, calldata: [UInt8]) {

    let coa: auth(EVM.Call, EVM.Withdraw) &EVM.CadenceOwnedAccount
    let preBalanceAtto: UInt
    let receiver: &{FungibleToken.Receiver}

    prepare(signer: auth(BorrowValue) &Account) {

        // 1. Borrow COA with BOTH Call (for swap) and Withdraw (for recovery) entitlements.
        self.coa = signer.storage.borrow<auth(EVM.Call, EVM.Withdraw) &EVM.CadenceOwnedAccount>(
            from: /storage/evm
        ) ?? panic("Signer has no COA at /storage/evm")

        // 2. Pull FLOW out of the signer's vault.
        let vaultRef = signer.storage.borrow<auth(FungibleToken.Withdraw) &FlowToken.Vault>(
            from: /storage/flowTokenVault
        ) ?? panic("No FlowToken vault")
        let toBridge <- vaultRef.withdraw(amount: swapAmount)

        // 3. Snapshot pre-deposit COA balance so we can compute swap delta later.
        self.preBalanceAtto = self.coa.balance().attoflow

        // 4. Bridge Cadence → EVM (atomic by construction).
        self.coa.deposit(from: <-toBridge)

        // 5. Build value for the EVM call. Here we forward the full swapAmount as msg.value.
        let value = EVM.Balance(attoflow: 0)
        value.setFLOW(flow: swapAmount)

        // 6. Make the EVM call.
        let result = self.coa.call(
            to: EVM.addressFromString(target),
            data: calldata,
            gasLimit: 500_000,
            value: value
        )

        // 7. CRITICAL: panic on EVM revert so the deposit also reverts.
        if result.status != EVM.Status.successful {
            panic("EVM swap failed: errorCode=".concat(result.errorCode.toString())
                  .concat(" data=").concat(String.encodeHex(result.data)))
        }

        // 8. Withdraw everything remaining in the COA back to Cadence.
        //    (In a real swap you'd compute the exact output; this drains the COA for clarity.)
        let postAtto = self.coa.balance().attoflow
        let withdrawBal = EVM.Balance(attoflow: postAtto)
        let recovered <- self.coa.withdraw(balance: withdrawBal)  // returns @FlowToken.Vault directly

        // 9. Deposit the recovered FLOW back into the signer's vault.
        self.receiver = signer.capabilities
            .borrow<&{FungibleToken.Receiver}>(/public/flowTokenReceiver)
            ?? panic("No FlowToken receiver")
        self.receiver.deposit(from: <-recovered)
    }
}
```

Notes on this pattern:

- **One entitled borrow, two operations.** A single `borrow` with `auth(EVM.Call, EVM.Withdraw)`
  unlocks both the swap and the recovery — don't borrow twice.
- **`setFLOW` for value.** Even though we just deposited from a UFix64, we build the call value
  by `setFLOW(flow: swapAmount)`, not by re-deriving atto units manually.
- **The status check is what makes this atomic.** Remove the `if result.status != successful`
  block and the transaction will commit even when the swap reverts — the FLOW will sit in the
  COA. With the check, a swap revert reverts the deposit too.
- **Withdraw is `coa.balance().attoflow`-based**, not `swapAmount`-based, because the swap may
  have changed the COA balance.

500,000 is a generous default suitable for Uniswap V2/V3-style swap routers. Calibrate down for simpler operations: 21,000 for plain value transfer, ~30,000 for small log emitters, 150–300k for typical AMM swaps. See [`evm-call.md`](evm-call.md) for the gas-cost table.

---

## Read-Only: Querying COA Balance in a Script

```cadence
import "EVM"

access(all)
fun main(address: Address): {String: UFix64} {
    let acct = getAccount(address)
    let coa = acct.capabilities.borrow<&EVM.CadenceOwnedAccount>(/public/evm)
        ?? panic("No public COA capability")
    let bal = coa.balance()
    return {
        "attoflow": UFix64(bal.attoflow / 10_000_000_000) / 100_000_000.0,  // lossy
        "inFLOW":   bal.inFLOW()
    }
}
```

For richer balance scripts (per-COA, per-EVM-address), see
`flow-cli/references/cadence-scripts.md` § "Get COA EVM Balance".

---

## Common Pitfalls

1. **No `result.status` check.** The single biggest source of stuck-in-COA funds. Every code
   path that does `coa.deposit` and then `coa.call` MUST `panic` on failure.

> See canonical treatment in [evm-call.md](evm-call.md) Pitfall 1.
> This entry is a context-specific summary; updates to the underlying behavior should land in the canonical file first.

2. **Borrowing the COA without `EVM.Withdraw` entitlement.** A tx that deposits and then needs
   to recover (e.g. EVM call returned a refund) will fail to withdraw. Always include
   `EVM.Withdraw` in the entitlement set when there's any chance the round trip needs to undo.

3. **Hand-rolling attoflow conversion.** `UInt(amount * 10^18)` from a `UFix64` doesn't compile
   the way you expect — `UFix64` can't represent `10^18`. Use `EVM.Balance(attoflow: 0)` then
   `setFLOW(flow: amount)`.

4. **Round-tripping `inFLOW()` and re-depositing.** You lose sub-10nF dust on every round trip.
   For invariant-sensitive accounting, work in `attoflow: UInt` throughout the EVM-touching
   path.

5. **Confusing the native FLOW bridge with the ERC20 bridge.** This file only covers native
   FLOW. ERC20 tokens use `onflow/flow-evm-bridge` — different contract, different fees,
   different escrow semantics. Don't apply patterns from this file to USDC, USDT, or any
   wrapped ERC20.

6. **Assuming `coa.balance()` is free.** It is a Cadence-layer call against EVM state and
   consumes CU. In tight loops, cache the result.

7. **Bridging FLOW to a COA address from another chain.** COA addresses are Flow-only.
   Sending FLOW to a COA address on Ethereum / Base / etc. is irrecoverable.

8. **Unnecessarily casting the withdraw return.** The return type of `coa.withdraw` is
   `@FlowToken.Vault` (concrete, not interface), so `as! @FlowToken.Vault` downcasts are
   unnecessary. The redundant cast in older sample code is harmless but adds clutter.

---

## Cross-References

- [`coa-lifecycle.md`](coa-lifecycle.md) — COA creation, storage paths, `auth(EVM.Call)` /
  `auth(EVM.Withdraw)` entitlement borrowing patterns.
- [`evm-call.md`](evm-call.md) — `coa.call` semantics, `EVM.Status`, gas limits, the
  `result.status` check that makes a round trip atomic.
- `flow-defi/references/protocol-architecture.md` § "Cross-VM Failure Modes" — atomicity
  fundamentals and batched-call patterns.
- `flow-cli/references/cadence-scripts.md` § "Get COA EVM Balance" — read-only balance scripts
  for monitoring.
- `cadence-audit/references/forte-anti-patterns.md` — audit patterns for COA-holding handlers
  and capability-controlled bridges.
