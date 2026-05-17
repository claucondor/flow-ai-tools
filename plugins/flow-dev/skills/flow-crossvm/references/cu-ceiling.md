# The 9,999 CU Per-Transaction Ceiling for CrossVM

Every Flow transaction — pure Cadence or CrossVM — is hard-capped at **9,999 CU** of Cadence computation. CrossVM transactions do not get a second budget for the EVM side; they get a separate, smaller-grained `gasLimit` on each `coa.call` that controls EVM execution only. The two budgets are independent, and a CrossVM transaction must pass both to succeed. In practice the **Cadence CU ceiling is the one that bites first** for loops, large ABI decoding, or multi-call patterns, because EVM gas is paid out of the COA's own `EVM.Balance` and can be raised per call, while the 9,999 CU is a protocol-level cap that cannot be raised.

> **Emulator vs mainnet/testnet:** On mainnet and testnet the 9,999 CU cap is enforced at the protocol level — no flag will raise it. On the **emulator**, 9,999 is the default `--compute-limit` for `flow transactions send`, *not* a hard protocol cap; passing `--compute-limit 50000` will happily seal a transaction that consumes 27,000+ CU. This is useful for measurement and stress testing, but **always design as if 9,999 is hard**, or you will ship code that passes locally and fails on testnet.

This reference focuses on how CrossVM operations sit inside the 9,999 CU budget, how to measure them empirically, and what to do when a transaction overruns. For the general Cadence CU optimization methodology (sweep technique, dict-write costs, resource inlining), cross-link to [cu-optimization.md](../../cadence-lang/references/cu-optimization.md) — that file was added in PR #34 and may not yet be in `main` at the time you read this. For the multi-tx escrow pattern used to chunk state-mutating CrossVM work across multiple transactions, see [coa-lifecycle.md](coa-lifecycle.md) and [multi-tx-escrow.md](../../cadence-lang/references/multi-tx-escrow.md).

## Two Independent Budgets

| Budget | Where it lives | Who pays | How to raise | Failure mode |
|---|---|---|---|---|
| Cadence CU | Protocol-level, per-transaction | Tx payer, in FLOW via the standard fee model | Cannot be raised — 9,999 is the cap | Whole transaction reverts, all Cadence and EVM side effects roll back |
| EVM gas | Per `coa.call` / `EVM.dryCall` | The COA's `EVM.Balance` (FLOW held inside the COA, debited as attoflow) | Pass a larger `gasLimit` argument; ensure the COA has enough `EVM.Balance` | Only that EVM call fails (`result.status != .successful`). Cadence transaction continues unless you panic on the status. |

The asymmetry between the two failure modes is the single most common source of CrossVM bugs:

- ✅ Cadence CU overrun → atomic revert across both VMs. Painful, but loud.
- ❌ EVM gas overrun → silent EVM revert. Cadence keeps going, state changes commit. You will not notice unless you check `result.status` on every call.

See [evm-call.md](evm-call.md) for the canonical `result.status` handling pattern.

## What Counts Against the 9,999 CU Budget

Every byte the Cadence interpreter touches counts, including:

- The transaction prologue (signing checks, `prepare` storage access).
- Each `EVM.dryCall` and `coa.call` invocation overhead.
- ABI encoding of the calldata (`EVM.encodeABI`) and decoding of `EVM.Result.data` (`EVM.decodeABI`).
- Borrowing the COA reference from storage.
- Any Cadence loops, dictionary writes, or resource moves around the EVM call.

The **EVM execution itself** does not consume Cadence CU — it consumes EVM gas. But the bridge layer that marshals calldata in and result data out absolutely does, and the bigger the EVM return data the more CU you pay on the Cadence side.

## Approximate CU Costs per CrossVM Op

Measured 2026-05-17 on Flow emulator v2.17.1 using parameterised sweep transactions; see `T12-VERIFICATION.md` for raw data and fits. Numbers reported here are the per-op marginal cost plus the per-tx intercept; non-linear ops are flagged inline.

| Operation | Measured Cadence CU | EVM gas (separate) | Notes |
|---|---|---|---|
| Borrow COA + check capability | ~7 CU single borrow (slope 1.75 CU/op, intercept ~5 CU, linear, R²=1.0000) | 0 | Pure Cadence storage read (M01) |
| `EVM.dryCall` returning a single word (e.g. ERC20 `balanceOf`) | ~16 CU single call (slope 5.49 CU/op, intercept ~10 CU, linear, R²=1.0000) | consumed but not charged | View-only; gas refunded (M02) |
| `EVM.dryCall` returning a large array (e.g. `uint256[8]` from a multicall) | ~25 CU single call (slope 16.52 CU/op, intercept ~9 CU, linear, R²=1.0000) | scales with `gasLimit` | Decoding overhead ~11 CU above single-word case (M03) |
| `coa.call` state-mutating (ERC20 transfer) | **Quadratic: `CU ≈ 0.108·N² + 6.70·N + 10.95`** (R²=1.0000). N=1 → 18 CU; N=50 → 617 CU; N=100 → 1,765 CU; N=250 → 8,458 CU | charged from COA balance | See "Linear vs Quadratic Ops" below. Naive linear extrapolation underestimates by ~4× at N=200 (M04) |
| `coa.deposit` (Cadence FLOW → EVM) | **Quadratic: `CU ≈ 0.0222·N² + 8.11·N + 7.69`** (R²=1.0000). N=1 → 16 CU; N=100 → 1,041 CU; N=500 → 9,617 CU (near cliff) | minimal (native value transfer) | See [flow-bridge.md](flow-bridge.md) (M05) |
| `coa.withdraw` (EVM FLOW → Cadence) | **Quadratic: `CU ≈ 0.0222·N² + 8.87·N + 7.91`** (R²=1.0000). N=1 → 17 CU; N=100 → 1,117 CU; N=500 → 9,999 CU (cliff) | minimal | Returns a `@FlowToken.Vault`; ~10% steeper than `coa.deposit` at large N (M06) |
| `EVM.encodeABI([arg1, arg2, ...])` for small args | ~8 CU per encode (slope 1.91 CU/op, intercept ~6 CU, linear, R²=1.0000) | 0 | Per-call, before the EVM hop (M07) |
| `EVM.decodeABI(types: [...], data: result.data)` for `(uint256, uint256)` | ~6 CU per decode (slope 0.47 CU/op, intercept ~5.5 CU, linear, R²=1.0000) | 0 | Scales with field count and total bytes (M08) |

Within-network variance is 0% (deterministic): M13 ran the same transaction 10× and produced 10 identical CU readings. Cross-network variance (emulator vs testnet) may still exceed 30% on storage-heavy ops, because emulator does not perfectly match the production weighting of storage reads — always re-sweep on testnet before publishing a `MAX_SAFE_N` for users.

### Linear vs Quadratic Ops

Only **state-mutating EVM calls** scale superlinearly in N within a single transaction: `coa.call` writes, `coa.deposit`, and `coa.withdraw` all fit `a·N² + b·N + c` with R² = 1.0000, while linear fits give R² ≈ 0.95–0.97. Read-only and Cadence-side ops (`EVM.dryCall`, `EVM.encodeABI`, `EVM.decodeABI`, COA borrows) all scale **linearly** with R² > 0.999. The split is sharp: anything that does not mutate EVM state is linear; anything that does is superlinear.

The most likely cause is per-call EVM block-state journaling: each state-mutating hop within the same Cadence transaction touches and re-validates the cumulative state proof, so the marginal cost grows with the number of prior writes in the same tx. In practice this means **10 transfers cost noticeably more than 10× one transfer**, and at large N a single tx ramps from ~7 CU/op to >30 CU/op — small-N rules of thumb break down. If you loop **state-writing** EVM ops, model the cost as `a·N² + b·N`, not `b·N + c`. For batching decisions: bigger batches still amortise the per-tx prologue, but the quadratic term eventually wins, so chunk state-writing work across transactions (see "Workarounds When You Hit the Ceiling" below) rather than trying to maximise N in a single tx.

## Measurement Methodology

The authoritative cost for a transaction comes from the `FlowFees.FeesDeducted` event emitted on every sealed transaction. The `executionEffort` field on that event is the CU consumed. The general sweep methodology — increase N, find the cliff, set MAX_SAFE_N at 10% headroom — is documented in `cadence-lang/references/cu-optimization.md` (PR #34, may not be in `main` yet). For CrossVM-specific measurement:

### 1. Build a parameterised test transaction

The transaction takes an `N: Int` argument and performs `N` identical CrossVM operations. For an `EVM.dryCall` sweep:

```cadence
import "EVM"

transaction(target: EVM.EVMAddress, calldata: [UInt8], n: Int) {
    prepare(acct: auth(BorrowValue) &Account) {
        let coa = acct.storage.borrow<&EVM.CadenceOwnedAccount>(
            from: /storage/evm
        ) ?? panic("no COA")

        var i = 0
        while i < n {
            let result = EVM.dryCall(
                from: coa.address(),
                to: target,
                data: calldata,
                gasLimit: 50_000,
                value: EVM.Balance(attoflow: 0)
            )
            assert(result.status == EVM.Status.successful, message: "dryCall failed")
            i = i + 1
        }
    }
}
```

### 2. Sweep N and read fees

Run the transaction at `N = [1, 2, 4, 8, 16, 32, 64, 128]` on the emulator. After each sealed transaction, query the `FlowFees.FeesDeducted` event and record `executionEffort`. Plot `N` versus `executionEffort`:

- **y-intercept** is the per-tx fixed overhead (prologue, COA borrow, etc.).
- **slope** is the marginal CU per CrossVM op.
- The **cliff** — the smallest `N` that fails with an out-of-CU error — confirms the 9,999 CU ceiling.

### 3. Set the production cap

Take the largest passing `N`, subtract 10% headroom, and that is `MAX_SAFE_N` for that operation class. Document it next to the transaction so future edits know the budget shape. Re-run on testnet because emulator under-weights storage and may overestimate `MAX_SAFE_N`.

### 4. CrossVM-specific gotcha

The slope can be **non-linear** when calldata size varies (e.g. concatenating arrays before each call). Always sweep with **identical** calldata so the slope reflects the EVM hop cost, not the encoding cost. Then run a second sweep that varies calldata size at fixed `N=1` to separately characterise encoding cost.

### Converting fee back to CU

`FlowFees.FeesDeducted.amount` is denominated in FLOW. To convert it to CU:

```
executionFee  = amount - inclusionFee
CU            = executionEffort  // direct field on the event
```

The `executionEffort` field on `FlowFees.FeesDeducted` is the **raw CU consumed**, before the multiplier and the surge factor. Read that field directly rather than back-computing from the total fee — the back-computation is brittle when `executionEffortCost` or `surgeFactor` changes.

```javascript
// flow events get FlowFees.FeesDeducted  --start <blockHeight> --end <blockHeight>
//
// Look for the `executionEffort` field and divide by 100_000_000.0
// to get the original CU integer (the contract stores it as UFix64
// in effort units, not raw CU).
```

Verified: `cu = round(executionEffort * 1e8)`. The field is `UFix64` with 8 decimals; multiplying by 1e8 recovers the integer CU. Confirmed across every measurement (noop prologue = 4 CU, all sweep rows consistent with their regressions).

## Worked Example: Reading 8 ERC20 Balances

A treasury dashboard transaction reads balances for 8 ERC20s on behalf of a single user. Two implementations, both legal, with measurably different CU profiles:

**Implementation A — 8 separate `EVM.dryCall`s.** Measured fit (M10a): `CU ≈ 7.4·K + 9.33` (linear, R²=1.0000). At K=8: **68 CU total** (~8.5 CU per token).

**Implementation B — 1 `EVM.dryCall` to a `BalanceMulticall` Solidity view.** Measured fit (M10b): `CU ≈ 4.85·K + 15.9` (linear, R²=0.998). At K=8: **58 CU total** (~4.85 CU per token + ~16 CU intercept).

At K=8 the speedup is ~17% (68 → 58 CU), **not 3×**. The crossover only becomes meaningful around K=32, where B is ~31% cheaper, and it really pays off near the cliff: Implementation A blows the default 9,999 CU budget around K ≈ 1,300, Implementation B around K ≈ 2,000. The case for the aggregator is real but smaller than "order-of-magnitude" — it is mostly about *headroom under the cliff*, not raw CU savings at small K. Pick B when K is large or when you need the room; for K ≤ 8 the implementations are effectively interchangeable on CU.

## Workarounds When You Hit the Ceiling

### Batching reads into one EVM call

Multiple `EVM.dryCall` invocations to read related state burn CU on each Cadence-side hop. Replace them with a single EVM-side aggregator (a `Multicall`-style contract or a custom view returning a struct):

```cadence
// ❌ 4 hops, ~4 × per-call overhead
let bal = EVM.decodeABI(types: [Type<UInt256>()], data:
    EVM.dryCall(from: coa.address(), to: erc20, data: balanceOfData,
                gasLimit: 50_000, value: EVM.Balance(attoflow: 0)).data)
let allow = EVM.decodeABI(types: [Type<UInt256>()], data:
    EVM.dryCall(from: coa.address(), to: erc20, data: allowanceData,
                gasLimit: 50_000, value: EVM.Balance(attoflow: 0)).data)
// ...two more hops...

// ✅ 1 hop, returns a struct, decoded once
let packed = EVM.decodeABI(
    types: [Type<UInt256>(), Type<UInt256>(), Type<UInt256>(), Type<UInt256>()],
    data: EVM.dryCall(from: coa.address(), to: aggregator, data: getAllData,
                      gasLimit: 200_000, value: EVM.Balance(attoflow: 0)).data
)
```

### Chunking state-mutating ops across transactions

A loop of `coa.call` invocations for state-mutating work (e.g. distributing rewards to N EVM addresses) is a guaranteed ceiling overrun for any meaningful `N`. The fix is to split the work into multiple transactions and reconstruct atomicity at the protocol layer using the multi-tx escrow pattern:

- Phase the work into `Pending → Executing → Settled` on a Cadence resource.
- Each transaction processes a chunk of `N_chunk` items.
- The escrow phase prevents partial-state observation by external callers.

See [coa-lifecycle.md](coa-lifecycle.md) for the COA-as-escrow pattern and [multi-tx-escrow.md](../../cadence-lang/references/multi-tx-escrow.md) for the phase discipline. The cost is that intra-chunk atomicity is preserved but cross-chunk atomicity is not — design the escrow so that a partial run is observably "in progress" until the final chunk settles.

### Off-chain pre-computation

Heavy ABI encoding for many calls can be done off-chain. The Cadence transaction takes the pre-encoded `[UInt8]` bytes as a transaction argument rather than concatenating strings or constructing types on-chain. This trades trust (the off-chain encoder must be correct) for CU — but ABI encoding is deterministic and easy to assert against, so it is usually safe.

### Move work into EVM

Iterating over thousands of EVM addresses from Cadence will blow the CU budget long before it blows the EVM gas budget, because each `coa.call` has Cadence-side overhead per iteration. A Solidity `Multicall` contract that loops in Solidity costs EVM gas (which is paid from the COA balance and can be raised), not CU. For batched writes, this is almost always the right answer.

## Decision Matrix: Cadence vs EVM

| Workload | Do it in | Why |
|---|---|---|
| Math under 8 decimals of precision | Cadence (`UFix64`) | Native fixed-point, no overflow worries, cheap per op |
| High-precision math (24 decimals, large value range) | Cadence (`UFix128`) | See [numeric-fixed-point.md](../../cadence-lang/references/numeric-fixed-point.md) (in `feature/cadence-async-primitives` branch). Avoids the Solidity wad/ray mess and keeps semantics in one place. |
| Storage of user balances | Cadence (resources) | Resource semantics give linear-typed safety; CU cost is bounded and predictable |
| Iterating over thousands of entries with updates | EVM (Solidity loop) | Solidity loops cost gas, not CU. The 9,999 CU cap kicks in fast in Cadence loops with EVM calls. |
| Calling existing EVM protocols (Uniswap, Aave forks) | CrossVM (`coa.call`) | The only practical option for composability with deployed EVM code |
| Holding ERC20s as collateral | CrossVM (COA custody) | The COA is the ERC20 holder; Cadence enforces the policy on top |
| One-off view (single ERC20 `balanceOf`) | CrossVM (`EVM.dryCall`) | Small CU cost, no gas charge |

## Anti-Pattern: Looping `coa.call` Without Measuring

```cadence
// ❌ Cannot fit more than ~110 iterations under 9,999 CU.
// Each iteration burns CU on the call overhead, ABI encoding,
// result decoding, and the assert. With `coa.call` at ~7 CU/op
// marginal at small N growing quadratically (measured fit
// 0.108·N² + 6.7·N), the loop dies around N≈110 on emulator at
// the default 9,999 cap. The naive linear estimate suggests
// N≈1,400 — it overstates capacity by ~13×.
transaction(recipients: [EVM.EVMAddress], amounts: [UInt256]) {
    prepare(acct: auth(BorrowValue) &Account) {
        let coa = acct.storage.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(
            from: /storage/evm) ?? panic("no COA")
        var i = 0
        while i < recipients.length {
            let data = EVM.encodeABIWithSignature(
                "transfer(address,uint256)",
                [recipients[i], amounts[i]]
            )
            let r = coa.call(to: erc20, data: data, gasLimit: 100_000,
                             value: EVM.Balance(attoflow: 0))
            assert(r.status == EVM.Status.successful, message: "transfer failed")
            i = i + 1
        }
    }
}
```

```cadence
// ✅ Push the loop into Solidity. The Cadence side issues one
// coa.call to a Multicall/BatchTransfer contract. Cadence CU is
// roughly constant; EVM gas scales with N but is paid from COA
// balance and can be raised via `gasLimit`.
transaction(recipients: [EVM.EVMAddress], amounts: [UInt256]) {
    prepare(acct: auth(BorrowValue) &Account) {
        let coa = acct.storage.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(
            from: /storage/evm) ?? panic("no COA")
        let data = EVM.encodeABIWithSignature(
            "batchTransfer(address,address[],uint256[])",
            [erc20, recipients, amounts]
        )
        let r = coa.call(to: batcher, data: data, gasLimit: 3_000_000,
                         value: EVM.Balance(attoflow: 0))
        assert(r.status == EVM.Status.successful, message: "batch failed")
    }
}
```

## Common Pitfalls

1. **Assuming EVM gas exhaustion reverts the transaction.** It does not. Only the EVM call reverts. The Cadence transaction continues. Always `assert(result.status == EVM.Status.successful, ...)` after every `coa.call` whose success is required for protocol correctness.
2. **Setting `gasLimit` too low to "save fees".** EVM gas on Flow is generally cheap relative to the cost of a silent failure. Pick `gasLimit` based on the EVM function's **worst-case** path (longest branch, largest dynamic array), not its average.
3. **Doing CU-heavy work between `coa.call`s.** Every Cadence operation between EVM hops eats into the same 9,999 budget. Move heavy aggregation/sorting/filtering into Solidity if you must do it inline with EVM calls.
4. **Not budgeting for the prologue.** The fixed overhead of a transaction (signing, COA borrow, capability checks) is **~4 CU** for an empty `noop` (5-sample mean, 0% variance); add a few more for COA borrow and capability checks. The prologue itself is negligible — it is the per-call intercepts in the cost table above that dominate fixed overhead, not the prologue.
5. **Sweeping at high N without binary search.** Going from `N=128` (passes) to `N=256` (fails) tells you nothing about the cliff. Binary-search between the last passing and first failing N to find the exact ceiling.
6. **Trusting emulator CU as production CU.** Emulator under-weights storage reads. Always re-sweep on testnet before quoting a `MAX_SAFE_N` to users.
7. **Forgetting that calldata size affects CU on the Cadence side.** `EVM.encodeABI([largeArray])` is not free. Sweep with varying calldata size to characterise the encoding slope independently.
8. **Treating `EVM.dryCall` as "free".** It does not charge EVM gas, but it still consumes Cadence CU for the bridge layer and the decoding of the result data. Large array reads can be surprisingly expensive.
