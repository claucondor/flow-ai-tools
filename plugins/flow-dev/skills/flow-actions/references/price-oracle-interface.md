# PriceOracle Interface

A `PriceOracle` is a DeFiActions struct that adapts an external price feed — on-chain oracle contract,
EVM vault NAV, or any push-based price keeper — into a single, uniform query surface:
`price(ofToken:)`. It is the highest-risk interface in the composition graph because oracle
manipulation is the leading exploit class in DeFi: a corrupted or stale price can cause a
liquidation engine to drain a healthy position, or let an underwater position escape liquidation
entirely. The interface trades unconditional freshness guarantees for composability; managing
staleness is the responsibility of the concrete implementation, not the interface.

> **Beta notice:** `DeFiActions` is in beta on Testnet and Mainnet. Interfaces may change before
> final release. Monitor [`onflow/FlowActions`](https://github.com/onflow/FlowActions) for breaking
> changes.

---

## Canonical Interface Definition

```cadence
import "DeFiActions"

/// PriceOracle
///
/// An interface for a price oracle adapter. Implementations should adapt this interface to
/// various price feed oracles deployed on Flow.
///
access(all) struct interface PriceOracle : DeFiActions.IdentifiableStruct {

    /// Returns the asset type serving as the price basis — e.g. the Vault type for USD in a
    /// FLOW/USD price pair. Callers use this to understand the denomination of the returned price.
    access(all) view fun unitOfAccount(): Type

    /// Returns the latest price of `ofToken` denominated in `unitOfAccount()` if available,
    /// otherwise `nil`. Callers must handle the nil case — the interface permits implementations
    /// to return nil when the price is genuinely unavailable (e.g. the token is not registered in
    /// a BandOracle symbol map). Implementations MAY also choose to revert rather than return nil
    /// (e.g. BandOracleConnectors reverts on stale data when `staleThreshold` is set).
    ///
    /// Interface-level post condition: result == nil || result! > 0.0
    /// A zero price is never valid; the contract enforces this at the interface boundary.
    access(all) fun price(ofToken: Type): UFix64? {
        post {
            result == nil || result! > 0.0:
            "PriceOracle must return a price greater than 0.0 if available"
        }
    }
}
```

### Method-by-method semantics

| Method | Mutability | Return | Semantics |
|---|---|---|---|
| `unitOfAccount()` | `view` | `Type` | The Vault type used as the price denominator. Callers should check this before interpreting any price value returned by `price(ofToken:)`. |
| `price(ofToken:)` | non-view (may call EVM, withdraw oracle fee) | `UFix64?` | Latest price of the given token expressed in units of `unitOfAccount()`. Returns `nil` when the token is unknown or price data is unavailable. May revert when staleness checking is enabled. |

`PriceOracle` also inherits `IdentifiableStruct`, which provides:
- `id() → UInt64?` — the `UniqueIdentifier` id for this component.
- `getComponentInfo() → ComponentInfo` — component type, id, and inner components for
  inspection and debugging.

There are **no** `getInputType()`, `getOutputType()`, `unitOfMeasure()`, or `lastUpdate()`
methods on the canonical interface. Staleness tracking is an implementation detail, not an
interface requirement. The BandOracle connector exposes `staleThreshold` as a constructor
parameter (see examples below).

---

## Numeric Type: UFix64 vs UFix128

The canonical interface uses `UFix64` (8 decimal places, max ~1.84 × 10^19). This is verified
from the `DeFiActions.cdc` source and all shipped connectors.

**UFix128** (24 decimal places, Cadence v1.7.0+, available on mainnet) is not used in the
current DeFiActions interface. When it matters:

| Scenario | UFix64 | UFix128 |
|---|---|---|
| FLOW/USD around $0.50–$100 | Sufficient — 8 decimals gives $0.00000001 granularity | Not needed |
| Stablecoin pair (USDC/USDT) near 1.0 | Sufficient | Not needed |
| Long-tail token priced at 10^-9 per USD | May lose precision below 8 decimal places | Preferred |
| Share price of ERC4626 vault (internally uses 18-decimal EVM arithmetic) | ERC4626PriceOracles normalizes to 18 decimals internally and then downcasts to UFix64 at the Cadence boundary | N/A — the downcast occurs inside the implementation |

For the precision decision matrix including integer overflow boundaries, see
[`../../cadence-lang/references/numeric-fixed-point.md`](../../cadence-lang/references/numeric-fixed-point.md).

Current practice: implement to `UFix64` now. If your protocol requires sub-cent precision for
a specific token, wrap the oracle in a custom adapter that normalizes to UFix64 after scaling.

---

## Push vs. Pull Oracles

| Model | Description | Flow pattern |
|---|---|---|
| **Push** | A privileged transaction writes an updated price to contract storage; readers query the stored value | Standard on Cadence — low CU per read, latency governed by how often the updater runs |
| **Pull** | Each read triggers an on-chain query to an external source | BandOracleConnectors is pull-based: every `price()` call invokes `BandOracle.getReferenceData(...)` and withdraws a FlowToken fee from a `Source` |

Flow's emulator and test environments are single-execution contexts, so pull oracles that
make EVM calls (like `ERC4626PriceOracles`) must be invoked inside a transaction (not a
view script) because EVM calls are non-view operations. Design your compositions accordingly:
if the PriceOracle used in an `AutoBalancer` rebalance tick calls EVM, the tick callback must
be a transaction, not a script.

---

## Staleness and Freshness

The `DeFiActions.PriceOracle` interface does **not** mandate a `lastUpdate()` method. Staleness
is an implementation responsibility.

### How the shipped connectors handle it

**BandOracleConnectors.PriceOracle** — constructor accepts `staleThreshold: UInt64?` (seconds).
When set, every `price()` call asserts:

```cadence
let now = UInt64(getCurrentBlock().timestamp)
assert(now < priceData.baseTimestamp + threshold, message: "...stale...")
assert(now < priceData.quoteTimestamp + threshold, message: "...stale...")
```

If `staleThreshold` is `nil`, no staleness check is performed. Revert is the chosen failure
mode because the oracle fee has already been paid; returning stale data silently would be worse.

**ERC4626PriceOracles.PriceOracle** — derives NAV from live EVM state on every call
(`totalAssets / totalShares`). Price is always current as of the block; no separate timestamp
check is needed. Returns `nil` when `totalShares == 0` (vault not yet seeded).

**MockOracle** (test-only) — no staleness checking. Stores mutable prices per token type in
contract storage. Prices are set by a permissionless `setPrice` call — appropriate for testing,
never for production.

### Recommendation for custom implementations

If your oracle reads from a push-based keeper contract, embed a `lastUpdated: UInt64` field in
the implementation and expose a `staleThreshold` constructor parameter. Assert inside `price()`:

```cadence
access(all) fun price(ofToken: Type): UFix64? {
    if let threshold = self.staleThreshold {
        let age = UInt64(getCurrentBlock().timestamp) - MyKeeper.lastUpdated
        assert(age <= threshold,
            message: "Oracle price is \(age)s old, exceeds threshold \(threshold)s")
    }
    return MyKeeper.prices[ofToken]
}
```

---

## Worked Examples

### 1. Minimal compliant implementation — settable price oracle for testing

This is a self-contained oracle that does not depend on IncrementFi or Band. Suitable for unit
tests and as a template for custom push-based implementations.

```cadence
import "FungibleToken"
import "DeFiActions"

/// SettablePriceOracle
///
/// A minimal push-based PriceOracle. An admin address updates prices; readers query them.
/// Includes a staleness guard.
///
/// DO NOT use in production without restricting setPrice to an admin entitlement or resource.
///
access(all) contract SettablePriceOracle {

    access(self) var prices: {Type: UFix64}
    access(self) var lastUpdatedAt: UInt64
    access(self) let unitOfAccountType: Type

    /// PriceOracle implements DeFiActions.PriceOracle
    access(all) struct Oracle : DeFiActions.PriceOracle {
        access(contract) var uniqueID: DeFiActions.UniqueIdentifier?
        access(self) let staleThreshold: UInt64?

        init(staleThreshold: UInt64?, uniqueID: DeFiActions.UniqueIdentifier?) {
            self.staleThreshold = staleThreshold
            self.uniqueID = uniqueID
        }

        access(all) view fun unitOfAccount(): Type {
            return SettablePriceOracle.unitOfAccountType
        }

        access(all) fun price(ofToken: Type): UFix64? {
            if let threshold = self.staleThreshold {
                let age = UInt64(getCurrentBlock().timestamp) - SettablePriceOracle.lastUpdatedAt
                assert(age <= threshold,
                    message: "Price data is \(age)s old; threshold is \(threshold)s")
            }
            if ofToken == self.unitOfAccount() {
                return 1.0   // unit of account prices itself at 1
            }
            return SettablePriceOracle.prices[ofToken]
        }

        access(all) fun getComponentInfo(): DeFiActions.ComponentInfo {
            return DeFiActions.ComponentInfo(
                type: self.getType(),
                id: self.id(),
                innerComponents: []
            )
        }

        access(contract) view fun copyID(): DeFiActions.UniqueIdentifier? {
            return self.uniqueID
        }

        access(contract) fun setID(_ id: DeFiActions.UniqueIdentifier?) {
            self.uniqueID = id
        }
    }

    /// Admin-only price setter — guard with entitlement or resource in production.
    access(all) fun setPrice(forToken: Type, price: UFix64) {
        pre { price > 0.0: "Price must be greater than 0" }
        self.prices[forToken] = price
        self.lastUpdatedAt = UInt64(getCurrentBlock().timestamp)
    }

    init(unitOfAccountIdentifier: String) {
        self.prices = {}
        self.lastUpdatedAt = UInt64(getCurrentBlock().timestamp)
        self.unitOfAccountType = CompositeType(unitOfAccountIdentifier)
            ?? panic("Invalid unitOfAccountIdentifier \(unitOfAccountIdentifier)")
    }
}
```

### 2. Band Oracle connector usage (pull-based, fee-paying)

```cadence
import "FungibleToken"
import "FlowToken"
import "DeFiActions"
import "BandOracleConnectors"

// In a transaction prepare block:
let operationID = DeFiActions.createUniqueIdentifier()

// feeSource must be a Source that provides FlowToken (e.g. a VaultSource wrapping the signer's vault)
let feeSource: {DeFiActions.Source} = ... // provide a Source<FlowToken.Vault>

let oracle = BandOracleConnectors.PriceOracle(
    unitOfAccount: Type<@FlowToken.Vault>(),   // prices denominated in FLOW
    staleThreshold: 3600,                       // revert if data is older than 1 hour
    feeSource: feeSource,
    uniqueID: operationID
)

// Query: price of token X in FLOW
let tokenXType = CompositeType("A.abc123.TokenX.Vault")!
let priceOrNil: UFix64? = oracle.price(ofToken: tokenXType)
let price = priceOrNil ?? panic("Token X price unavailable from BandOracle")
```

### 3. ERC4626 vault share price (cross-VM pull)

```cadence
import "EVM"
import "FlowEVMBridgeConfig"
import "DeFiActions"
import "ERC4626PriceOracles"

// In a script or transaction (non-view — EVM calls are not view):
let vaultAddress = EVM.addressFromString("0xYourERC4626VaultHex")
let underlyingType = Type<@YourToken.Vault>()

let oracle = ERC4626PriceOracles.PriceOracle(
    vault: vaultAddress,
    asset: underlyingType,
    uniqueID: nil
)

// shareType is the bridged Cadence type for the ERC4626 share token
let shareType = FlowEVMBridgeConfig.getTypeAssociated(with: vaultAddress)!
let navPerShare: UFix64? = oracle.price(ofToken: shareType)
// navPerShare == totalAssets / totalShares, normalized from 18-decimal EVM arithmetic
```

### 4. Oracle as slippage guard in a Swapper composition

Consult an oracle to validate that a swap quote is within an acceptable deviation from the
known market price before executing. `priceOracle.unitOfAccount()` must be the same for both
token lookups.

```cadence
// priceOracle.unitOfAccount() == Type<@USDC.Vault>()
let inPrice  = oracle.price(ofToken: swapper.inType())  ?? panic("No price for input token")
let outPrice = oracle.price(ofToken: swapper.outType()) ?? panic("No price for output token")
let expectedOut = (inAmount * inPrice) / outPrice
let quote = swapper.quoteOut(forProvided: inAmount, reverse: false)
let deviation = expectedOut > quote.outAmount
    ? (expectedOut - quote.outAmount) / expectedOut
    : (quote.outAmount - expectedOut) / expectedOut
assert(deviation <= 0.01,
    message: "Quote deviates \(deviation * 100.0)% from oracle — possible manipulation")
```

See [`swapper-interface.md`](swapper-interface.md) for full `quoteIn`/`quoteOut` semantics.

---

## Common Pitfalls

### No staleness check on a push oracle

```cadence
// ❌ Anti-pattern: PriceOracle implementation that returns stored price with no timestamp guard
access(all) fun price(ofToken: Type): UFix64? {
    return MyKeeper.prices[ofToken]  // could be days old
}
```

```cadence
// ✅ Correct: enforce a staleThreshold on every read
access(all) fun price(ofToken: Type): UFix64? {
    let age = UInt64(getCurrentBlock().timestamp) - MyKeeper.lastUpdated
    assert(age <= self.staleThreshold, message: "Stale price: \(age)s old")
    return MyKeeper.prices[ofToken]
}
```

### Unguarded price setter — anyone can manipulate

```cadence
// ❌ Anti-pattern: public setter with no access control
access(all) fun setPrice(forToken: Type, price: UFix64) {
    self.prices[forToken] = price   // any transaction can call this
}
```

```cadence
// ✅ Correct: guard the setter behind a resource held by admin or an entitlement
access(all) resource Admin {
    access(all) fun setPrice(forToken: Type, price: UFix64) {
        MyOracle.prices[forToken] = price
        MyOracle.lastUpdated = UInt64(getCurrentBlock().timestamp)
    }
}
```

### Single oracle as sole liquidation input

```cadence
// ❌ Anti-pattern: liquidation decision depends on one oracle — one compromised source
//    drains the entire protocol
if oracle.price(ofToken: collateralType)! < debtThreshold {
    liquidate(position)
}
```

```cadence
// ✅ Correct: require consensus from at least two independent oracle sources
let priceA = oracleA.price(ofToken: collateralType) ?? panic("Oracle A unavailable")
let priceB = oracleB.price(ofToken: collateralType) ?? panic("Oracle B unavailable")
let deviation = priceA > priceB
    ? (priceA - priceB) / priceA
    : (priceB - priceA) / priceB
assert(deviation <= 0.02, message: "Oracle disagreement exceeds 2% — halt liquidation")
let price = (priceA + priceB) / 2.0   // median of two; use more sources for higher value
if price < debtThreshold { liquidate(position) }
```

### Consumer silently accepts nil price

```cadence
// ❌ Anti-pattern: force-unwrapping hides a nil from an unregistered token
let price = oracle.price(ofToken: someType)!  // panics with an opaque message
```

```cadence
// ✅ Correct: nil-check with a descriptive abort
let price = oracle.price(ofToken: someType)
    ?? panic("PriceOracle returned nil for \(someType.identifier) — token not registered")
```

### Using oracle price for an operation denominated in a different unit

```cadence
// ❌ Anti-pattern: oracle.unitOfAccount() is FLOW but the protocol needs USD prices
//    Using it raw introduces a hidden FLOW/USD risk layer
let flowDenominatedPrice = oracle.price(ofToken: collateral)!
// ...used directly as if it were a USD price
```

```cadence
// ✅ Correct: chain two oracles or check unitOfAccount() before using the value
assert(oracle.unitOfAccount() == Type<@USDC.Vault>(),
    message: "Expected USD-denominated oracle, got \(oracle.unitOfAccount().identifier)")
let usdPrice = oracle.price(ofToken: collateral)!
```

---

## See Also

- [`swapper-interface.md`](swapper-interface.md) — Swappers use PriceOracle as a slippage guard;
  see the quote-validation pattern above for the integration point.
- [`composition-patterns.md`](composition-patterns.md) — How PriceOracle fits into the broader
  Source → Swapper → Sink graph, including AutoBalancer rebalance thresholds.
- [`../../cadence-lang/references/numeric-fixed-point.md`](../../cadence-lang/references/numeric-fixed-point.md) —
  Fix128 / UFix128 precision decision matrix; relevant when evaluating whether UFix64's 8 decimal
  places are sufficient for your price pair.
- [`../../cadence-audit/references/audit-checklist.md`](../../cadence-audit/references/audit-checklist.md) —
  Oracle audit pitfalls checklist: staleness, single-source risk, admin key exposure on price setters.
