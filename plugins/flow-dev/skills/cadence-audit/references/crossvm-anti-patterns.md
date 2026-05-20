# CrossVM Anti-Patterns

CrossVM audits sit on a different failure surface than ordinary Cadence audits because the transaction spans two execution environments with independent budgets, independent error models, and asymmetric atomicity. The 9,999 CU per-tx ceiling and per-`coa.call` EVM `gasLimit` are separate budgets; a Cadence panic reverts everything but a `coa.call` revert reverts only the EVM side and leaves Cadence to commit whatever happened before and after. EVM return bytes are attacker-controllable input, not a typed value, and a misplaced `auth(EVM.Call)` capability is bearer drain authority over an EVM address. This reference enumerates the ten CrossVM-specific anti-patterns auditors must check. Cross-reference [evm-call.md](evm-call.md) (canonical `coa.call` pattern), [coa-lifecycle.md](coa-lifecycle.md) (COA custody and entitlements), [cu-ceiling.md](cu-ceiling.md) (9,999 CU shared budget), and [flow-bridge.md](flow-bridge.md) (native FLOW bridge atomicity).

---

## C1 — Ignoring `result.status` on `coa.call` (Critical)

`coa.call` returns; the implementer assumes "no panic = success" and moves on. Cadence does **not** auto-revert when an EVM call fails — it sets `result.status = EVM.Status.failed` and continues. Every state change the Cadence side makes before and after the failed call commits while the EVM side rolled back. The single most common CrossVM bug in production.

### Bad

```cadence
import "EVM"

transaction(target: String, calldata: [UInt8], amount: UInt256) {
    prepare(signer: auth(BorrowValue) &Account) {
        let coa = signer.storage.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(
            from: /storage/evm) ?? panic("no COA")
        // ❌ result discarded — silent EVM failure
        coa.call(to: EVM.addressFromString(target), data: calldata,
                 gasLimit: 200_000, value: EVM.Balance(attoflow: 0))
        // Captured-but-not-inspected is the same bug: let _ = coa.call(...)
        OrderBook.markFilled(amount: amount)   // commits regardless of EVM outcome
    }
}
```

### Why it's bad

`OrderBook.markFilled` runs and commits; the user is debited / credited on the Cadence side, but the EVM swap that was supposed to provide matching liquidity reverted with its state intact at the pre-call snapshot. The protocol has now invented value. Worse, if the failure was caused by attacker-controlled inputs (slippage griefing, MEV reorder, intentional revert in a malicious target), the attacker gets to choose when to make this happen.

### Correct

```cadence
let result = coa.call(to: EVM.addressFromString(target), data: calldata,
                      gasLimit: 200_000, value: EVM.Balance(attoflow: 0))
if result.status != EVM.Status.successful {
    panic("EVM call failed [code=".concat(result.errorCode.toString())
        .concat("] ").concat(result.errorMessage))
}
OrderBook.markFilled(amount: amount)
```

### Detection hint

Grep every `.call(` and `EVM.dryCall(` / `.dryCall(` site. For each, verify the following statements include `result.status == EVM.Status.successful` (or `!= ...successful`) with a `panic` / `assert` branch. Captured-but-unread `let _ =` or `let result =` with no subsequent reference to `result.status` is the smell. Acceptable forms: `assert(result.status == EVM.Status.successful, ...)` or `if result.status != EVM.Status.successful { panic(...) }`.

See also [evm-call.md](../flow-crossvm/references/evm-call.md) Pitfall 1 for the canonical implementation pattern.

---

## C2 — Looping `coa.call` without measuring CU (High)

A transaction loops `coa.call` over an input array without measuring CU. The 9,999 CU per-tx ceiling is hit silently — the transaction fails partway through and refunds neither the deposited FLOW nor the work already done.

### Bad

```cadence
import "EVM"

transaction(recipients: [EVM.EVMAddress], amounts: [UInt256], tokenHex: String) {
    prepare(signer: auth(BorrowValue) &Account) {
        let coa = signer.storage.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(
            from: /storage/evm) ?? panic("no COA")
        let token = EVM.addressFromString(tokenHex)
        var i = 0
        // ❌ unbounded loop; no MAX_SAFE_N derived from a CU sweep
        while i < recipients.length {
            let data = EVM.encodeABIWithSignature(
                "transfer(address,uint256)", [recipients[i], amounts[i]])
            let r = coa.call(to: token, data: data, gasLimit: 100_000,
                             value: EVM.Balance(attoflow: 0))
            assert(r.status == EVM.Status.successful, message: "transfer failed")
            i = i + 1
        }
    }
}
```

### Why it's bad

Each iteration consumes `[UNVERIFIED: ~200–500 CU]` for the `coa.call` plus encode/decode overhead. The loop dies around `N=20–40` recipients with `computation limited exceeded`. Because the loop is mid-flight when CU runs out, intermediate `Transfer` events were rolled back along with everything else — but the user sees only the failed top-level transaction with no signal about which iteration was the cliff. Re-running with smaller N is the only path forward.

### Correct

```cadence
// ✅ push the loop into Solidity; one coa.call, EVM gas scales with N
let data = EVM.encodeABIWithSignature(
    "batchTransfer(address,address[],uint256[])", [token, recipients, amounts])
let r = coa.call(to: batcher, data: data,
                 gasLimit: 3_000_000, value: EVM.Balance(attoflow: 0))
assert(r.status == EVM.Status.successful, message: "batch failed: ".concat(r.errorMessage))
```

If a Solidity batcher is not available, use `coa.dryCall` to estimate per-iteration CU, set a hardcoded `MAX_SAFE_N` from a sweep (see [cu-ceiling.md](cu-ceiling.md) § "Measurement Methodology"), and reject the transaction with a `pre` condition when `recipients.length > MAX_SAFE_N`. Chunk remaining work across transactions using the multi-tx escrow phase machine.

### Detection hint

Grep `while`, `for ... in`, `recipients.length` patterns wrapping any `.call(` / `.dryCall(`. For each, verify (a) the upper bound on the iteration count is a constant `MAX_SAFE_N` derived from measurement, and (b) a `pre` condition or guard enforces that bound. An unbounded loop with no ceiling assertion is the smell.

---

## C3 — Assuming atomicity across Cadence + bridge + EVM call (Critical)

Mental model: "I'm in one Cadence transaction; everything inside it commits or reverts together." False for the EVM side. A sequence — withdraw FLOW from Cadence vault → `coa.deposit` → `coa.call` → read result — is only atomic *if* `result.status` is checked. Otherwise the deposit commits, the EVM call silently reverts, and the FLOW is stranded in the COA.

### Bad

```cadence
import "EVM"
import "FungibleToken"
import "FlowToken"

transaction(amount: UFix64, swapTarget: String, swapData: [UInt8]) {
    prepare(signer: auth(BorrowValue) &Account) {
        let coa = signer.storage.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(
            from: /storage/evm) ?? panic("no COA")
        let vRef = signer.storage.borrow<auth(FungibleToken.Withdraw) &FlowToken.Vault>(
            from: /storage/flowTokenVault) ?? panic("no vault")
        let chunk <- vRef.withdraw(amount: amount) as! @FlowToken.Vault
        coa.deposit(from: <-chunk)                            // FLOW now in COA
        let value = EVM.Balance(attoflow: 0)
        value.setFLOW(flow: amount)
        // ❌ no status check; missing EVM.Withdraw entitlement; FLOW gets stuck
        let _ = coa.call(to: EVM.addressFromString(swapTarget), data: swapData,
                         gasLimit: 500_000, value: value)
    }
}
```

### Why it's bad

`coa.deposit` is unconditional. The `coa.call` returns `EVM.Status.failed` and Cadence keeps running. The transaction commits, the user's Cadence-side `FlowToken.Vault` is debited, and the FLOW is locked inside the COA's EVM balance. If the signer borrowed without `EVM.Withdraw`, even a follow-up transaction cannot recover without separate setup. Funds are not burned but stuck behind a permissions trap.

### Correct

```cadence
// ✅ borrow with EVM.Withdraw + panic on EVM failure
let coa = signer.storage.borrow<auth(EVM.Call, EVM.Withdraw) &EVM.CadenceOwnedAccount>(
    from: /storage/evm) ?? panic("no COA")
// ... deposit + setFLOW as above ...
let result = coa.call(to: EVM.addressFromString(swapTarget), data: swapData,
                      gasLimit: 500_000, value: value)
if result.status != EVM.Status.successful {
    // Panic reverts everything: deposit, vault withdraw, the whole tx.
    panic("EVM swap failed [".concat(result.errorCode.toString()).concat("] ")
        .concat(result.errorMessage))
}
```

### Detection hint

For every transaction containing `coa.deposit` followed by `coa.call` in the same block, trace forward from each `coa.call` and verify a panic-on-status-failure exists before the transaction ends. Absence is the smell. Also verify the COA was borrowed with `EVM.Withdraw` in addition to `EVM.Call` whenever value is forwarded as `EVM.Balance` — without it the recovery transaction has no path back into Cadence.

---

## C4 — Sharing COA auth capabilities across consumers (Critical)

> See canonical treatment in [../flow-crossvm/references/coa-entitlements.md](../flow-crossvm/references/coa-entitlements.md) Anti-pattern B.
> This entry is a context-specific summary; updates to the underlying behavior should land in the canonical file first.

A protocol issues one `auth(EVM.Call) &EVM.CadenceOwnedAccount` capability and hands the same `Capability` value to multiple consumers — an escrow, a router, a frontend helper, a keeper bot. Capabilities are bearer authority. There is no per-consumer revocation when one cap is shared.

### Bad

```cadence
import "EVM"

transaction(escrowAddr: Address, routerAddr: Address, keeperAddr: Address) {
    prepare(signer: auth(IssueStorageCapabilityController, PublishInboxCapability) &Account) {
        let cap = signer.capabilities.storage
            .issue<auth(EVM.Call) &EVM.CadenceOwnedAccount>(/storage/evm)
        // ❌ same cap to three consumers — one bug in any one drains the COA
        signer.inbox.publish(cap, name: "escrowCOA", recipient: escrowAddr)
        signer.inbox.publish(cap, name: "routerCOA", recipient: routerAddr)
        signer.inbox.publish(cap, name: "keeperCOA", recipient: keeperAddr)
    }
}
```

### Why it's bad

Capabilities are not view-only proofs of ownership. Each holder can issue arbitrary `coa.call` invocations: ERC20 `approve` to drain spend allowances, `transfer` of every ERC20 the COA holds, swaps that route through attacker pools. Revoking the capability deletes it for all three consumers simultaneously — there is no path to disable the keeper while keeping the escrow running. A single audit lapse on any consumer is a full COA compromise.

### Correct

```cadence
// ✅ one cap per consumer, narrow entitlements, separate controller IDs
let storage = signer.capabilities.storage
let escrowCap = storage.issue<auth(EVM.Call, EVM.Withdraw) &EVM.CadenceOwnedAccount>(
    /storage/evm)                                              // call + withdraw
let routerCap = storage.issue<auth(EVM.Call) &EVM.CadenceOwnedAccount>(/storage/evm)
let keeperCap = storage.issue<auth(EVM.Call) &EVM.CadenceOwnedAccount>(/storage/evm)
signer.inbox.publish(escrowCap, name: "escrowCOA", recipient: escrowAddr)
signer.inbox.publish(routerCap, name: "routerCOA", recipient: routerAddr)
signer.inbox.publish(keeperCap, name: "keeperCOA", recipient: keeperAddr)
// Deleting only keeperCap's controller leaves escrow and router intact.
```

### Detection hint

Grep `capabilities.storage.issue<auth(EVM.` patterns in setup transactions. For each issued capability, count distinct `publish` / `inbox.publish` / function-argument sites the returned value is passed to. Count > 1 with the same controller is the smell. Also flag any `access(all) let` field of type `Capability<auth(EVM.Call) &EVM.CadenceOwnedAccount>` on a shared resource.

---

## C5 — Publishing the COA auth capability at `/public/evm` (Critical)

> See canonical treatment in [../flow-crossvm/references/coa-entitlements.md](../flow-crossvm/references/coa-entitlements.md) Anti-pattern C.
> This entry is a context-specific summary; updates to the underlying behavior should land in the canonical file first.

The canonical convention: `/storage/evm` holds the COA resource, `/public/evm` holds an un-entitled `&EVM.CadenceOwnedAccount` capability (reads + deposits, no call/withdraw/deploy). A setup script that copies an old example can `publish` an `auth(EVM.Call)` capability at `/public/evm` — at which point any account on the network can borrow it and drain the COA.

### Bad

```cadence
import "EVM"

transaction() {
    prepare(signer: auth(SaveValue, IssueStorageCapabilityController,
                          PublishCapability, UnpublishCapability) &Account) {
        if signer.storage.type(at: /storage/evm) == nil {
            let coa <- EVM.createCadenceOwnedAccount()
            signer.storage.save(<-coa, to: /storage/evm)
        }
        signer.capabilities.unpublish(/public/evm)
        // ❌ auth(EVM.Call) at /public/evm — any account can drain
        let cap = signer.capabilities.storage
            .issue<auth(EVM.Call) &EVM.CadenceOwnedAccount>(/storage/evm)
        signer.capabilities.publish(cap, at: /public/evm)
    }
}
```

### Why it's bad

`/public/*` capabilities are resolvable by any Cadence script or transaction via `getAccount(addr).capabilities.borrow<...>(...)`. The publisher fixed the type to `auth(EVM.Call)` — borrowing returns the auth-entitled reference. From any attacker transaction: `getAccount(victim).capabilities.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(/public/evm)!.call(to: drainTarget, ...)` sends every drainable ERC20 to the attacker. The COA is bearer-controllable for the duration the publication exists.

### Correct

```cadence
// ✅ un-entitled cap at /public/evm; auth caps stay in /storage and inbox
let publicCap = signer.capabilities.storage
    .issue<&EVM.CadenceOwnedAccount>(/storage/evm)
signer.capabilities.publish(publicCap, at: /public/evm)
```

If a third party needs `EVM.Call` authority, issue a separate narrow cap and hand it through `inbox.publish(..., recipient: namedAccount)` to a single named recipient — never via `/public/*`.

### Detection hint

Grep `publish(.* at: /public/evm)` and `capabilities.publish(.* /public/evm)`. For each, verify the capability type is exactly `&EVM.CadenceOwnedAccount` (no `auth(...)` prefix). Any `auth(EVM.Call)`, `auth(EVM.Withdraw)`, `auth(EVM.Deploy)`, `auth(EVM.Owner)` at `/public/evm` is critical-severity. Also grep `publish(.* /public/.*)` more broadly — any capability with an `auth(EVM.*)` prefix at any `/public/*` path is the same bug.

---

## C6 — Misusing UFix64 for attoflow conversion (Medium)

Hand-rolling the conversion between Cadence FLOW (`UFix64`, 8 decimals) and EVM attoflow (`UInt`, 18 decimals) is brittle. `UFix64 * 1_000_000_000_000_000_000` overflows the type; `UInt(flow * 1e18)` does not compile because UFix64 cannot represent `1e18`. The contract exposes `EVM.Balance(attoflow:).setFLOW(...)` to handle this — bypassing it invites silent precision bugs.

### Bad

```cadence
import "EVM"

let amount: UFix64 = 1.5
// ❌ compile-time failure: 1e18 does not fit UFix64
let atto: UInt = UInt(amount * 1_000_000_000_000_000_000.0)
let bad = EVM.Balance(attoflow: atto)

// ❌ also fragile — manual factor-of-10^10 idiom that breaks under refactor
let manualAtto: UInt = UInt(amount * 100_000_000.0) * 10_000_000_000
```

### Why it's bad

UFix64's max value is roughly `184e9` FLOW — multiplying any non-trivial amount by `1e18` overflows on the literal alone, so the code does not compile. Even with the `10^8 × 10^10` decomposition, the manual idiom is copy-pasted across the codebase and one stale copy is the bug. The contract-provided `setFLOW(flow:)` does the conversion in one place that the runtime maintains.

### Correct

```cadence
// ✅ canonical conversion: UFix64 → attoflow
let amount: UFix64 = 1.5
let balance = EVM.Balance(attoflow: 0)
balance.setFLOW(flow: amount)

// ✅ construct directly from attoflow for EVM-native amounts
let bigBalance = EVM.Balance(attoflow: 1_500_000_000_000_000_000)   // 1.5 FLOW
```

For the rare case where the source is a high-precision EVM-side value being converted into UFix64, use `balance.inFLOW()` and accept the documented sub-10nF dust loss — see [flow-bridge.md](flow-bridge.md) § "Decimal Reconciliation".

### Detection hint

Grep for the literals `1e18`, `1_000_000_000_000_000_000`, `10_000_000_000_000_000_000`, and `UFix64.*\*.*1e` patterns. For each, verify the surrounding code uses `EVM.Balance(attoflow: ...)` directly with an integer literal or `balance.setFLOW(flow: ...)`. Hand-multiplied conversions are the smell. Also flag `UInt(someUFix64 * ...)` patterns near EVM call sites.

---

## C7 — Trusting EVM-side return values without validating ranges (High)

A `coa.call` to an EVM contract returns `result.data: [UInt8]`. The Cadence side decodes those bytes with `EVM.decodeABI` and uses the values directly — as a price, balance, or authorisation token. The EVM contract is opaque to Cadence's type system: its return bytes are attacker-controllable input.

### Bad

```cadence
import "EVM"

transaction(oracleHex: String) {
    prepare(signer: auth(BorrowValue) &Account) {
        let coa = signer.storage.borrow<&EVM.CadenceOwnedAccount>(from: /storage/evm)
            ?? panic("no COA")
        let priceSel: [UInt8] = [0x98, 0xd5, 0xfd, 0xca]    // getPrice()
        let r = coa.dryCall(to: EVM.addressFromString(oracleHex), data: priceSel,
                            gasLimit: 50_000, value: EVM.Balance(attoflow: 0))
        assert(r.status == EVM.Status.successful, message: "oracle call failed")
        let decoded = EVM.decodeABI(types: [Type<UInt256>()], data: r.data)
        let price = decoded[0] as! UInt256
        // ❌ used directly — malicious oracle returns 2^255 → downstream wraparound
        Treasury.markToMarket(price: price)
    }
}
```

### Why it's bad

The EVM contract at `oracleHex` is not statically known to be the intended oracle. Even if the address is correct, a malicious contract owner can upgrade it (via UUPS / transparent proxy) or pre-set return values for specific callers. Without range checking, the Cadence treasury accepts any number from `0` to `2^256 - 1`, including overflow-inducing values that break downstream arithmetic. The bug surface is the union of every Solidity exploit and every Cadence numeric-handling bug.

### Correct

```cadence
// ✅ range-check after decode; bounds are protocol invariants, not oracle hints
let minPrice: UInt256 = 1
let maxPrice: UInt256 = 1_000_000_000_000_000_000_000_000  // 1e24 sanity cap
assert(price >= minPrice && price <= maxPrice,
       message: "oracle returned out-of-range price")
Treasury.markToMarket(price: price)
```

For higher-stakes flows (collateral pricing, liquidation thresholds), enforce a TWAP, require two independent oracles, or require the value to be signed by a known keeper key before trusting it.

### Detection hint

Grep `decodeABI(types:` followed by `as!` casts and immediate field assignment / function-argument use. For each, verify a range check or invariant assertion exists between the decode and the consuming statement. Direct use of decoded values in arithmetic, comparisons against collateral limits, or as authorisation flags without validation is the smell. Pay special attention to `Bool` returns — a malicious contract returning `true` for a permission check must not be sufficient evidence of authorisation.

---

## C8 — Re-entrancy via EVM callback into Cadence (Critical)

A Cadence transaction calls EVM contract X via `coa.call`. X internally calls back into a Cadence-side capability (via a bridge handler, an inbox-published cap, or an off-chain relayer that submits a new tx). Cadence's resource semantics protect against re-entry on the **same borrow scope** but the EVM-side call is a fresh execution frame, and if it re-enters a different borrow of the same resource, Cadence's protection does not catch it.

### Bad

```cadence
import "EVM"

access(all) resource VaultManager {
    access(self) var totalDeposited: UFix64
    access(self) let coa: Capability<auth(EVM.Call) &EVM.CadenceOwnedAccount>
    access(self) let strategyAddr: EVM.EVMAddress

    access(all) fun depositAndNotify(amount: UFix64) {
        let coaRef = self.coa.borrow() ?? panic("coa cap")
        let calldata = EVM.encodeABIWithSignature(
            "onDeposit(uint256)", [UInt256(amount * 100_000_000.0)])
        // ❌ EVM strategy callback observes totalDeposited BEFORE we update it
        let r = coaRef.call(to: self.strategyAddr, data: calldata,
                            gasLimit: 500_000, value: EVM.Balance(attoflow: 0))
        assert(r.status == EVM.Status.successful, message: "strategy notify failed")
        self.totalDeposited = self.totalDeposited + amount       // updated AFTER
    }
}
```

### Why it's bad

If the EVM strategy holds a Cadence capability — published via a bridge handler, inbox, or any cross-VM messaging primitive — its callback into Cadence sees `totalDeposited` at its pre-update value. The callback can withdraw based on stale invariants, mint extra shares, or re-enter `depositAndNotify`. Critically, this is **not** classic Solidity re-entrancy: Cadence-to-EVM within a single tx is atomic at the FVM layer, but if the EVM call triggers an off-chain relayer that submits a **new Flow transaction** before the original tx seals, that new tx is genuinely concurrent and per-tx invariants do not help.

### Correct

```cadence
// ✅ checks-effects-interactions: update Cadence state BEFORE EVM call
access(all) fun depositAndNotify(amount: UFix64) {
    self.totalDeposited = self.totalDeposited + amount         // effects first
    let coaRef = self.coa.borrow() ?? panic("coa cap")
    let calldata = EVM.encodeABIWithSignature(
        "onDeposit(uint256)", [UInt256(amount * 100_000_000.0)])
    let r = coaRef.call(to: self.strategyAddr, data: calldata,
                        gasLimit: 500_000, value: EVM.Balance(attoflow: 0))
    if r.status != EVM.Status.successful {
        panic("strategy notify failed: ".concat(r.errorMessage))
    }
}
```

For relayed off-chain callback paths, add an explicit re-entry guard (a `Bool` flag set before the EVM call, cleared after, asserted false at the entry of any callback handler). For high-stakes flows, gate cross-VM callbacks behind a phase machine — see [coa-lifecycle.md](coa-lifecycle.md) § "Custody pattern".

### Detection hint

For every method that performs `coa.call` followed by state mutation, flag the order. State should be updated before the EVM call, not after. Also identify every Cadence capability the protocol exposes to EVM-side callers (bridge handlers, `inbox.publish` to EVM-derived addresses, fixed-address callbacks). For each, ask: "can this be re-entered mid-`coa.call`?" If yes, an explicit re-entry guard or phase check is required.

---

## C9 — No timeout or fallback on stuck CrossVM operations (Medium)

A multi-step CrossVM flow — deposit FLOW → call EVM strategy → wait for callback → withdraw result — gets half-done. The strategy returns a non-standard error, the callback never fires, the relayer goes down, or a network upgrade changes EVM behaviour. Funds end up in the COA with no on-chain recovery path.

### Bad

```cadence
import "EVM"

access(all) resource Position {
    access(self) let coa: Capability<auth(EVM.Call) &EVM.CadenceOwnedAccount>
    access(self) var status: String         // "open" | "closed"

    access(all) fun openAndStake(strategy: EVM.EVMAddress, amount: UInt256) {
        let coaRef = self.coa.borrow() ?? panic("coa cap")
        let calldata = EVM.encodeABIWithSignature("stake(uint256)", [amount])
        let r = coaRef.call(to: strategy, data: calldata,
                            gasLimit: 500_000, value: EVM.Balance(attoflow: 0))
        assert(r.status == EVM.Status.successful, message: "stake failed")
        self.status = "open"
        // ❌ no close() / reclaim() — if strategy.unstake() reverts forever, stuck
    }
}
```

### Why it's bad

If `strategy.unstake()` reverts permanently (paused, migrated, malicious upgrade), funds are locked inside the EVM strategy. The Cadence-side `Position` resource has no escape hatch: no admin sweep the user can call after a deadline, no time-based force-close that yields whatever the COA holds back to the depositor. The position is permanent.

### Correct

```cadence
// ✅ explicit time-based recovery
access(all) entitlement Reclaim
access(self) let coa: Capability<auth(EVM.Call, EVM.Withdraw) &EVM.CadenceOwnedAccount>
access(self) let deadline: UFix64

access(Reclaim) fun reclaim(): @{FungibleToken.Vault} {
    pre {
        getCurrentBlock().timestamp > self.deadline: "deadline not passed"
        self.status != "reclaimed": "already reclaimed"
    }
    let coaRef = self.coa.borrow() ?? panic("coa cap")
    self.status = "reclaimed"
    return <- coaRef.withdraw(balance: coaRef.balance())
}
```

Pair this with an observable `event Reclaimed(positionId: UInt64, attoflow: UInt)` so indexers can flag the rescue path. For protocols where time-based reclaim is too weak, add a parallel admin sweep gated by governance. See [coa-lifecycle.md](coa-lifecycle.md) § "Custody pattern" for the phase-machine variant.

### Detection hint

For every Cadence resource that holds a `Capability<auth(EVM.Call) &EVM.CadenceOwnedAccount>` or directly performs `coa.call`, ask: "if the EVM-side counterparty becomes permanently unavailable, can the user / admin still recover the COA balance?" If the only recovery is a contract upgrade, that is the smell. Look for explicit `deadline`, `expireAt`, or `reclaim` methods; absence indicates a missing recovery path.

---

## C10 — Storing EVM addresses as `String` (Low)

EVM addresses are 20 raw bytes wrapped in `EVM.EVMAddress` for type safety and automatic length validation. Storing them as `String` means every read site re-parses via `EVM.addressFromString` and every comparison is a string comparison that ignores EIP-55 checksum capitalisation.

### Bad

```cadence
import "EVM"

access(all) contract TokenRegistry {
    // ❌ String comparison — case-sensitive, doesn't normalise EIP-55 checksum
    access(all) let usdcAddress: String

    init() {
        self.usdcAddress = "0xA0b86a33E6441e0C8DCDCDef1d8aD2c4Df90F4Df9"
    }
    access(all) fun isUSDC(addr: String): Bool {
        return addr == self.usdcAddress
    }
}
```

### Why it's bad

`addr == self.usdcAddress` does literal string comparison. `0xa0b86a33...` and `0xA0b86a33...` are different strings even though they refer to the same EVM address. EIP-55 checksum encoding makes mismatch the *expected* case when one side is hand-typed and the other comes from a wallet, indexer, or RPC. Bugs surface as "the protocol thinks USDC is not USDC" — wrong from the user's perspective and exploitable if the lookup gates authorisation. Every read site also pays the `EVM.addressFromString` parse cost again, and malformed inputs panic at parse time rather than at construction time.

### Correct

```cadence
// ✅ EVM.EVMAddress — raw-byte equality, validated at construction
access(all) let usdcAddress: EVM.EVMAddress

init() {
    self.usdcAddress = EVM.addressFromString(
        "0xA0b86a33E6441e0C8DCDCDef1d8aD2c4Df90F4Df9")
}
access(all) fun isUSDC(addr: EVM.EVMAddress): Bool {
    return addr == self.usdcAddress
}
```

`EVM.addressFromString` accepts both `0x`-prefixed and unprefixed hex and validates 20-byte length — invalid inputs panic at construction. Storing as `EVM.EVMAddress` unlocks direct use in `coa.call(to: ...)` and in `EVM.encodeABI([addr])` without per-call reparsing.

### Detection hint

Grep for `let .*: String` fields whose initialiser is a hex literal starting with `0x` of length 40 or 42, and field names containing `address`, `addr`, `token`, `pool`, `router`, `vault`, `oracle`, `target`. For each, verify the field type is `EVM.EVMAddress` not `String`. Also flag function parameters typed `String` that are clearly intended to be EVM addresses (parameter name like `targetAddr: String`).

---

## Audit checklist for CrossVM code

For any transaction, script, or contract that touches `EVM`, `coa.call`, `coa.deposit`, `coa.withdraw`, or holds a `Capability<auth(EVM.*) &EVM.CadenceOwnedAccount>`, the auditor must answer **yes** to every question — or document a deliberate exception.

### Result checking
- [ ] Does every `coa.call` invocation explicitly check `result.status == EVM.Status.successful` and panic / assert / revert on any non-success status?
- [ ] Does every `EVM.dryCall` and `coa.dryCall` whose return value is consumed downstream check `result.status` before decoding `result.data`?
- [ ] When decoding `result.data`, are the resulting values range-checked or invariant-checked before being used in arithmetic, comparisons, or authorisation logic?
- [ ] Is `EVM.Status.invalid` distinguished from `EVM.Status.failed` (one is a Cadence-side encoding bug, the other is an EVM-side revert) and handled appropriately?

### Budget and CU
- [ ] For every loop containing `coa.call` / `coa.dryCall` / `EVM.dryCall` / `coa.deposit` / `coa.withdraw`, is there a hardcoded `MAX_SAFE_N` bound enforced via `pre` condition or guard?
- [ ] Has `MAX_SAFE_N` been derived from a CU sweep (see [cu-ceiling.md](cu-ceiling.md) § "Measurement Methodology") with 10% headroom, not assumed?
- [ ] Are heavy iterations pushed into Solidity via a Multicall / batch contract rather than iterated from Cadence?
- [ ] Is `gasLimit` on each `coa.call` sized for the EVM function's **worst-case** path — including dynamic-array growth and largest-branch execution?

### Capability hygiene
- [ ] Is every `auth(EVM.Call) &EVM.CadenceOwnedAccount` capability issued to exactly one consumer, with a separate capability per consumer (not a shared cap)?
- [ ] Is the entitlement set on each issued capability the **minimum** the consumer needs — `EVM.Call` for call-only consumers, `EVM.Call | EVM.Withdraw` only when recovery is required, never `EVM.Owner` unless full custody is the deliberate goal?
- [ ] Is the `/public/evm` capability strictly un-entitled (`&EVM.CadenceOwnedAccount`, no `auth(...)` prefix)?
- [ ] Is every issued auth capability paired with its controller ID stored alongside, so it can be deleted to revoke?
- [ ] Is the COA resource itself stored at `/storage/evm` (not moved into another resource), so wallet UIs and indexers continue to resolve it?

### Atomicity and recovery
- [ ] When a transaction performs `coa.deposit` followed by `coa.call`, does it panic on EVM failure so the deposit also reverts?
- [ ] When a transaction needs to recover from a partial EVM-side failure, is the COA borrowed with `auth(EVM.Call, EVM.Withdraw)` so recovery is possible without a separate setup transaction?
- [ ] For multi-step CrossVM flows (deposit → call → callback → withdraw), is there an explicit time-based reclaim or admin sweep that returns COA balance to Cadence after a deadline?
- [ ] If the protocol exposes any Cadence capability that could be invoked from an EVM-side callback (bridge handlers, off-chain relayers, fixed-address callbacks), is there an explicit re-entry guard or phase check?
- [ ] Is contract state updated **before** the `coa.call` (checks-effects-interactions order), so any callback observes the post-state?

### Type safety
- [ ] Are EVM addresses stored as `EVM.EVMAddress`, not `String`?
- [ ] Are attoflow ↔ UFix64 conversions performed via `EVM.Balance.setFLOW(flow:)` and `balance.inFLOW()`, not hand-rolled with literals like `1e18`?
- [ ] When `EVM.encodeABI` is used, does the Cadence integer width (`UInt8`, `UInt16`, ..., `UInt256`) match the Solidity parameter width exactly?
- [ ] Is `EVM.encodeABI` only used for argument types it natively supports, and are complex tuples encoded explicitly via concatenation or off-chain pre-packing?

### Observability
- [ ] Are recovery paths (reclaim, sweep, force-close) paired with events that indexers can observe and alert on?
- [ ] Are EVM-side failure modes (status != successful) emitted as Cadence events when the protocol decides to continue rather than panic, so off-chain monitors can detect them?

## See also

- [evm-call.md](evm-call.md) — canonical `coa.call` / `EVM.dryCall` / `coa.dryCall` patterns, `EVM.Status` taxonomy, calldata encoding, gas-limit defaults
- [coa-lifecycle.md](coa-lifecycle.md) — COA creation, entitlements, custody patterns, the three capability anti-patterns from the COA side
- [cu-ceiling.md](cu-ceiling.md) — 9,999 CU shared budget, measurement methodology, batching workarounds, decision matrix Cadence vs EVM
- [flow-bridge.md](flow-bridge.md) — native FLOW bridge (`coa.deposit` / `coa.withdraw`), decimal reconciliation, atomicity guarantees, dust-loss pitfalls
- `cadence-audit/references/audit-checklist.md` — general Cadence audit checklist (apply in addition to this file)
- `cadence-audit/references/forte-anti-patterns.md` — scheduled-transaction anti-patterns that intersect with CrossVM (handler-held COA capabilities)
