# Sink Interface

A Sink is a struct interface in the DeFiActions framework that represents the destination end of a
token flow — the place where fungible tokens are deposited. It is the inverse of the Source
interface: whereas a Source offers tokens on demand and reports what it can provide, a Sink accepts
tokens up to a capacity it reports in advance. Implementing a custom Sink is appropriate whenever
tokens need to be directed into a protocol-specific position (staking pool, lending market, wrapped
vault) without embedding that protocol's details in the calling transaction.

> **Beta notice:** `DeFiActions` is in beta on Testnet and Mainnet. Interfaces may change before
> final release. Monitor [`onflow/FlowActions`](https://github.com/onflow/FlowActions) for breaking
> changes.

---

## Interface Definition

```cadence
import "FungibleToken"
import "DeFiActions"

access(all) struct interface Sink : DeFiActions.IdentifiableStruct {

    /// Returns the Vault type this Sink accepts.
    access(all) view fun getSinkType(): Type

    /// Returns an estimate of how much can be withdrawn from the depositing Vault
    /// for this Sink to reach capacity.
    access(all) fun minimumCapacity(): UFix64

    /// Deposits up to the Sink's capacity from the provided Vault.
    /// The interface-level pre-condition enforces vault-type matching.
    /// The interface-level post-condition emits DeFiActions.Deposited.
    access(all) fun depositCapacity(from: auth(FungibleToken.Withdraw) &{FungibleToken.Vault}) {
        pre {
            from.getType() == self.getSinkType():
                "Invalid vault for deposit — \(from.getType().identifier) is not \(self.getSinkType().identifier)"
        }
        post {
            DeFiActions.emitDeposited(
                type: from.getType().identifier,
                beforeBalance: before(from.balance),
                afterBalance: from.balance,
                fromUUID: from.uuid,
                uniqueID: self.uniqueID?.id,
                sinkType: self.getType().identifier
            ): "Unknown error emitting DeFiActions.Deposited from Sink \(self.getType().identifier)"
        }
    }

    // Inherited from IdentifiableStruct:

    /// Returns a ComponentInfo struct describing this Sink and its inner components.
    access(all) fun getComponentInfo(): DeFiActions.ComponentInfo

    /// Returns the UniqueIdentifier id, or nil if none is set.
    access(all) view fun id(): UInt64?
}
```

---

## Method-by-Method Semantics

### `getSinkType(): Type`

Returns the exact Cadence `Type` of the `FungibleToken.Vault` this Sink will accept. The
interface-level pre-condition on `depositCapacity` compares `from.getType()` against this value
and panics on mismatch, so an incorrect return here causes every deposit attempt to revert.

**Contracts:**
- Must be a `view` function; implementations must not mutate state to determine the type.
- Must be stable for the lifetime of the Sink struct instance. Callers cache this value to size
  withdrawals from a companion Source before calling `depositCapacity`.
- Must return the concrete `@VaultImpl` type, not the abstract `@{FungibleToken.Vault}` type.

### `minimumCapacity(): UFix64`

Returns a lower-bound estimate of how many tokens the Sink can absorb right now. Callers use this
value to size the `maxAmount` parameter passed to a companion Source's `withdrawAvailable`. A return
of `0.0` signals that the Sink is currently full or unavailable and the deposit step should be
skipped.

**Contracts:**
- Not a `view` function; implementations may read on-chain state (e.g., current vault balance,
  pool utilization).
- Liveness priority: if the Sink's underlying capability is invalid, return `0.0` rather than
  panicking.
- The value is a snapshot; it may decrease between the call and the subsequent `depositCapacity`
  if another operation executes in the same transaction (see "Failure Modes" below).
- Implementations that have no upper bound (e.g., an unbounded accumulation vault) should return
  `UFix64.max`.

### `depositCapacity(from: auth(FungibleToken.Withdraw) &{FungibleToken.Vault})`

Withdraws up to `minimumCapacity()` tokens from `from` and routes them to the Sink's underlying
destination. The caller retains ownership of `from`; the Sink withdraws from it via the
`FungibleToken.Withdraw` entitlement — it never takes ownership of the vault resource itself.

**Contracts — what the Sink may assume:**
- `from.getType() == self.getSinkType()` is guaranteed by the interface pre-condition.
- The caller has already sized `from.balance` using `minimumCapacity()`, so `from.balance` is
  typically at or below the capacity at the moment of the call.

**Contracts — what the Sink must guarantee:**
- After return, `from.balance >= 0.0` (cannot over-withdraw; Cadence arithmetic panics on
  underflow anyway).
- The Sink must not deposit more than `minimumCapacity()` tokens; depositing excess causes the
  underlying position to exceed its stated limit.
- The Sink must not hold or store the `from` reference beyond the call frame. Storing a
  capability reference after the function returns is a lifecycle violation.
- A no-op on capacity `0.0` or a broken capability is the expected graceful path — do not panic
  unless failing silently would leave the Sink in an inconsistent internal state.

### `getComponentInfo(): DeFiActions.ComponentInfo` / `id(): UInt64?`

`getComponentInfo()` returns a `ComponentInfo` describing this Sink's concrete type and
`UniqueIdentifier` id. Flat Sinks return `innerComponents: []`; wrappers include their inner
connector's info. `id()` is a convenience accessor for `self.uniqueID?.id` and surfaces in every
`DeFiActions.Deposited` event, enabling end-to-end operation traceability.

---

## Vault Validation Invariants

The following invariants must hold on every conforming Sink implementation. The first is enforced
by the interface; the remaining two are implementation responsibilities.

### 1. Type check (interface-enforced)

```cadence
// Enforced automatically by the Sink interface pre-condition.
// from.getType() == self.getSinkType()
```

An implementation cannot bypass this check. If a Sink implementation returns the wrong type from
`getSinkType()`, every deposit will revert with a type-mismatch panic.

### 2. Empty-vault check before destruction (caller responsibility)

After `depositCapacity` returns, the caller must assert `vault.balance == 0.0` before destroying
the vault. A non-zero residual means the Sink absorbed less than provided — a logic or capacity
error (see anti-patterns).

### 3. Capacity check (implementation responsibility)

The implementation must not deposit more than `minimumCapacity()` tokens. The canonical pattern:

```cadence
let amount = cap <= from.balance ? cap : from.balance
self.depositVault.borrow()!.deposit(from: <-from.withdraw(amount: amount))
```

---

## Events

Event emission is handled by the interface-level post-condition via `DeFiActions.emitDeposited`.
Implementations do not need to emit events manually; they will be emitted automatically after
`depositCapacity` returns.

```cadence
// Emitted by DeFiActions contract after every depositCapacity call
access(all) event Deposited(
    type: String,       // vault type identifier
    amount: UFix64,     // tokens actually deposited (before.balance - after.balance)
    fromUUID: UInt64,   // UUID of the source vault
    uniqueID: UInt64?,  // UniqueIdentifier.id of this Sink, if set
    sinkType: String    // concrete type identifier of this Sink implementation
)
```

**The `Deposited` event fires only when the deposit actually moves tokens.** If the sink's `depositCapacity(from:)` is called with a vault whose balance is zero or fits exactly into the post-deposit residual, no `Deposited` event is emitted. This is symmetric with `Withdrawn` on the Source side (see [source-interface.md](source-interface.md)). Consumers that need to confirm a sink received tokens MUST check the event payload, not just event presence.

Implementations may add protocol-specific events (e.g., `CapacityReached`, `Rejected`) but are
not required to. A silent no-op is the correct behavior for skipped deposits.

---

## Failure Modes

### Capacity unexpectedly zero (race within the same transaction)

A transaction that composes multiple Sinks may observe that the second Sink's `minimumCapacity()`
returns `0.0` even though a non-zero value was returned earlier in the same `prepare` block.
This happens when both Sinks share an underlying vault and the first deposit reduced the remaining
headroom to zero.

**Mitigation:** re-read `minimumCapacity()` immediately before calling `depositCapacity`, not in
`prepare`. Use the execute-block sizing pattern shown in the worked examples.

### Wrong vault type passed

The interface pre-condition panics with a descriptive message if `from.getType() != getSinkType()`.
This is the intended behavior: silently accepting the wrong type would corrupt the underlying
position. The only fix is to ensure the Source and Sink in a composition are paired on the same
vault type, or to interpose a Swapper.

### Sink underlying contract paused

Some protocol contracts expose a pause mechanism. A Sink backed by such a contract should check the
pause flag inside `depositCapacity` and return as a no-op when paused, rather than panicking.
The `minimumCapacity()` method should also return `0.0` when the underlying contract is paused so
that callers skip the deposit step entirely.

---

## Minimal Compliant Implementation — `VaultSink`

`FungibleTokenConnectors.VaultSink` (from the FlowActions repository) is the canonical
non-IncrementFi Sink. It wraps any `FungibleToken.Vault` capability and applies an optional
hard cap on the destination vault balance.

```cadence
import "FungibleToken"
import "DeFiActions"
import "DeFiActionsUtils"

access(all) struct VaultSink : DeFiActions.Sink {
    access(all) let depositVaultType: Type
    access(all) let maximumBalance: UFix64
    access(self) let depositVault: Capability<&{FungibleToken.Vault}>
    access(contract) var uniqueID: DeFiActions.UniqueIdentifier?

    init(max: UFix64?, depositVault: Capability<&{FungibleToken.Vault}>,
         uniqueID: DeFiActions.UniqueIdentifier?) {
        pre {
            depositVault.check(): "Invalid Vault Capability"
            DeFiActionsUtils.definingContractIsFungibleToken(
                depositVault.borrow()!.getType()
            ): "Vault type does not conform to FungibleToken"
            (max ?? UFix64.max) > 0.0: "maximumBalance must be > 0.0"
        }
        self.maximumBalance   = max ?? UFix64.max
        self.depositVaultType = depositVault.borrow()!.getType()
        self.depositVault     = depositVault
        self.uniqueID         = uniqueID
    }

    access(all) view fun getSinkType(): Type { return self.depositVaultType }

    access(all) fun minimumCapacity(): UFix64 {
        if let vault = self.depositVault.borrow() {
            return vault.balance < self.maximumBalance
                ? self.maximumBalance - vault.balance : 0.0
        }
        return 0.0  // broken capability — liveness fallback
    }

    access(all) fun depositCapacity(from: auth(FungibleToken.Withdraw) &{FungibleToken.Vault}) {
        let cap = self.minimumCapacity()
        if !self.depositVault.check() || cap == 0.0 { return }
        let amount = cap <= from.balance ? cap : from.balance
        self.depositVault.borrow()!.deposit(from: <-from.withdraw(amount: amount))
    }

    access(all) fun getComponentInfo(): DeFiActions.ComponentInfo {
        return DeFiActions.ComponentInfo(type: self.getType(), id: self.id(), innerComponents: [])
    }
    access(contract) view fun copyID(): DeFiActions.UniqueIdentifier? { return self.uniqueID }
    access(contract) fun setID(_ id: DeFiActions.UniqueIdentifier?) { self.uniqueID = id }
}
```

---

## Worked Example — Source to Sink (Simple Transfer)

```cadence
import "FungibleToken"
import "DeFiActions"
import "FungibleTokenConnectors"

transaction(recipientVaultCap: Capability<&{FungibleToken.Vault}>) {
    let source: FungibleTokenConnectors.VaultSource
    let sink:   FungibleTokenConnectors.VaultSink

    prepare(acct: auth(BorrowValue) &Account) {
        let opID = DeFiActions.createUniqueIdentifier()
        let withdrawCap = acct.capabilities.storage
            .issue<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>(/storage/flowTokenVault)
        self.source = FungibleTokenConnectors.VaultSource(min: nil, withdrawVault: withdrawCap, uniqueID: opID)
        self.sink   = FungibleTokenConnectors.VaultSink(max: nil, depositVault: recipientVaultCap, uniqueID: opID)
    }

    execute {
        // Size withdrawal by sink headroom, then deposit and assert residual
        let vault <- self.source.withdrawAvailable(maxAmount: self.sink.minimumCapacity())
        self.sink.depositCapacity(from: &vault as auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
        assert(vault.balance == 0.0, message: "Residual after deposit")
        destroy vault
    }
}
```

---

## Common Pitfalls

### ❌ Sink that does not check vault type

```cadence
// WRONG — skips the type check and silently accepts any vault
access(all) fun depositCapacity(from: auth(FungibleToken.Withdraw) &{FungibleToken.Vault}) {
    self.pool.deposit(from: <-from.withdraw(amount: from.balance))
    // If `from` is the wrong type, the pool may store an incompatible vault
    // or panic deep inside protocol code with a confusing error.
}
```

The interface pre-condition already enforces type equality, but a Sink that overrides
`getSinkType()` with the wrong type, or that accepts `&{FungibleToken.Vault}` (abstract) instead
of the concrete type, undermines this check. Always return the concrete vault type from
`getSinkType()`.

### ❌ Sink that holds the `from` reference beyond the call

```cadence
// WRONG — stores a reference to the caller's vault
access(self) var borrowedVault: auth(FungibleToken.Withdraw) &{FungibleToken.Vault}?

access(all) fun depositCapacity(from: auth(FungibleToken.Withdraw) &{FungibleToken.Vault}) {
    self.borrowedVault = from  // lifecycle violation: reference outlives the caller's frame
    // …
}
```

Storing a borrowed reference inside the Sink struct is undefined behavior. The vault resource
that `from` points to is owned by the calling transaction and may be destroyed after
`depositCapacity` returns. Only withdraw from `from` within the function body; never store the
reference.

### ❌ Sink that consumes less than provided without signalling it

```cadence
// WRONG — deposits only half, discards the rest silently
access(all) fun depositCapacity(from: auth(FungibleToken.Withdraw) &{FungibleToken.Vault}) {
    let half = from.balance / 2.0
    self.pool.deposit(from: <-from.withdraw(amount: half))
    // from still has half.balance — caller's residual assert will catch this,
    // but only if the caller remembers to assert. The Sink should not create
    // this situation in the first place.
}
```

If a Sink can only absorb a portion of the available tokens, `minimumCapacity()` must reflect that
limit accurately so the caller sizes the withdrawal correctly. A Sink that deposits less than
`from.balance` without a capacity-driven reason leaves a residual that the caller must handle,
breaking the composition contract.

### ❌ Sink that panics instead of returning `0.0` when unavailable

```cadence
// WRONG — panics on a broken capability instead of returning 0.0
access(all) fun minimumCapacity(): UFix64 {
    return self.poolCap.borrow()!.availableCapacity()  // force-unwrap panics if cap is broken
}
```

The Sink liveness contract requires a graceful fallback. A broken capability is a normal runtime
condition (the capability controller may have been revoked). Return `0.0` so the calling transaction
can choose to route tokens elsewhere rather than aborting the entire operation.

---

## Cross-links

- **Source interface** — `source-interface.md` — the inverse connector that provides tokens;
  `minimumAvailable()` is the sizing counterpart to `minimumCapacity()`.
- **Composition patterns** — `composition-patterns.md` — how to wire Source → Swapper → Sink
  atomically, including the execute-block sizing pattern and residual-vault assertion.
- **Swapper interface** — `swapper-interface.md` — when the Source and Sink operate on different
  vault types, a Swapper must be interposed between them.
- **Cadence scaffold** — `cadence-scaffold` skill / `scaffold-defi.md` — generates
  IncrementFi-coupled transactions that follow the composition rules described here.
