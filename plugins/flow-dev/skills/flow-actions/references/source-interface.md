# Source Interface

A `Source` is the "where money comes from" abstraction in the DeFiActions framework. It is a
struct interface that decouples token extraction from any specific protocol — staking pools,
plain vaults, ERC-4626 EVM positions, and custom yield strategies look identical to the caller
once wrapped in a Source. Because Sources are protocol-agnostic by design, the same transaction
logic drives funds from IncrementFi, Trado, Morpho, or a custom strategy without modification.
Implement a custom Source when you need to connect a new funding mechanism to a DeFiActions
pipeline.

> BETA NOTICE: DeFiActions is in beta and interfaces may change. Definitions below are sourced
> from `onflow/FlowActions` (`main` branch, `cadence/contracts/interfaces/DeFiActions.cdc`).

---

## Interface Definition

```cadence
import "FungibleToken"
import "DeFiActions"

// Source extends IdentifiableStruct, adding operation-traceability fields.
// The three methods below are the Source-specific surface area.
access(all) struct interface Source : DeFiActions.IdentifiableStruct {

    /// Returns the Vault type this Source claims to offer.
    /// CAUTION: Untrusted implementations may return a different type than
    /// withdrawAvailable() produces. Callers MUST validate returned vault type.
    access(all) view fun getSourceType(): Type

    /// Returns a non-blocking estimate of how many tokens are currently extractable.
    access(all) fun minimumAvailable(): UFix64

    /// Withdraws the lesser of maxAmount or minimumAvailable().
    /// Returns an empty Vault if nothing is available — never reverts.
    /// CAUTION: Callers MUST validate the returned vault's type.
    access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault} {
        post {
            DeFiActions.emitWithdrawn(
                type: result.getType().identifier,
                amount: result.balance,
                withdrawnUUID: result.uuid,
                uniqueID: self.uniqueID?.id ?? nil,
                sourceType: self.getType().identifier
            ): "Unknown error emitting DeFiActions.Withdrawn"
        }
    }
}
```

`Source` also inherits from `DeFiActions.IdentifiableStruct`:

```cadence
access(contract) var uniqueID: DeFiActions.UniqueIdentifier?
access(all)      view fun id(): UInt64?
access(all)           fun getComponentInfo(): DeFiActions.ComponentInfo
access(contract) view fun copyID(): DeFiActions.UniqueIdentifier?
access(contract)      fun setID(_ id: DeFiActions.UniqueIdentifier?)
```

---

## Method Contracts

### `getSourceType(): Type`

**Caller may assume**: Return value is stable for the struct's lifetime.

**Implementation must guarantee**: The return type always matches the vault type produced by
`withdrawAvailable`. Must not mutate state (`view`). Must succeed in all states.

---

### `minimumAvailable(): UFix64`

**Caller may assume**: A conservative underestimate — actual extractable amount may be
higher. Not a hard lower bound. Calling it multiple times without intervening state changes
returns consistent values.

**Implementation must guarantee**:
- Must not revert under any conditions.
- Returns `0.0` when the underlying source is inaccessible or empty.
- Has no side effects.

---

### `withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault}`

**Caller may assume**: Returned vault type matches `getSourceType()`. If the source is empty
or unreachable, returns an empty vault rather than panicking.

**Implementation must guarantee**:
- Returned vault type equals `getSourceType()`.
- Returned balance `<= maxAmount`.
- If `minimumAvailable()` is `0.0` or the capability is `nil`, return an empty vault — do
  not revert.
- Use `DeFiActionsUtils.getEmptyVault(self.type)` for empty returns.
- The `post` block emits `DeFiActions.Withdrawn` automatically when amount `> 0.0`.
  Implementations must not emit this event manually.

**Entitlement requirement**: Caller must hold `FungibleToken.Withdraw` authorization.

---

## Events

The `withdrawAvailable` interface-level `post` block emits:

```cadence
access(all) event Withdrawn(
    type: String,          // vault type identifier
    amount: UFix64,        // balance of returned vault
    withdrawnUUID: UInt64, // uuid of returned vault resource
    uniqueID: UInt64?,     // operation ID if set, else nil
    sourceType: String     // concrete struct type of this Source
)
```

Event fires only when `amount > 0.0`. An empty-vault return emits no event. Callers needing
to detect empty extraction must check `vault.balance == 0.0` explicitly.

---

## Capacity Semantics

`minimumAvailable()` is a best-effort underestimate, not a guarantee. Between
`minimumAvailable()` and `withdrawAvailable()` in the same transaction, on-chain state can
shift (e.g., a staking snapshot updates). Implementations must re-read the underlying balance
inside `withdrawAvailable()` rather than relying on any cached value:

```cadence
// Correct — re-read at withdrawal time
access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault} {
    let available = self.minimumAvailable()          // fresh read
    if !self.myVault.check() || available == 0.0 || maxAmount == 0.0 {
        return <- DeFiActionsUtils.getEmptyVault(self.withdrawVaultType)
    }
    let amount = available <= maxAmount ? available : maxAmount
    return <- self.myVault.borrow()!.withdraw(amount: amount)
}
```

In a `Source → Sink` pipeline, size the withdrawal to the Sink's declared capacity:

```cadence
let vault <- source.withdrawAvailable(maxAmount: sink.minimumCapacity())
sink.depositCapacity(from: &vault as auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
assert(vault.balance == 0.0, message: "Residual after deposit")
destroy vault
```

---

## Failure Modes

| Scenario | Required behavior |
|---|---|
| `minimumAvailable()` returns `0.0`, caller calls `withdrawAvailable` | Return empty vault. No revert. |
| Underlying contract is paused | Return `0.0` from `minimumAvailable()`, empty vault from `withdrawAvailable()`. Check capability before borrowing. |
| Capability is `nil` mid-execution | Return empty vault. Guard with `if let x = self.cap.borrow() { ... }` pattern. |
| `maxAmount == 0.0` passed by caller | Return empty vault immediately. |

---

## Worked Example 1 — `VaultSource` (Canonical Protocol-Agnostic Source)

`FungibleTokenConnectors.VaultSource` (shipped with FlowActions) wraps any
`auth(FungibleToken.Withdraw) &{FungibleToken.Vault}` capability. An optional `minimumBalance`
parameter reserves a floor so the vault is never fully drained.

```cadence
import "FungibleToken"
import "DeFiActionsUtils"
import "DeFiActions"

access(all) struct VaultSource : DeFiActions.Source {

    access(all) let withdrawVaultType: Type
    access(all) let minimumBalance: UFix64
    access(contract) var uniqueID: DeFiActions.UniqueIdentifier?
    access(self) let withdrawVault: Capability<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>

    init(
        min: UFix64?,
        withdrawVault: Capability<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>,
        uniqueID: DeFiActions.UniqueIdentifier?
    ) {
        pre {
            withdrawVault.check(): "Provided invalid Capability"
            DeFiActionsUtils.definingContractIsFungibleToken(withdrawVault.borrow()!.getType()):
                "Vault type does not conform to FungibleToken"
        }
        self.minimumBalance = min ?? 0.0
        self.withdrawVault = withdrawVault
        self.uniqueID = uniqueID
        self.withdrawVaultType = withdrawVault.borrow()!.getType()
    }

    access(all) view fun getSourceType(): Type { return self.withdrawVaultType }

    access(all) fun minimumAvailable(): UFix64 {
        if let vault = self.withdrawVault.borrow() {
            return self.minimumBalance < vault.balance
                ? vault.balance - self.minimumBalance
                : 0.0
        }
        return 0.0
    }

    access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault} {
        let available = self.minimumAvailable()
        if !self.withdrawVault.check() || available == 0.0 || maxAmount == 0.0 {
            return <- DeFiActionsUtils.getEmptyVault(self.withdrawVaultType)
        }
        let amount = available <= maxAmount ? available : maxAmount
        return <- self.withdrawVault.borrow()!.withdraw(amount: amount)
    }

    access(all) fun getComponentInfo(): DeFiActions.ComponentInfo {
        return DeFiActions.ComponentInfo(type: self.getType(), id: self.id(), innerComponents: [])
    }
    access(contract) view fun copyID(): DeFiActions.UniqueIdentifier? { return self.uniqueID }
    access(contract)      fun setID(_ id: DeFiActions.UniqueIdentifier?) { self.uniqueID = id }
}
```

---

## Worked Example 2 — `UniqueIdentifier` and Operation Traceability

All connectors in a single pipeline share one `UniqueIdentifier` created at transaction start.
It appears in every emitted event, allowing full trace reconstruction from event logs.

```cadence
import "FungibleToken"
import "DeFiActions"
import "FungibleTokenConnectors"

transaction(recipient: Address, amount: UFix64) {

    let source: FungibleTokenConnectors.VaultSource
    let sink: FungibleTokenConnectors.VaultSink

    prepare(acct: auth(BorrowValue, IssueStorageCapabilityController) &Account) {
        let operationID = DeFiActions.createUniqueIdentifier()

        let senderCap = acct.capabilities.storage
            .issue<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>(/storage/flowTokenVault)

        let recipientCap = getAccount(recipient)
            .capabilities.get<&{FungibleToken.Vault}>(/public/flowTokenReceiver)

        self.source = FungibleTokenConnectors.VaultSource(
            min: nil, withdrawVault: senderCap, uniqueID: operationID
        )
        self.sink = FungibleTokenConnectors.VaultSink(
            max: nil, depositVault: recipientCap, uniqueID: operationID
        )
    }

    pre { self.source.minimumAvailable() >= amount: "Insufficient balance" }

    execute {
        let cap = self.sink.minimumCapacity()
        let maxWithdraw = cap < amount ? cap : amount
        let vault <- self.source.withdrawAvailable(maxAmount: maxWithdraw)
        self.sink.depositCapacity(from: &vault as auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
        assert(vault.balance == 0.0, message: "Residual balance after deposit")
        destroy vault
    }
}
```

---

## Anti-patterns

### Returning a vault of the wrong type

```cadence
// ❌ getSourceType() declares FlowToken.Vault but withdrawAvailable returns USDC
access(all) view fun getSourceType(): Type { return Type<@FlowToken.Vault>() }
access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault} {
    return <- self.usdcVault.borrow()!.withdraw(amount: maxAmount)  // type mismatch!
}

// ✅ Capture vault type at init; both methods reference the same field
access(all) view fun getSourceType(): Type { return self.withdrawVaultType }
access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault} {
    return <- self.withdrawVault.borrow()!.withdraw(amount: withdrawalAmount)
}
```

### Reverting instead of returning an empty vault

```cadence
// ❌ Panics propagate through the entire composed pipeline
access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault} {
    let pool = self.poolCap.borrow() ?? panic("Pool unavailable")
    assert(pool.available() > 0.0, message: "Nothing to withdraw")  // breaks callers
    return <- pool.withdraw(amount: pool.available())
}

// ✅ Return empty vault on all non-critical failures
access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault} {
    let available = self.minimumAvailable()
    if !self.poolCap.check() || available == 0.0 || maxAmount == 0.0 {
        return <- DeFiActionsUtils.getEmptyVault(self.withdrawVaultType)
    }
    return <- self.poolCap.borrow()!.withdraw(amount: available <= maxAmount ? available : maxAmount)
}
```

### Mutating external state beyond the extraction

```cadence
// ❌ Side effects in withdrawAvailable break composability
access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault} {
    SomeOtherProtocol.triggerRebalance()  // side effect — breaks pipeline guarantees
    return <- self.vault.borrow()!.withdraw(amount: maxAmount)
}

// ✅ withdrawAvailable does one thing: extract and return tokens
```

### Caching `minimumAvailable()` at construction time

```cadence
// ❌ Stale cached value — wrong when state changes before execute{} runs
access(all) let cachedAvailable: UFix64  // captured in init()
access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault} {
    return <- self.vault.borrow()!.withdraw(amount: self.cachedAvailable)  // stale!
}

// ✅ Recompute inside withdrawAvailable
access(FungibleToken.Withdraw) fun withdrawAvailable(maxAmount: UFix64): @{FungibleToken.Vault} {
    let available = self.minimumAvailable()  // fresh read every time
    ...
}
```

---

## Common Pitfalls

**Missing `auth(FungibleToken.Withdraw)` on the stored capability**: An unentitled capability
compiles but cannot call `vault.withdraw()`. Verify at construction with `cap.check()`.

**`access(all)` instead of `access(FungibleToken.Withdraw)` on `withdrawAvailable`**: The
interface requires the entitlement guard. Dropping it allows unauthenticated drains.

**Not using `DeFiActionsUtils.getEmptyVault(type)` for empty returns**: The return type is
non-optional (`@{FungibleToken.Vault}`), so `nil` is not valid. Always use the utility.

**Re-using a `UniqueIdentifier` across unrelated operations**: Create one per pipeline with
`DeFiActions.createUniqueIdentifier()`. Sharing conflates independent event streams.

---

## Cross-links

- [sink-interface.md](sink-interface.md) — the companion interface for deposit operations
- [swapper-interface.md](swapper-interface.md) — transforms tokens between types in a pipeline
- [composition-patterns.md](composition-patterns.md) — how Source composes with Sink and Swapper
