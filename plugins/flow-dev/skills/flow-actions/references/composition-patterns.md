# Composition Patterns: Source → Swapper → Sink

DeFiActions compositions are Cadence transactions where a Source, optional Swapper, and Sink are
wired together in a single atomic execution. Because all steps execute inside one Cadence
transaction, the entire pipeline is all-or-nothing — there is no EVM-style async settlement where
hop 1 can commit while hop 2 reverts. A single `UniqueIdentifier` created at the start of the
transaction is threaded through every connector, binding all emitted events into one traceable
operation. Understanding how to wire these three interface types correctly — and in what order —
is the central skill of DeFiActions development.

> **Beta notice:** DeFiActions is in beta. Interface signatures may change before final release.
> Monitor [`onflow/FlowActions`](https://github.com/onflow/FlowActions) for breaking changes.

---

## The Four Transaction Phases

Every DeFiActions composition follows Cadence's transaction phase order:

```
prepare  →  pre  →  execute  →  post
```

| Phase | Responsibility |
|---|---|
| `prepare` | Borrow capabilities, create `UniqueIdentifier`, construct all connectors |
| `pre` | Assert preconditions — single boolean expression, no variable declarations |
| `execute` | Withdraw → (swap) → deposit; assert vault empty before destruction |
| `post` | Assert final state — single boolean expression, no variable declarations |

**Pre and post conditions must be single boolean expressions.** Cadence does not allow variable
declarations inside `pre` or `post` blocks. Read balances or sizes you need for post-assertions
during `prepare` and store them as transaction-level fields.

---

## Pattern 1: Source → Sink (no swap)

**When to use:** The Source and Sink operate on the same vault type. No token conversion is
needed — tokens flow from one position to another unchanged.

```cadence
import "FungibleToken"
import "DeFiActions"
import "FungibleTokenConnectors"

transaction(
    recipientAddress: Address,
    amount: UFix64
) {
    let source: FungibleTokenConnectors.VaultSource
    let sink: FungibleTokenConnectors.VaultSink
    let startingBalance: UFix64
    let availableAtStart: UFix64  // snapshot — see note below

    prepare(acct: auth(BorrowValue, IssueStorageCapabilityController) &Account) {
        let operationID = DeFiActions.createUniqueIdentifier()

        // Issue a withdraw-entitled capability for the signer's FLOW vault
        let withdrawCap = acct.capabilities.storage
            .issue<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>(/storage/flowTokenVault)

        // Resolve the recipient's public deposit capability
        let depositCap = getAccount(recipientAddress)
            .capabilities.get<&{FungibleToken.Vault}>(/public/flowTokenReceiver)
            ?? panic("Recipient has no FlowToken receiver at expected path")

        // Both connectors share the same operationID for end-to-end traceability
        self.source = FungibleTokenConnectors.VaultSource(
            min: nil, withdrawVault: withdrawCap, uniqueID: operationID
        )
        self.sink = FungibleTokenConnectors.VaultSink(
            max: nil, depositVault: depositCap, uniqueID: operationID
        )

        // Capture sender's balance for the post-condition (must happen in prepare)
        self.startingBalance = acct.borrow<&{FungibleToken.Vault}>(from: /storage/flowTokenVault)!
            .balance

        // Snapshot minimumAvailable() in prepare — NOT in pre.
        // FungibleTokenConnectors.VaultSource.minimumAvailable() and
        // FungibleTokenConnectors.VaultSink.minimumCapacity() are NOT declared `view` in
        // FungibleTokenConnectors v1.0.0. Cadence rejects non-view calls inside pre/post blocks
        // with "error: impure operation performed in view context". Always capture these in
        // prepare and reference the field in pre.
        self.availableAtStart = self.source.minimumAvailable()
    }

    // Single boolean expression — references prepare-time snapshot, not a direct call
    pre {
        self.availableAtStart >= amount:
            "Source has insufficient balance: available \(self.availableAtStart), needed \(amount)"
    }

    execute {
        // Size withdrawal by the lesser of requested amount and sink headroom
        let headroom = self.sink.minimumCapacity()
        let maxWithdraw = headroom < amount ? headroom : amount
        let vault <- self.source.withdrawAvailable(maxAmount: maxWithdraw)
        self.sink.depositCapacity(from: &vault as auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
        // Invariant: vault must be fully consumed — any residual is a composition error
        assert(vault.balance == 0.0, message: "Residual balance after deposit: \(vault.balance)")
        destroy vault
    }

    // Single boolean expression — reads startingBalance captured in prepare
    post { self.startingBalance - amount <= self.source.minimumAvailable() + amount:
        "Post-transfer source balance is unexpectedly low" }
}
```

**Type alignment check:** `source.getSourceType()` must equal `sink.getSinkType()`. The Sink
interface pre-condition enforces this at deposit time and will revert with a descriptive message
if types differ. Detect mismatches early by asserting in `prepare` rather than discovering them
at deposit time.

---

## Pattern 2: SwapSource → Sink

**When to use:** The Source produces token0 but the Sink accepts token1. Wrap the Swapper around
the Source so that the combined `SwapSource` presents the output token type to the Sink.

`SwapConnectors.SwapSource` composes a `{DeFiActions.Source}` with a `{DeFiActions.Swapper}`. Its
`getSourceType()` returns `swapper.outType()`, and its `withdrawAvailable()` calls the inner
source's `withdrawAvailable()` then pipes the vault through `swapper.swap()` in one step.

```cadence
import "FungibleToken"
import "DeFiActions"
import "SwapConnectors"
import "FungibleTokenConnectors"
import "MyProtocolSwapConnectors"

transaction(
    poolAddress: Address,
    minExpectedOut: UFix64,
    recipientAddress: Address
) {
    let swapSource: SwapConnectors.SwapSource
    let sink: FungibleTokenConnectors.VaultSink
    let startingRecipientBalance: UFix64
    let swapSourceAvailableAtStart: UFix64  // snapshot — minimumAvailable() is not view

    prepare(acct: auth(BorrowValue, IssueStorageCapabilityController) &Account) {
        let operationID = DeFiActions.createUniqueIdentifier()

        // Source: signer's token0 vault (e.g. FLOW)
        let withdrawCap = acct.capabilities.storage
            .issue<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>(/storage/flowTokenVault)
        let baseSource = FungibleTokenConnectors.VaultSource(
            min: nil, withdrawVault: withdrawCap, uniqueID: operationID
        )

        // Determine pool's canonical token ordering to set Swapper direction correctly
        let pool = getAccount(poolAddress)
            .capabilities.borrow<&{MyProtocol.PoolPublic}>(/public/pool)
            ?? panic("Pool not found at \(poolAddress)")
        let token0Type = pool.token0Type()
        let token1Type = pool.token1Type()

        // Safety rule: source token MUST be inType() — reverse construction if needed
        let sourceType = baseSource.getSourceType()
        let reverse = sourceType != token0Type
        let swapper = MyProtocolSwapConnectors.Swapper(
            inVault:  reverse ? token1Type : token0Type,
            outVault: reverse ? token0Type : token1Type,
            pool:     poolAddress,
            uniqueID: operationID
        )

        // SwapSource pre-condition asserts source.getSourceType() == swapper.inType()
        // This fires at construction time in prepare — fail-fast before any vault moves
        self.swapSource = SwapConnectors.SwapSource(
            swapper: swapper, source: baseSource, uniqueID: operationID
        )

        // Sink: recipient's token1 vault
        let depositCap = getAccount(recipientAddress)
            .capabilities.get<&{FungibleToken.Vault}>(/public/token1Receiver)
            ?? panic("Recipient has no token1 receiver")
        self.sink = FungibleTokenConnectors.VaultSink(
            max: nil, depositVault: depositCap, uniqueID: operationID
        )

        self.startingRecipientBalance = getAccount(recipientAddress)
            .capabilities.borrow<&{FungibleToken.Vault}>(/public/token1Receiver)?.balance ?? 0.0

        // Snapshot: minimumAvailable() is not view — cannot call in pre block
        self.swapSourceAvailableAtStart = self.swapSource.minimumAvailable()
    }

    pre { self.swapSourceAvailableAtStart > 0.0: "SwapSource has nothing to withdraw" }

    execute {
        let headroom = self.sink.minimumCapacity()
        let vault <- self.swapSource.withdrawAvailable(maxAmount: headroom)
        // Slippage guard: assert swap output meets minimum before depositing
        assert(
            vault.balance >= minExpectedOut,
            message: "Swap output \(vault.balance) is below minimum \(minExpectedOut)"
        )
        self.sink.depositCapacity(from: &vault as auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
        assert(vault.balance == 0.0, message: "Residual after deposit: \(vault.balance)")
        destroy vault
    }

    post { self.startingRecipientBalance + minExpectedOut <=
        (getAccount(recipientAddress)
            .capabilities.borrow<&{FungibleToken.Vault}>(/public/token1Receiver)?.balance ?? 0.0):
        "Recipient balance did not increase by expected minimum" }
}
```

---

## Pattern 3: Source → SwapSink

**When to use:** The Source produces token0, but the destination protocol requires token1 to be
deposited. Rather than swapping before the deposit, the `SwapSink` wraps the Swapper around the
Sink: deposits arrive as token0 and the SwapSink converts them to token1 internally before
forwarding to the underlying Sink.

`SwapConnectors.SwapSink` composes a `{DeFiActions.Swapper}` with a `{DeFiActions.Sink}`. Its
`getSinkType()` returns `swapper.inType()` (accepts token0), and its `depositCapacity()` calls
`swapper.swap()` then the inner sink's `depositCapacity()` in one step.

```cadence
import "FungibleToken"
import "DeFiActions"
import "SwapConnectors"
import "FungibleTokenConnectors"
import "MyProtocolSwapConnectors"
import "MyProtocolSinkConnectors"

transaction(
    poolAddress: Address,
    stakingAddress: Address,
    minStakeIncrease: UFix64
) {
    let source: FungibleTokenConnectors.VaultSource
    let swapSink: SwapConnectors.SwapSink
    let startingStake: UFix64
    let sourceAvailableAtStart: UFix64  // snapshot — minimumAvailable() is not view

    prepare(acct: auth(BorrowValue, IssueStorageCapabilityController) &Account) {
        let operationID = DeFiActions.createUniqueIdentifier()

        let withdrawCap = acct.capabilities.storage
            .issue<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>(/storage/flowTokenVault)
        self.source = FungibleTokenConnectors.VaultSource(
            min: nil, withdrawVault: withdrawCap, uniqueID: operationID
        )

        // Inner Sink: accepts token1 (the staking protocol's expected token)
        let innerSink = MyProtocolSinkConnectors.StakeSink(
            staker: acct.address, protocol: stakingAddress, uniqueID: operationID
        )

        // Swapper: converts token0 (FLOW from Source) to token1 (staking token)
        let swapper = MyProtocolSwapConnectors.Swapper(
            inVault:  self.source.getSourceType(),   // token0 — must match source output
            outVault: innerSink.getSinkType(),        // token1 — must match inner sink input
            pool:     poolAddress,
            uniqueID: operationID
        )

        // SwapSink pre-condition asserts swapper.outType() == sink.getSinkType()
        self.swapSink = SwapConnectors.SwapSink(
            swapper: swapper, sink: innerSink, uniqueID: operationID
        )

        self.startingStake = MyProtocol.getStake(address: acct.address)

        // Snapshot: minimumAvailable() is not view — cannot call in pre block
        self.sourceAvailableAtStart = self.source.minimumAvailable()
    }

    pre { self.sourceAvailableAtStart > 0.0: "Source is empty" }

    execute {
        let headroom = self.swapSink.minimumCapacity()
        let vault <- self.source.withdrawAvailable(maxAmount: headroom)
        self.swapSink.depositCapacity(from: &vault as auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
        assert(vault.balance == 0.0, message: "Residual after deposit: \(vault.balance)")
        destroy vault
    }

    post { MyProtocol.getStake(address: self.source.id() ?? 0) >= self.startingStake + minStakeIncrease:
        "Stake did not increase by minimum expected amount" }
}
```

---

## Pattern 4: Source → Swapper → Sink (Direct Chaining)

**When to use:** You need full visibility into each step of the swap — for example, to assert
intermediate vault types, apply a per-step slippage guard, or interpose logging. Instead of using
`SwapSource`/`SwapSink` wrappers, call withdraw, swap, and deposit as discrete steps in
`execute`.

This is the lowest-level composition. It is more verbose but makes every type boundary explicit.

```cadence
import "FungibleToken"
import "DeFiActions"
import "SwapConnectors"
import "FungibleTokenConnectors"
import "MyProtocolSwapConnectors"

/// VaultSource → FixedRateSwapper → VaultSink
/// Canonical protocol-agnostic example using stub connectors from T16-T18.
transaction(
    recipientAddress: Address,
    swapRate: UFix64,
    minExpectedOut: UFix64
) {
    let source: FungibleTokenConnectors.VaultSource
    let swapper: MyProtocolSwapConnectors.FixedRateSwapper
    let sink: FungibleTokenConnectors.VaultSink
    let startingSourceBalance: UFix64
    let sourceAvailableAtStart: UFix64  // snapshot — minimumAvailable() is not view

    prepare(acct: auth(BorrowValue, IssueStorageCapabilityController) &Account) {
        // Single UniqueIdentifier threads all events together
        let operationID = DeFiActions.createUniqueIdentifier()

        // Source: signer's token0 (e.g. FLOW)
        let withdrawCap = acct.capabilities.storage
            .issue<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>(/storage/flowTokenVault)
        self.source = FungibleTokenConnectors.VaultSource(
            min: nil, withdrawVault: withdrawCap, uniqueID: operationID
        )

        // Swapper: token0 → token1 at a fixed rate
        // inType MUST match source.getSourceType() — checked explicitly below
        self.swapper = MyProtocolSwapConnectors.FixedRateSwapper(
            inVault:  self.source.getSourceType(),
            outVault: Type<@MyProtocol.Token1.Vault>(),
            rate:     swapRate,
            uniqueID: operationID
        )
        // Type boundary assertion at construction time — fail fast in prepare
        assert(
            self.source.getSourceType() == self.swapper.inType(),
            message: "Source type \(self.source.getSourceType().identifier) "
                   + "!= Swapper inType \(self.swapper.inType().identifier)"
        )

        // Sink: recipient's token1 vault
        let depositCap = getAccount(recipientAddress)
            .capabilities.get<&{FungibleToken.Vault}>(/public/token1Receiver)
            ?? panic("Recipient has no token1 receiver at expected path")
        self.sink = FungibleTokenConnectors.VaultSink(
            max: nil, depositVault: depositCap, uniqueID: operationID
        )
        // Type boundary assertion — swapper output must match sink input
        assert(
            self.swapper.outType() == self.sink.getSinkType(),
            message: "Swapper outType \(self.swapper.outType().identifier) "
                   + "!= Sink type \(self.sink.getSinkType().identifier)"
        )

        self.startingSourceBalance = acct
            .borrow<&{FungibleToken.Vault}>(from: /storage/flowTokenVault)!.balance

        // Snapshot: minimumAvailable() is not view — cannot call in pre block
        self.sourceAvailableAtStart = self.source.minimumAvailable()
    }

    pre { self.sourceAvailableAtStart > 0.0: "Source has no available balance" }

    execute {
        // Step 1: withdraw from source
        let headroom = self.sink.minimumCapacity()
        let inVault <- self.source.withdrawAvailable(maxAmount: headroom)

        // Step 2: swap token0 → token1
        // Quotes are advisory — assert actual output, not quote.outAmount
        let q = self.swapper.quoteOut(forProvided: inVault.balance, reverse: false)
        assert(q.outAmount > 0.0, message: "Swapper returned unavailable quote")
        let outVault <- self.swapper.swap(quote: q, inVault: <-inVault)

        // Slippage guard on the actual output vault
        assert(
            outVault.balance >= minExpectedOut,
            message: "Swap output \(outVault.balance) below minimum \(minExpectedOut)"
        )

        // Step 3: deposit into sink
        self.sink.depositCapacity(from: &outVault as auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
        assert(outVault.balance == 0.0, message: "Residual after deposit: \(outVault.balance)")
        destroy outVault
    }

    post { self.startingSourceBalance > self.source.minimumAvailable():
        "Source balance did not decrease — withdrawal may not have occurred" }
}
```

---

## Pattern 5: Multi-Hop via SequentialSwapper

**When to use:** The direct path from source token to sink token doesn't exist in a single pool.
Chain multiple Swappers via `SwapConnectors.SequentialSwapper`, which exposes the same
`{DeFiActions.Swapper}` interface so it composes identically with `SwapSource` or direct chaining.

`SequentialSwapper` validates `hop[n].outType() == hop[n+1].inType()` at construction time. Any
hop-ordering error surfaces in `prepare` before vault movement.

```cadence
import "DeFiActions"
import "SwapConnectors"
import "IncrementFiSwapConnectors"

// In prepare:
let operationID = DeFiActions.createUniqueIdentifier()

// Hop 1: TokenA → TokenB (e.g. stablecoin intermediate)
let hop1 = IncrementFiSwapConnectors.Swapper(
    path: ["A.abc.TokenA", "A.abc.TokenB"],
    inVault:  Type<@TokenA.Vault>(),
    outVault: Type<@TokenB.Vault>(),
    uniqueID: operationID
)

// Hop 2: TokenB → TokenC (final destination token)
let hop2 = IncrementFiSwapConnectors.Swapper(
    path: ["A.abc.TokenB", "A.abc.TokenC"],
    inVault:  Type<@TokenB.Vault>(),
    outVault: Type<@TokenC.Vault>(),
    uniqueID: operationID
)

// SequentialSwapper asserts hop1.outType() == hop2.inType() at construction
let multiHop = SwapConnectors.SequentialSwapper(
    swappers: [hop1, hop2],
    uniqueID: operationID
)
// multiHop.inType()  == TokenA.Vault
// multiHop.outType() == TokenC.Vault

// From here: use multiHop exactly like any single-hop Swapper
// e.g. SwapConnectors.SwapSource(swapper: multiHop, source: baseSource, uniqueID: operationID)
```

Each hop in a `SequentialSwapper` carries the same `operationID`, so all intermediate `Swapped`
events are traceable to the same pipeline.

---

## Pattern Selection Decision Tree

```
Start: What token does the Source produce? What token does the Sink accept?
│
├── Same token type
│   └── Pattern 1: Source → Sink
│
└── Different token types
    │
    ├── One swap required
    │   │
    │   ├── Prefer to wrap around the Source side?
    │   │   └── Pattern 2: SwapSource(Swapper + Source) → Sink
    │   │
    │   ├── Prefer to wrap around the Sink side?
    │   │   └── Pattern 3: Source → SwapSink(Swapper + Sink)
    │   │
    │   └── Need per-step slippage guards or explicit intermediate assertions?
    │       └── Pattern 4: Source → Swapper → Sink (direct chaining)
    │
    └── Multiple hops required (no direct pool exists)
        └── Pattern 5: SequentialSwapper for multi-hop, then apply Pattern 2, 3, or 4
```

**Guidance on Pattern 2 vs. Pattern 3:**
- Use Pattern 2 (SwapSource) when the upstream protocol naturally speaks token0 and the
  downstream protocol naturally speaks token1. The swap "upgrades" the output before presenting
  it to the Sink.
- Use Pattern 3 (SwapSink) when you have direct access to the source vault (e.g. the user's own
  wallet) but the target protocol requires a specific token. The Sink handles conversion
  internally, keeping the Source simple.
- Pattern 4 is most appropriate when building custom tooling, writing tests, or when the
  composition logic itself needs to be observable (e.g. an auto-balancer that conditionally
  bypasses the swap if types already match).

---

## Pre/Post Conditions Placement Guide

### In `pre` — what to check before execution

```cadence
// ❌ Calling minimumAvailable() directly in pre — DOES NOT COMPILE
// FungibleTokenConnectors.VaultSource.minimumAvailable() is NOT view.
// Cadence rejects non-view calls in pre/post with "impure operation performed in view context".
pre {
    self.source.minimumAvailable() >= minimumAmount: "..."  // compile error
}
```

```cadence
// ✅ Correct: snapshot in prepare, reference the field in pre
// In the transaction-level field declarations:
//   let availableAtStart: UFix64
// In prepare:
//   self.availableAtStart = self.source.minimumAvailable()
pre {
    self.availableAtStart >= minimumAmount:
        "Source balance \(self.availableAtStart) is below minimum \(minimumAmount)"
}
```

```cadence
// ❌ Variable declaration in pre — DOES NOT COMPILE in Cadence
pre {
    let avail = self.source.minimumAvailable()  // syntax error
    avail >= minimumAmount: "..."
}
```

Typical `pre` checks:
- Source has sufficient available balance for the operation
- Sink has non-zero capacity (use only if a zero-capacity deposit would be misleading)
- Caller holds a required capability (checked via `cap.check()` in `prepare` rather than `pre`)

### In `post` — what to assert after execution

```cadence
// Capture snapshot values in prepare, reference them in post
let startingBalance: UFix64    // set in prepare
let startingStake: UFix64      // set in prepare

post {
    // ✅ Single boolean expression referencing prepare-time snapshots
    self.startingBalance - withdrawnAmount <=
        acct.borrow<&{FungibleToken.Vault}>(from: /storage/myVault)!.balance:
        "Source balance did not decrease as expected"
}
```

Typical `post` checks:
- Sink balance increased by at least `minExpectedOut` (slippage lower bound)
- Source balance decreased by approximately the withdrawn amount
- A staked position's balance reflects the new deposit

**Never place vault creation, intermediate vault operations, or resource moves in `pre`/`post`.
All resource movement belongs in `execute`.**

---

## Atomicity Guarantees

### All-or-nothing

Every step in a DeFiActions composition — `withdrawAvailable`, `swap`, `depositCapacity` — runs
inside a single Cadence transaction. If any step panics (type mismatch, slippage floor violated,
broken capability), the entire transaction reverts. No tokens are moved, no events are emitted,
and no state changes persist. This is fundamentally different from EVM multi-call patterns where
individual calls can commit independently.

### Partial fills are not allowed

The composition contract requires that `vault.balance == 0.0` after `depositCapacity` returns.
If a Sink can only absorb a fraction of the vault, the caller's `assert(vault.balance == 0.0)`
will revert the entire transaction — the partial fill is rejected. Implementations must size
withdrawals to `sink.minimumCapacity()` to prevent this. If the Sink is temporarily full
(`minimumCapacity() == 0.0`), abort rather than proceeding with a zero deposit:

```cadence
execute {
    let headroom = self.sink.minimumCapacity()
    assert(headroom > 0.0, message: "Sink is at capacity — aborting")
    let vault <- self.source.withdrawAvailable(maxAmount: headroom)
    self.sink.depositCapacity(from: &vault as auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
    assert(vault.balance == 0.0, message: "Residual after deposit: \(vault.balance)")
    destroy vault
}
```

### Vault empty before destruction

The `destroy vault` call at the end of `execute` is a Cadence resource destruction. Cadence
panics if a `FungibleToken.Vault` resource is destroyed with `balance > 0.0` (non-zero balances
cannot be lost). The `assert(vault.balance == 0.0)` before `destroy` makes this invariant
explicit and produces a descriptive error message rather than an opaque resource-destruction
panic.

---

## Six Critical Safety Rules

These rules are blocking — a composition that violates any one is either broken or a security
risk.

### 1. Import syntax: string only

```cadence
// ✅ Correct
import "DeFiActions"
import "SwapConnectors"
import "FungibleTokenConnectors"

// ❌ Address-based import — breaks portability across networks
import DeFiActions from 0x...
```

### 2. Pre/post: single boolean expressions only

No variable declarations. No multi-statement logic. Store snapshot values in `prepare` fields.

### 3. Accept addresses as transaction parameters

```cadence
// ✅ Correct — poolAddress is a transaction parameter
transaction(poolAddress: Address, minOut: UFix64) { ... }

// ❌ Hardcoded address — breaks on every non-mainnet environment
let pool = getAccount(0xAbc123...).capabilities.borrow(...)
```

### 4. Token ordering: source token must be inType

When constructing a Swapper or `SwapSource`, verify that `source.getSourceType() == swapper.inType()`.
Determine the pool's canonical token ordering in `prepare` and reverse the Swapper's construction
arguments if needed:

```cadence
let reverse = source.getSourceType() != pool.token0Type()
let swapper = MySwapper(
    inVault:  reverse ? pool.token1Type() : pool.token0Type(),
    outVault: reverse ? pool.token0Type() : pool.token1Type(),
    uniqueID: operationID
)
```

### 5. Assert vault empty before destruction

```cadence
assert(vault.balance == 0.0, message: "Residual balance after deposit: \(vault.balance)")
destroy vault
```

Never `destroy` a vault without the balance assertion. A non-zero residual means the composition
lost tokens silently — the Sink absorbed less than expected.

### 6. Single UniqueIdentifier across all connectors

Create one `operationID` per pipeline. Pass it to every connector (Source, Swapper, Sink,
SwapSource, SwapSink). This is what links all emitted events (`Withdrawn`, `Swapped`,
`Deposited`) into a single traceable operation.

```cadence
// ✅ One ID, all connectors
let operationID = DeFiActions.createUniqueIdentifier()
let source  = VaultSource(..., uniqueID: operationID)
let swapper = MySwapper(..., uniqueID: operationID)
let sink    = VaultSink(..., uniqueID: operationID)

// ❌ Separate IDs break event correlation
let source  = VaultSource(..., uniqueID: DeFiActions.createUniqueIdentifier())
let swapper = MySwapper(...,   uniqueID: DeFiActions.createUniqueIdentifier())
let sink    = VaultSink(...,   uniqueID: DeFiActions.createUniqueIdentifier())
```

---

## Token Ordering Anti-Patterns

### ❌ SwapSource with mismatched source/swapper types

```cadence
// Source produces TokenA; Swapper expects TokenB as inType
let source  = VaultSource(/* TokenA vault */, uniqueID: operationID)
let swapper = MySwapper(inVault: Type<@TokenB.Vault>(), outVault: Type<@TokenC.Vault>(),
                        uniqueID: operationID)
// PANIC in prepare: SwapSource pre-condition fires
// "Source outputs A.xxx.TokenA.Vault but Swapper takes A.xxx.TokenB.Vault"
let swapSource = SwapConnectors.SwapSource(swapper: swapper, source: source, uniqueID: operationID)
```

```cadence
// ✅ Correct: align swapper.inType() with source.getSourceType()
let sourceType = source.getSourceType()   // TokenA
let swapper    = MySwapper(inVault: sourceType, outVault: Type<@TokenC.Vault>(),
                           uniqueID: operationID)
let swapSource = SwapConnectors.SwapSource(swapper: swapper, source: source, uniqueID: operationID)
```

### ❌ Source → Sink direct chain without type check

```cadence
// Source produces TokenA; Sink expects TokenB
let vault <- source.withdrawAvailable(maxAmount: 100.0)
sink.depositCapacity(from: &vault as auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
// PANIC at Sink pre-condition: "Invalid vault for deposit — TokenA is not TokenB"
```

```cadence
// ✅ Correct: assert or interpose a Swapper before wiring
assert(
    source.getSourceType() == sink.getSinkType(),
    message: "Type mismatch: source produces \(source.getSourceType().identifier), "
           + "sink expects \(sink.getSinkType().identifier) — interpose a Swapper"
)
```

### ❌ SwapSink with mismatched swapper/sink types

```cadence
// Swapper produces TokenB; Sink accepts TokenC
let swapper = MySwapper(inVault: Type<@TokenA.Vault>(), outVault: Type<@TokenB.Vault>(), ...)
let sink    = MySink(/* accepts TokenC */)
// PANIC in prepare: SwapSink pre-condition fires
// "Swapper outputs A.xxx.TokenB.Vault but Sink takes A.xxx.TokenC.Vault"
let swapSink = SwapConnectors.SwapSink(swapper: swapper, sink: sink, uniqueID: operationID)
```

### ❌ Ignoring the `reverse: Bool` semantics on quotes

```cadence
// Priced in reverse (outType → inType), then called swap() instead of swapBack()
let q = swapper.quoteIn(forDesired: 50.0, reverse: true)   // prices outType → inType
let out <- swapper.swap(quote: q, inVault: <-outTypeVault)
// PANIC: swap() pre-condition requires inVault.getType() == swapper.inType()
// but outTypeVault is swapper.outType()

// ✅ Correct: match swap call direction to quote direction
let q   = swapper.quoteIn(forDesired: 50.0, reverse: true)
let out <- swapper.swapBack(quote: q, residual: <-outTypeVault)
```

---

## Capability Nil-Check Pattern

Every capability borrow must be guarded. A broken capability mid-pipeline produces a
non-descriptive panic. Check in `prepare` and surface a clear message:

```cadence
// ❌ Force-unwrap produces an opaque panic message
let pool = poolCapability.borrow()!

// ✅ Explicit nil-check with context
let pool = poolCapability.borrow()
    ?? panic("Pool capability is nil — controller may have been revoked for address \(poolAddress)")
```

Connectors (`VaultSource`, `VaultSink`) perform `cap.check()` in their `init` pre-condition and
return `0.0`/empty vault on nil borrows during execution. Custom connector implementations must
follow the same pattern.

---

## Common Pitfalls

**Forgetting to size by `minimumCapacity()`:** Withdrawing a fixed amount without checking the
Sink's headroom first will cause `vault.balance > 0.0` after `depositCapacity`, triggering the
residual assertion. Always use `maxWithdraw = min(requestedAmount, sink.minimumCapacity())`.

**Stale `minimumAvailable()`/`minimumCapacity()` from `prepare`:** These values are snapshots.
If other operations in `execute` modify the same vault before the withdraw or deposit, the
captured size may be wrong. Re-read sizing values inside `execute`, not in `prepare`.

**Sharing a Swapper struct across multiple pipelines:** Each pipeline must use a distinct
Swapper instance and its own `UniqueIdentifier`. A Swapper holds a reference to one ID; sharing
it across pipelines conflates their event streams.

**Zero-quote not checked before swap:** A quote with `inAmount == 0.0 && outAmount == 0.0` means
the Swapper has no price for the requested amount. Calling `swap()` with such a quote is
implementation-defined and may produce zero output or revert. Always assert `q.outAmount > 0.0`
before proceeding.

**Variable declarations in `pre`/`post`:** Cadence does not allow `let`/`var` in pre/post blocks.
Move any snapshot captures to `prepare` fields and reference them directly in the condition
expression.

**Capability issued inside `execute` instead of `prepare`:** `IssueStorageCapabilityController`
authorization is a `prepare`-phase operation. Attempting to issue a new storage capability in
`execute` will fail at authorization. Issue all capabilities in `prepare`.

---

## Cross-links

- [source-interface.md](source-interface.md) — `withdrawAvailable`, `minimumAvailable`,
  `getSourceType` semantics and the `VaultSource` reference implementation
- [sink-interface.md](sink-interface.md) — `depositCapacity`, `minimumCapacity`, `getSinkType`
  semantics and the `VaultSink` reference implementation
- [swapper-interface.md](swapper-interface.md) — token ordering rules, `SwapSource`/`SwapSink`
  wrapper pre-conditions, `SequentialSwapper`, quote vs. execution guarantees
- [price-oracle-interface.md](price-oracle-interface.md) — using a `PriceOracle` as a slippage
  guard on top of a Swapper quote
- [scheduled-integration.md](scheduled-integration.md) — how to execute a composition on a
  recurring schedule using `FlowTransactionScheduler`
- [../../cadence-lang/references/scheduled-transactions.md](../../cadence-lang/references/scheduled-transactions.md) —
  the Forte API used by scheduled callbacks
- [../../cadence-scaffold/references/scaffold-defi.md](../../cadence-scaffold/references/scaffold-defi.md) —
  IncrementFi-coupled restake scaffold that implements the canonical restake variant of Pattern 2
