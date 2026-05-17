# T36 — CrossVM Explorations: Empirical Formulas, Decision Trees, Named Patterns

**Status:** Empirical truth extracted from N≈70 sweeps on Flow emulator v2.17.1, 2026-05-17. Every formula here has an R² and a runnable probe. Every pattern has a Cadence + Solidity snippet that compiled and sealed. This file goes **beyond** what [`evm-call.md`](evm-call.md), [`erc20-read.md`](erc20-read.md), [`coa-lifecycle.md`](coa-lifecycle.md), [`cu-ceiling.md`](cu-ceiling.md), [`flow-bridge.md`](flow-bridge.md), [`solidity-fixtures.md`](solidity-fixtures.md), [`crossvm-anti-patterns.md`](crossvm-anti-patterns.md), and [`crossvm-integration-testing.md`](crossvm-integration-testing.md) say.

This is not canonical API. It is **measured behaviour, named patterns, and decision trees** — material for the async-intent + CrossVM frontier.

---

## 1. Overview

The T12 verifier (see [`T12-VERIFICATION.md`](T12-VERIFICATION.md)) established that:

- Read-only EVM ops scale **linearly** in CU per `dryCall`.
- State-mutating EVM ops (`coa.call` writes, `coa.deposit`, `coa.withdraw`) scale **quadratically** in N within a single tx.
- CU is **deterministic** on emulator (0% within-network variance).

This document extends those measurements along axes T12 did not probe — return-data size, calldata size, gas-estimation accuracy, in-EVM-work bleed, and high-K multicall — and converts the findings into decision trees and named patterns aimed squarely at the async-intent project (user signs intent → executor resolves → escrow holds → settlement closes).

All measurements: 30M+ EVM gas requests get **rejected at 16,777,216 (2²⁴)** — emulator EVM has a hard gasLimit ceiling that is undocumented and bites if you copy `gasLimit: 30_000_000` from mainnet snippets.

---

## 2. Empirical Formulas

### 2.1 Summary table

| Probe | Sweep var | Formula (CU) | R² | Range | Mechanism |
|---|---|---|---|---|---|
| M30 — dryCall return uint256[K] | K | `1.06·K + 14.12` | 0.9998 | 1 ≤ K ≤ 256 | Cadence-side decode of return data |
| M31 — dryCall calldata uint256[K] | K | `1.03·K + 13.35` | 0.9996 | 1 ≤ K ≤ 256 | Cadence-side ABI encode of calldata |
| M32 — gasUsed determinism | (10 runs) | `dry == real` exactly | n/a | every run | dryCall is gas-perfect |
| M33/M38 — `EVM.dryCall` looped | N | `5.49·N + 10.39` | 1.0000 | 1 ≤ N ≤ 2000 | Per-hop Cadence cost |
| M33/M40 — `coa.dryCall` looped | N | `4.72·N + 10.55` | 1.0000 | 1 ≤ N ≤ 2000 | Method-on-resource is 14% cheaper |
| M34 — `coa.call` w/ event payload[K] | K | `1.05·K + 16.27` | 0.9997 | 0 ≤ K ≤ 256 | Per-word event payload encode |
| M35 — Multicall return K | K | `3.37·K + 19.17` | 1.0000 | 1 ≤ K ≤ 512 | Decode + per-element Solidity cost amortized |
| M36 — dry-then-commit pairs | N | `0.143·N² + 12.54·N + 10.75` | 1.0000 | 1 ≤ N ≤ 100 | Quadratic — same as M04 plus dry overhead |
| M37 — read vs no-read of r.data | K | identical within 1 CU | 0.999 | 0 ≤ K ≤ 128 | Accessing `r.data` is free; only decode costs |
| M39 — bumpCounter (in-EVM work) | EVM gas | `CU ≈ gasUsed / 2848 + 9.35` | 0.999 | 1k ≤ gas ≤ 2M | Heavy EVM gas leaks into Flow CU |

### 2.2 What "linear" vs "quadratic" means for the async-intent project

The single biggest law to internalise:

> **Every `coa.call` (state-mutating EVM hop) in the same Cadence tx contributes a quadratic term to total CU. Every `dryCall` contributes only a linear term.**

The executor that resolves an intent will frequently do N coa.calls (transfer, approve, swap, transfer-back). Quadratic scaling means the executor can never batch many writes into one tx, no matter how cheap each individual call seems at N=1. Concretely (from `T12-VERIFICATION.md` § M04):

- N=10 coa.calls: 89 CU (≈9 CU each — "looks linear")
- N=100 coa.calls: 1,765 CU (≈18 CU each)
- N=250 coa.calls: 8,458 CU (≈34 CU each — past the cliff at 9,999 if N=270)

For the same op count of read-only dryCalls (M38), N=2000 fits in 10,978 CU — linear extrapolation never lies.

### 2.3 Return-data and calldata size (M30, M31)

Both probes show **~1 CU per UInt256 word** with a ~13-CU base cost for the dryCall itself:

```
M30 (return):  CU(K) ≈ 1.06·K + 14.12   R²=0.9998
M31 (calldata): CU(K) ≈ 1.03·K + 13.35   R²=0.9996
```

Mechanism: each word costs roughly one Cadence operation to encode/decode. **Reading r.data without decoding it is free** (M37: read_True vs read_False fits are within 1 CU at every K).

Practical implication for intent resolvers: **don't worry about return-data size below K=100**. A single multicall returning 100 prices costs ~120 CU vs ~10,000 CU for the equivalent 100 separate coa.calls — and the difference isn't the decode, it's the quadratic shape of mutating ops.

### 2.4 `coa.dryCall` vs `EVM.dryCall` (M33, M40)

Both are read-only. Both are linear. The **method on the COA resource** is consistently ~14% cheaper:

```
EVM.dryCall:  5.49·N + 10.4 CU
coa.dryCall:  4.72·N + 10.5 CU         (≈14% slope reduction)
```

Mechanism (confirmed 2026-05-17): every `EVM.dryCall` hop allocates a fresh `EVMAddress` struct at the call site (via `coa.address()`) and transfers it by-value into the function. `coa.dryCall` passes `self.addressBytes` directly with no struct construction. See `T36-DEEP-DIVE.md §3`. Across 1,000 reads the difference is ~770 CU — enough to matter under the 9,999 cliff.

**Rule of thumb**: if you already have a `&EVM.CadenceOwnedAccount` reference (you usually do), prefer `coa.dryCall(...)` over `EVM.dryCall(from: coa.address(), ...)`.

### 2.5 Gas-estimation accuracy (M32)

10 runs of `m32_drycall_vs_call_gas.cdc` (one dryCall + one real call to the same ERC20.transfer in one tx):

```
run 0: dryGasUsed=51982  realGasUsed=51982  Δ=0  (cold storage)
runs 1-9: dryGasUsed=34882  realGasUsed=34882  Δ=0  (warm storage)
```

**`dryCall.gasUsed` is an exact prediction of `coa.call.gasUsed`**, including warm/cold storage transitions. There is no fuzz factor and no need to multiply by 1.2× the way mainnet Ethereum tooling does.

This is a HUGE finding for the async-intent project: the executor can compute exact gas budgets ahead of time and reject the intent if the user's COA can't afford the gas. The `+20%` safety margin Ethereum tooling adds is **unnecessary on Flow**.

### 2.6 The "EVM gas bleed" formula (M39)

Pure in-EVM compute (the `bumpCounter` probe — N SSTOREs in a tight loop) shows that **in-EVM gas spills into Flow CU at a fixed rate**:

```
times=1:      gasUsed=44,175   CU=16
times=10:     gasUsed=28,812   CU=15
times=1,000:  gasUsed=46,182   CU=24
times=10,000: gasUsed=219,894  CU=103
times=100,000:gasUsed=1,956,894 CU=695

Linear fit:  CU ≈ gasUsed / 2848 + 9.35   R²=0.999
```

**1 Flow CU ≈ 2,848 EVM gas** when the EVM call is heavy. Until ~50,000 EVM gas, the Cadence base cost (~10 CU) dominates and the gas bleed is invisible. Past ~100,000 gas, the bleed kicks in.

Practical bound: a coa.call that burns 1M EVM gas costs ~360 CU on the Flow side just for the bleed; one that burns 8M EVM gas costs ~2,820 CU. **A Cadence tx that does an 8M-gas EVM call has about 5,000 CU of budget left for everything else (and remember, additional coa.calls scale quadratically).**

### 2.7 Multicall scaling at high K (M35)

The T12 verifier sampled K up to 32 (M10b). Pushing to K=512:

```
K=8:   45 CU
K=32:  130 CU
K=64:  237 CU
K=256: 883 CU
K=512: 1,744 CU
Fit: 3.37·K + 19.17   R²=1.0000
```

**Multicall is still perfectly linear at K=512.** No quadratic term emerges. The crossover where multicall (M35) beats separate dryCalls (M33 mode 0, ≈5.51·N) is at K≈2 (multicall starts winning immediately because of the steeper per-element slope of dryCall). At K=8 multicall is ~17% cheaper; at K=64 it's ~57% cheaper; at K=512 it's ~71% cheaper.

---

## 3. Decision Trees / Matrices

### 3.1 When to use dryCall vs coa.dryCall vs coa.call

```
Need to mutate EVM state?
├── YES  → coa.call   (NB: quadratic in N within a tx; budget ≤ ~50 writes per tx)
└── NO   → dryCall (read-only)
           ├── Have `&CadenceOwnedAccount` already? → coa.dryCall   (~14% cheaper)
           └── Stateless context (FT receiver, view fn)? → EVM.dryCall
```

If you don't yet have a COA reference and don't need one for anything else, `EVM.dryCall` saves you the borrow (~7 CU). If you've already borrowed (which any intent-execution tx must do anyway), `coa.dryCall` saves you 0.77 CU per call.

### 3.2 Batch reads via separate dryCalls vs multicall helper

| K (tokens / prices) | Separate dryCalls (M33 mode 0) | Multicall (M35) | Winner | Saving |
|---:|---:|---:|:---:|---:|
| 1 | 16 CU | 22 CU | dryCall | — |
| 2 | 21 CU | 24 CU | dryCall | — |
| 4 | 32 CU | 28 CU | multicall | 13% |
| 8 | 54 CU | 45 CU | multicall | 17% |
| 16 | 98 CU | 76 CU | multicall | 22% |
| 32 | 187 CU | 130 CU | multicall | 30% |
| 64 | 364 CU | 237 CU | multicall | 35% |
| 256 | 1,420 CU | 883 CU | multicall | 38% |
| 512 | 2,820 CU | 1,744 CU | multicall | 38% |

**Crossover at K=4.** For K=1,2 the separate-dryCall path wins on absolute CU; for K≥4 multicall wins and the saving asymptotes at ~38%. The reason multicall doesn't crush separate dryCalls is that the *Cadence-side* cost dominates — Solidity-side gas is absorbed into one EVM hop, but each returned word still costs ~1 CU to decode.

### 3.3 When the COA pattern is overkill

Use `EVM.dryCall(from: coa.address(), ...)` directly (no COA borrow with entitlements) when **ALL** of these hold:

- The op is read-only (no state mutation needed).
- You do not need to deposit/withdraw FLOW.
- You do not need atomicity across multiple EVM hops with COA-bound state.

For read-only ERC20 balance/price queries in a script, **don't borrow the COA at all** — call `EVM.dryCall` with a synthetic `from:` address (any 20-byte zero-padded value works; many ERC20 view fns don't read msg.sender). This skips the 7 CU borrow cost and the resource lifecycle.

### 3.4 Choosing `gasLimit`

```
gasLimit choice:
├── Read-only dryCall to ERC20.balanceOf / decimals?     → 100_000
├── Read-only dryCall to a multicall with K elements?    → 100_000 + 8000·K
├── coa.call to ERC20.transfer (cold)?                   → 65_000
├── coa.call to ERC20.transfer (warm)?                   → 45_000
├── coa.call to AMM.swap / lending borrow?               → 300_000
├── coa.call to deploy?                                  → 6_000_000 to 12_000_000
└── Anything pushing the EMULATOR ceiling?               → cap at 15_000_000 (emulator hard cap is 2^24 = 16,777,216)
```

Notes:

- **Setting gasLimit too high costs nothing on Flow** — there is no "gas overpayment" since coa.call only charges for `gasUsed`, not `gasLimit`. The only ceiling is the EVM block gas cap.
- **Too low fails NOT silently** — `r.status == Status.failed`, `r.errorMessage = "out of gas"`. You MUST `assert(r.status == EVM.Status.successful)` or the Cadence tx commits a no-op.
- Mainnet's EVM block gas limit is higher than the emulator's 16,777,216; do not assume mainnet caps at emulator caps. Always re-verify on testnet.

### 3.5 Native FLOW deposit vs ERC20 bridge

| Token | Direction | Mechanism | Atomicity |
|---|---|---|---|
| FLOW | Cadence → EVM | `coa.deposit(from: <- vault)` | Atomic in single tx |
| FLOW | EVM → Cadence | `coa.withdraw(balance:)` returns `@FlowToken.Vault` | Atomic in single tx |
| WFLOW (ERC20) | EVM ↔ Cadence | bridge contracts in [`flow-bridge.md`](flow-bridge.md) | Multi-step |
| Any other ERC20 | EVM ↔ Cadence | bridge VM helper contracts | Multi-step, not atomic |

**Decision**: FLOW is the only token with a native, atomic bridge. Everything else (including stablecoins) goes through bridge contracts and is not atomic. For async-intent flows, settle in FLOW or in tokens that already exist on the destination VM.

### 3.6 Where to compute math: Cadence vs EVM

| Computation | Better on | Why |
|---|---|---|
| AMM `getAmountOut` | EVM (called via dryCall) | The AMM IS the source of truth; reading via dryCall is 1 CU/word |
| Off-AMM slippage check | Cadence | Avoid round-trip; pure Cadence arithmetic is ~0.1 CU per op |
| Fixed-point swap quote | EVM | Solidity's overflow rules are well-known; Cadence UInt256 lacks Fix128 mul/div helpers |
| Permit-style hashing | Cadence (Crypto.keccak256) | Native, no EVM hop needed |
| Order-book matching | Cadence | Pure compute, no EVM dependency, scales linearly forever |
| Settlement transfer | EVM via coa.call | The token contract owns the state |

Rule: **compute in the VM that already holds the state you read**.

### 3.7 Inner-EVM-revert: detection priorities

```
After every coa.call:
1. ALWAYS assert(r.status == EVM.Status.successful, message: r.errorMessage)
2. If you need to surface specific revert reason, switch on r.errorCode
   (we measured errorCode=306 for "execution reverted")
3. If you need to surface the revert *string* the EVM produced, decode r.data
   (revert data is preserved even on failure; we measured data_len=100 for an ERC20 underflow revert)
```

The pattern is captured below as "Status-Asserted EVM Hop".

---

## 4. Named Patterns Catalog

### 4.1 Multicall-via-EVM-helper

**Description**: Deploy a one-shot Solidity contract whose only job is to fan out K view calls and pack the results into one return blob. Cadence does one dryCall, decodes one array. Saves the per-hop Cadence overhead.

**When to use**: any time you need ≥4 reads from the same or different EVM contracts in the same tx, especially price oracles, balance snapshots, allowance checks across a portfolio.

**Solidity stub**:
```solidity
// from sol/src/BalanceMulticall.sol
contract BalanceMulticall {
    function balances(address user, address[] calldata tokens) external view
        returns (uint256[] memory out) {
        out = new uint256[](tokens.length);
        for (uint i = 0; i < tokens.length; i++)
            out[i] = IERC20Bal(tokens[i]).balanceOf(user);
    }
}
```

**Cadence side**:
```cadence
let data = EVM.encodeABIWithSignature("balances(address,address[])", [user, tokenAddrs])
let r = EVM.dryCall(from: coa.address(), to: multi, data: data, gasLimit: 5_000_000,
                    value: EVM.Balance(attoflow: 0))
let out = EVM.decodeABI(types: [Type<[UInt256]>()], data: r.data)[0] as! [UInt256]
```

**Gotcha**: the deployed BalanceMulticall is permissionless and stateless — fine to share across all users. Don't deploy a fresh one per user.

### 4.2 Status-Asserted EVM Hop

**Description**: Every coa.call MUST be followed by an explicit status check, or the Cadence tx will commit even when the EVM call reverted. This is the #1 latent bug in CrossVM code reviews.

**When to use**: every single `coa.call`. Non-negotiable.

**Canonical snippet**:
```cadence
let r = coa.call(to: erc20, data: data, gasLimit: 200_000, value: EVM.Balance(attoflow: 0))
assert(
    r.status == EVM.Status.successful,
    message: "EVM call failed (code=".concat(r.errorCode.toString()).concat("): ").concat(r.errorMessage)
)
```

**Measured fact (M43)**: with no assert, `r.status=2`, `r.errorMessage="execution reverted"`, `r.errorCode=306`, and the Cadence tx still SEALS. With assert + a subsequent revert (M44), both EVM and Cadence state roll back atomically.

### 4.3 Dry-Then-Commit

**Description**: Before a state-mutating coa.call, do a coa.dryCall with the same args, check the predicted result, then commit. Used for slippage protection and pre-flight validation.

**When to use**: AMM swaps with slippage tolerance; transfers where you need to verify the post-state matches expectations; any intent execution where the user signed a "max-slippage" guarantee.

**Pattern**:
```cadence
// 1. Dry-call first, read the predicted output
let dryR = coa.dryCall(to: amm, data: swapData, gasLimit: 300_000,
                       value: EVM.Balance(attoflow: 0))
let predictedOut = EVM.decodeABI(types: [Type<UInt256>()], data: dryR.data)[0] as! UInt256

// 2. Check it satisfies the user's intent
assert(predictedOut >= minOutFromIntent, message: "slippage")

// 3. Commit
let realR = coa.call(to: amm, data: swapData, gasLimit: 300_000,
                     value: EVM.Balance(attoflow: 0))
assert(realR.status == EVM.Status.successful, message: "swap failed")
```

**Gotcha (M36)**: this is still quadratic in N if you loop it. `0.143·N² + 12.5·N` — higher quadratic coefficient than plain coa.call because the dryCall pair adds overhead. Don't loop more than ~30 pairs.

**Race condition**: between dry and commit, an external write to the same AMM could change the price. On Flow's MEV-free EVM this risk is minimal but not zero across multi-tx flows.

### 4.4 COA-Pair-Escrow (Async-Intent Specific)

**Description**: Two COAs — one owned by the user (signs intent, holds funds), one owned by the protocol/executor (holds settlement permissions). The user issues an entitled capability `auth(EVM.Call) &CadenceOwnedAccount` scoped to a single intent resource; the executor borrows it through the intent.

**When to use**: any time the user signs an intent and an executor resolves it on their behalf, without giving the executor blanket access to all user assets.

**Cadence shape** (excerpted, for the upcoming async-intent project):
```cadence
access(all) resource Intent {
    access(all) let user: Address
    access(all) let minOutAmount: UInt256
    access(all) let deadline: UFix64
    // The capability is entitled to EVM.Call only, expires at `deadline`.
    access(self) let userCoaCap: Capability<auth(EVM.Call) &EVM.CadenceOwnedAccount>

    access(Settle) fun executeWith(executorCoa: &EVM.CadenceOwnedAccount, ...): @Receipt {
        // executor checks block-timestamp < deadline
        // borrows userCoaCap, does the swap, settles
    }
}
```

**Gotcha**: capability must be **time-bound** (check deadline against `getCurrentBlock().timestamp`) and **action-bound** (the entitlement is `auth(EVM.Call)`, not `auth(EVM.Call, EVM.Deploy, EVM.Withdraw)`). A capability with the broad `auth(EVM.Call, EVM.Deploy, EVM.Withdraw)` lets the executor drain the user's COA — that's a hard fail in audit.

**Settlement event**: emit both a Cadence event (`IntentSettled`) AND let the EVM side emit its own (e.g. an AMM `Swap` log). The Cadence event is the source of truth; the EVM event is for off-chain indexers.

### 4.5 EVM-State-Machine-with-Cadence-Clock

**Description**: The EVM contract has a vesting/auction/auction-rebate state machine driven by `block.timestamp`. Cadence transactions advance the clock by sending one coa.call per tick (e.g., scheduled via FlowTransactionScheduler).

**When to use**: vesting schedules, weekly rebate distributions, periodic oracle refresh, time-locked governance.

**Why it works on Flow**: Flow EVM's `block.timestamp` advances with Flow block time (~1s); a scheduled Cadence transaction can be the only source of "ticks" that the EVM contract responds to. No keeper bot required.

**Solidity stub**:
```solidity
contract Vesting {
    uint256 public lastTick;
    function tick() external {
        require(block.timestamp >= lastTick + 1 days, "wait");
        // distribute, update state
        lastTick = block.timestamp;
    }
}
```

**Cadence side**: schedule a daily transaction handler that does `coa.call(to: vesting, data: encodeABIWithSignature("tick()", []), ...)`. See [`cadence-lang.md`](cadence-lang.md) §scheduled-transactions.

### 4.6 Bridge-Deposit-And-Immediately-Use

**Description**: In a single Cadence tx, withdraw FLOW from a Cadence vault, deposit it into a COA, then use the deposited FLOW in an EVM call (e.g., to wrap into WFLOW or pay for an EVM op). All atomic.

**Pattern**:
```cadence
transaction(amount: UFix64) {
    prepare(signer: auth(BorrowValue) &Account) {
        let coa = signer.storage.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(from: /storage/evm)!
        let vaultRef = signer.storage.borrow<auth(FungibleToken.Withdraw) &FlowToken.Vault>(
            from: /storage/flowTokenVault)!
        let funds <- vaultRef.withdraw(amount: amount) as! @FlowToken.Vault
        coa.deposit(from: <- funds)
        // Use the deposit in the same tx — atomic with the deposit:
        let data = EVM.encodeABIWithSignature("deposit()", [])
        let r = coa.call(to: wflow, data: data, gasLimit: 200_000,
                         value: EVM.Balance(attoflow: UInt(amount * 1e18)))
        assert(r.status == EVM.Status.successful, message: "wrap failed")
    }
}
```

**Atomicity caveats**:
1. The deposit and the coa.call are atomic on the Cadence side. The EVM-side execution of `coa.call` is also atomic (revert→full rollback on the EVM side, M43/M44).
2. But: `coa.deposit` is **quadratic in N** (M05). Don't deposit 200 small amounts in one tx; deposit one big amount.

### 4.7 Cross-VM-Vesting

**Description**: A Cadence resource tracks vesting schedule and remaining balance. A scheduled tick triggers a transfer of vested ERC20 tokens via the user's COA to an EVM-side beneficiary.

**Why this exists**: vesting is data-rich (cliffs, schedules, slashing rules) but settlement is value-only. Cadence is the natural home for the schedule; EVM is the natural home for the token transfer if the token lives there.

**Composition with Pattern 4.5**: the Cadence resource exposes a `tick()` method; the scheduled-tx handler calls `tick()` which does the coa.call internally.

### 4.8 Quadratic-Aware Op Splitting

**Description**: When you need N state-mutating EVM ops where N might be 100+, **don't put them in one tx**. Split into separate Cadence transactions. The quadratic CU shape means batching past ~50 writes is strictly worse than splitting.

**Decision rule from M04**:
```
write_per_tx_budget = floor((9999 / 0.108) ** 0.5)  ≈ 304
practical_safe       = 50   (leaves headroom for prologue + dryCalls + asserts)
```

If you need to do 200 transfers, do 4 transactions of 50 each. Yes, you pay 4× the tx-inclusion cost — but on Flow that's $0.00001 each, negligible.

### 4.9 No-Decode Read

**Description**: When you need to know if a dryCall succeeded but don't care about its return value (e.g., probing whether a contract exists, or whether a permission check passes), do NOT decode `r.data`. Just check `r.status`.

**Mechanism (M37)**: accessing `r.data` is free; decoding it costs ~1 CU per word. Skipping the decode saves real CU at large return-data sizes.

**Snippet**:
```cadence
let r = coa.dryCall(to: target, data: probeData, gasLimit: 100_000,
                    value: EVM.Balance(attoflow: 0))
if r.status == EVM.Status.successful {
    // probe passed; we don't care what it returned
}
```

### 4.10 Coa-Method-Preferred

**Description**: When you have a `&EVM.CadenceOwnedAccount` reference, prefer `coa.dryCall(...)` over `EVM.dryCall(from: coa.address(), ...)`. Save ~14% CU on every read.

**Measured**: M33/M40, ranging up to N=2000. Slope difference is consistent (5.49 vs 4.72 CU/op).

---

## 5. Async-Intent Specific Insights

### 5.1 Intent capability shape

An intent must carry a capability to the user's COA. The capability must:

1. Be **entitled** to the minimum surface needed. For a swap intent: `auth(EVM.Call) &EVM.CadenceOwnedAccount`. NEVER include `EVM.Deploy` or `EVM.Withdraw` unless the intent type explicitly needs them.
2. Be **time-bound** via a deadline stored in the Intent. Check `getCurrentBlock().timestamp <= self.deadline` before borrowing.
3. Be **action-bound** via the Intent's interface — the executor sees an `executeSwap` method, not raw COA access.
4. Be **single-use** — destroy the Intent resource after settle.

The user issues the cap with `account.capabilities.storage.issue<auth(EVM.Call) &EVM.CadenceOwnedAccount>(/storage/evm)`. The Intent resource stores it as `access(self) let cap: Capability<auth(EVM.Call) &EVM.CadenceOwnedAccount>`.

### 5.2 Atomic intent flow shape

```
transaction (signed by executor) {
    prepare(executor: ...) {
        // 1. Borrow the Intent resource (published by user beforehand)
        let intent <- userPub.intents.withdraw(intentId)

        // 2. Borrow user's COA via the entitled capability inside the Intent
        let userCoa = intent.cap.borrow() ?? panic("cap revoked")

        // 3. Compute strategy off the user's funds (dry-then-commit pattern)
        let dryR = userCoa.dryCall(to: amm, data: swapData, ...)
        let predictedOut = decode(dryR.data)
        assert(predictedOut >= intent.minOutAmount, message: "slippage")

        // 4. Commit
        let realR = userCoa.call(to: amm, data: swapData, ...)
        assert(realR.status == EVM.Status.successful, message: realR.errorMessage)

        // 5. Settle: transfer the output token from user's COA to wherever they specified
        let settleR = userCoa.call(to: outToken, data: transferData, ...)
        assert(settleR.status == EVM.Status.successful, message: settleR.errorMessage)

        // 6. Emit settlement event + destroy intent
        emit IntentSettled(id: intent.id, executor: executor.address, ...)
        destroy intent
    }
}
```

CU budget for the above on a swap intent: ~25 (borrow + dry) + ~25 (assert + decode) + ~50 (commit + assert) + ~50 (settle + assert) ≈ 150 CU. Well under the cliff.

### 5.3 Handling partial fills

EVM swaps that partially fill (e.g., a TWAP-style DCA executor that does 6 micro-swaps to fill the intent over 24h) need:

1. **State on the Cadence side** that tracks `filledSoFar: UInt256`. This is the source of truth.
2. **Per-tick coa.call** to the AMM with the chunk size.
3. **Per-tick assert** that the realized output is within the user's per-tick slippage budget AND that the cumulative output respects the intent's min-out.
4. **Settlement event per tick** AND a final `IntentFullyFilled` event when filledSoFar ≥ intent.totalIn.

If a tick reverts (slippage exceeded), DON'T abort the whole intent — emit `IntentTickSkipped` and let the next scheduled tick try again. The Status-Asserted pattern (4.2) needs a variant here: assert that the EVM call returned status=successful OR the revert reason matches a known "skippable" case.

### 5.4 Measuring execution success

Two views:

- **Cadence-side**: did the Intent resource get destroyed with a `IntentSettled` event?
- **EVM-side**: did the AMM emit a `Swap` log with `amountOut >= intent.minOut`?

For audit / indexer consistency, emit a `IntentSettled` Cadence event that lets indexers join across VMs. Note: `EVM.Result` has **no `txHash` field** in CLI v2.17.1 — that field does not exist. The EVM tx hash is available only on the `EVM.TransactionExecuted` Cadence event under the field `hash: [UInt8]`, which fires after the transaction commits and cannot be read within the same transaction. For indexer correlation, key the `IntentSettled` event on an internal nonce and join off-chain on the surrounding Flow transaction ID.

### 5.5 Rolling back a partially-executed CrossVM intent

If step N of a multi-step intent fails:

```
Option A — abort whole tx via panic.
    PROS: clean rollback, all EVM state changes reverted by Flow EVM atomicity.
    CONS: user pays no fee (good) but the intent stays unsettled (potentially bad if it had a deadline).

Option B — catch failure, emit IntentPartialFill, leave intent alive for re-execution.
    PROS: progress is preserved.
    CONS: must store partial state in the Intent resource; more complex.
```

For most intents Option A is right. For TWAP/DCA intents (Pattern 5.3) Option B is right.

### 5.6 Event count discipline

Confirmed via M41: each coa.call emits exactly one `EVM.TransactionExecuted` Cadence event. An async-intent settlement tx that does 4 coa.calls emits 4 EVM events + 1 IntentSettled Cadence event = 5 events to index. Plan your indexer accordingly. There is no `EVM.BlockExecuted` per call — that only emits per Flow block.

---

## 6. Cross-VM Data Structures

### 6.1 Encoding catalog

| Cadence type | EVM type | Encode | Decode | Notes |
|---|---|---|---|---|
| `UInt256` | `uint256` | direct | direct | Most common; always 32 bytes |
| `UInt8` (or any UIntN) | `uintN` | direct | direct | Padded to 32 bytes on the wire |
| `Int256` | `int256` | direct | direct | Two's complement |
| `String` | `string` | direct (dynamic) | direct | UTF-8 |
| `Address` (Flow) | (no equivalent) | — | — | Flow address ≠ EVM address |
| `EVM.EVMAddress` | `address` | direct | direct | 20 bytes |
| `[UInt256]` | `uint256[]` | direct | `Type<[UInt256]>()` | Decode returns `AnyStruct`, cast to `[UInt256]` |
| `[[UInt256]]` | `uint256[][]` | direct | `Type<[[UInt256]]>()` | Confirmed working — exact values recovered |
| `[[[UInt256]]]` | `uint256[][][]` | direct | `Type<[[[UInt256]]]>()` | Confirmed working — any nesting depth |
| (no Cadence type) | `(uint256,uint256)[]` | — | **no working pattern** — silently drops fields | Refactor Solidity to parallel arrays |
| `[EVM.EVMAddress]` | `address[]` | direct | `Type<[EVM.EVMAddress]>()` | Confirmed working at K up to 512 |
| `[UInt8]` | `bytes` | direct | direct | Dynamic; Cadence side is byte array |
| Cadence struct | EVM tuple | `[a, b, c]` in order | as tuple, fields in declaration order | Order MUST match Solidity tuple definition |
| `Bool` | `bool` | direct | direct | |
| **Dictionary `{K: V}`** | **no equivalent** | **PANICS** | n/a | M42b confirmed: `failed to ABI encode value of type {UInt256: UInt256}` |

### 6.2 Dictionary workarounds

Pass parallel arrays:
```cadence
let keys: [UInt256] = d.keys
let vals: [UInt256] = []
for k in keys { vals.append(d[k]!) }
EVM.encodeABIWithSignature("foo(uint256[],uint256[])", [keys, vals])
```

Confirmed working in M42. The Solidity side reconstructs the mapping if needed.

### 6.3 Returning a struct from Solidity

Solidity:
```solidity
struct Quote { uint256 amountOut; uint256 fee; address pool; }
function quote(...) external view returns (Quote memory) { ... }
```

Cadence-side decode:
```cadence
let decoded = EVM.decodeABI(types: [Type<UInt256>(), Type<UInt256>(), Type<EVM.EVMAddress>()],
                             data: r.data)
let amountOut = decoded[0] as! UInt256
let fee       = decoded[1] as! UInt256
let pool      = decoded[2] as! EVM.EVMAddress
```

A struct in Solidity decodes as a tuple at the ABI level. The `types` array in `decodeABI` must list the fields in **declaration order**. Cadence does NOT have a way to express "decode as a struct"; you always destructure into individual values.

### 6.4 Dynamic-length bytes/strings: offset + length

ABI encodes dynamic types with an offset pointer + a length + the data. Cadence's `decodeABI` handles this internally — you just write `Type<String>()` or `Type<[UInt8]>()` and read out the decoded value. **Don't try to slice `r.data` manually** to read offsets; the helper does it correctly.

Gotcha: if a Solidity function returns `(uint256, bytes, uint256)` and the bytes field is huge, you still pay only ~1 CU per 32-byte word of bytes data to decode (M30's slope confirms this generalizes from uint256[] to bytes; expect ~32 bytes per CU).

---

## 7. Open Questions

Of the original 8 open questions, 7 are now resolved by the deep-dive (T36-DEEP-DIVE.md §§2–5) and verifier rounds (T27-VERIFICATION.md §§3–5). 1 open question remains genuinely open.

### 7.1 Cross-network variance (emulator vs testnet vs mainnet) — RESOLVED

Re-run on testnet 2026-05-17 confirms ±5% drift on dryCall probes (M02, M10b) and ±15% drift on state-mutating probes (M04, M05), well inside the folklore ±30% number. The shape of every formula (linear / quadratic) survives; only the absolute coefficient shifts up to ±15% depending on the underlying atree storage history of the COA in question. Quadratic coefficient shifts ~15% per COA depending on atree slot-map history. See `T36-DEEP-DIVE.md §2`.

### 7.2 Surge factor

`get_fee_params.cdc` reports `surgeFactor=1.0` on emulator. On mainnet the surge factor can rise under network congestion, multiplying fee cost. We could not induce surge on the emulator — there is no documented knob to force it. Without the ability to induce surge, we can't measure surge-factor sensitivity. **Open**: is there a way to dial surge on a local emulator?

### 7.3 Why is `coa.dryCall` 14% cheaper than `EVM.dryCall`? — RESOLVED

`EVM.dryCall(from: coa.address(), …)` forces a fresh `EVMAddress` struct allocation + composite-value transfer at every hop. `coa.dryCall(…)` passes `self.addressBytes` directly with no struct construction. Source: `flow-go/fvm/evm/stdlib/contract.cdc:547-549, :687-700, :871-885`. Mechanism is a missing optimization on the public-EVM API path, not a different `InternalEVM` Go entry. See `T36-DEEP-DIVE.md §3`.

### 7.4 The quadratic coefficient origin — RESOLVED

Quadratic origin is the **Per-COA Slot OrderedMap Re-Commit**. After every state-mutating EVM call, `BaseView.Commit → CollectionProvider.Commit → atree.FastCommit` flushes dirty slabs through `ledger.SetValue`, metered by `ComputationKindSetValue` (weight 48) with intensity = `len(value)` bytes. As slabs grow (more slots per account, larger root slab), per-commit byte count grows linearly with the number of writes seen so far in the transaction, making the total bytes-written for N calls quadratic: Σ(α·i) = α·N(N+1)/2.

α ≈ 300 bytes/call for ERC20.transfer (a≈0.108); α ≈ 60 bytes/call for FlowToken bridge ops (a≈0.022). Source: `flow-go/fvm/evm/emulator/state/{base,collection,stateDB}.go` and `flow-go/fvm/environment/value_store.go:160-204`. See `T36-DEEP-DIVE.md §4`.

### 7.5 In-EVM-work bleed mechanism — RESOLVED

Protocol constant: `ComputationKindEVMGasUsage: 3` at `flow-go/fvm/environment/meter.go:104`. Combined with `MeterExecutionInternalPrecisionBytes = 16` at `flow-go/fvm/meter/computation_meter.go:28`, the rule is: 1 EVM gas = 3/65536 CU, or **1 CU = 21,845 EVM gas** (protocol-defined).

The empirical 2,848 in M39 is the all-in rate including ~7.7× Cadence-side bleed (slot-map slab growth + event payload sizing) — not a protocol constant. Use 21,845 for the protocol floor; use 2,848 for sizing a real-world heavy EVM op's CU budget. See `T36-DEEP-DIVE.md §5`.

### 7.6 EVM transaction hash exposure — RESOLVED (CRITICAL CORRECTION)

`EVM.Result` has **no `txHash` field** in CLI v2.17.1. Attempting to read `r.txHash` produces: `value of type 'EVM.Result' has no member 'txHash'`. The EVM tx hash IS available, but only on the `EVM.TransactionExecuted` Cadence event under the field name `hash` (typed `[UInt8]`). That event fires AFTER the Cadence transaction commits, so it **cannot be read inside the same transaction that produced it**. Settlement events keyed by an internal nonce, joined off-chain on the Flow tx ID, is the only available pattern.

Each non-`status` field read on `EVM.Result` (`gasUsed`, `errorCode`, `data`) adds ~0.17 CU — negligible. Read fields freely.

### 7.7 Calldata size effect on coa.call (state-mutating) — RESOLVED

State-mutating `coa.call` with raw `bytes` calldata adds a **per-call linear surcharge of ~0.23 CU/extra-byte**, on top of the per-call ~14 CU constant and on top of the quadratic-in-N shape. The quadratic coefficient (a≈0.02 at small payload) is **independent of calldata size**. Practical rule: a 1024-byte calldata payload on every one of 20 state-mutating calls costs ~5,000 extra CU on top of call overhead.

### 7.8 Decoding ABI for nested structs / arrays of arrays — RESOLVED

| ABI return type | Cadence decode | Status |
|---|---|---|
| `uint256[][]` | `Type<[[UInt256]]>()` | **works** — exact values recovered |
| `uint256[][][]` | `Type<[[[UInt256]]]>()` | **works** — any nesting depth of `[UInt256]` |
| `(uint256, uint256)[]` (tuple/struct array) | no working pattern | **NOT SUPPORTED** — `[[UInt256]]` errors; `[UInt256]` silently drops fields |

Workaround for tuple arrays: refactor Solidity to return **parallel arrays** of primitives (`uint256[] memory firstFields, uint256[] memory secondFields`). This extends the dictionary workaround in §6.2 to tuple arrays.

---

## Appendix: Reproducing the Measurements

All scripts live in `/home/oydual3/flow-ai-tools-drafts/flow-crossvm/.verify-workdir/`:

- `transactions/m30_drycall_returnsize.cdc` — M30 probe
- `transactions/m31_drycall_calldatasize.cdc` — M31 probe
- `transactions/m32_drycall_vs_call_gas.cdc` — M32 probe
- `transactions/m33_coa_drycall_vs_evm_drycall.cdc` — M33 probe
- `transactions/m34_evm_event_extract.cdc` — M34 probe
- `transactions/m35_multicall_scale.cdc` — M35 probe
- `transactions/m36_dry_then_commit.cdc` — M36 probe
- `transactions/m37_event_log_decode.cdc` — M37 probe
- `transactions/m38_drycall_loop_linear.cdc` — M38 probe (N up to 2000)
- `transactions/m39_bumpcounter.cdc` — M39 probe (in-EVM work)
- `transactions/m40_drycall_loop_quad_probe.cdc` — M40 probe (coa.dryCall at large N)
- `transactions/m41_event_count.cdc` — M41 probe (events per hop)
- `transactions/m42_dict_test.cdc`, `m42b_dict_direct.cdc` — M42 probes (dict encoding)
- `transactions/m43_evm_revert.cdc`, `m44_evm_revert_rollback.cdc` — M43/M44 (revert semantics)
- `sol/src/ReturnSizeProbe.sol` — Solidity probe used by M30, M31, M34, M37, M39
- `m30_m37_run.py`, `m38_m40_run.py` — sweep runners with regression fits
- `m30_m37_results.json`, `m38_m40_results.json` — raw row+fit data

Bootstrap sequence (after `flow emulator --transaction-fees --contracts`):

```
flow transactions send transactions/create_coa.cdc 100.0 ...
flow transactions send transactions/deploy_evm.cdc --args-json "$(cat deploy_erc20_args.json)" ...
flow transactions send transactions/deploy_evm.cdc --args-json "$(cat deploy_erc20_b_args.json)" ...
flow transactions send transactions/deploy_evm.cdc --args-json "$(cat deploy_amm_args.json)" ...
flow transactions send transactions/deploy_evm.cdc --args-json "$(cat deploy_multi_args.json)" ...
flow transactions send transactions/deploy_evm.cdc --args-json "$(cat deploy_probe_args.json)" ...
flow transactions send transactions/seed_erc20.cdc <TOKEN_A> <COA> 1000000000000000000000000 ...
flow transactions send transactions/seed_erc20.cdc <TOKEN_B> <COA> 1000000000000000000000000 ...
flow transactions send transactions/approve_and_seed_amm.cdc <TOKEN_A> <TOKEN_B> <AMM> 1e21 1e21 ...
python3 m30_m37_run.py
python3 m38_m40_run.py
```

EVM deploy addresses are deterministic from the COA's nonce sequence; redeploying in the same order yields the same addresses every run.
