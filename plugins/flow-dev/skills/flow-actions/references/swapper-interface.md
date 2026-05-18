# Swapper Interface

A Swapper is a `DeFiActions` struct interface that transforms a `FungibleToken.Vault` of one type
into a `FungibleToken.Vault` of a different type, typically by routing through a DEX, AMM, or
fixed-rate exchange. Unlike Source and Sink — which describe one-sided flows in or out of a
protocol — a Swapper owns both ends of a token pair, making token ordering the central concern:
the interface explicitly names one side `inType()` (token accepted) and the other `outType()` (token
produced), and the `reverse: Bool` flag on both quote methods allows the same struct to price the
swap in either direction without constructing a new Swapper. This bidirectionality is what makes
Swappers the critical connector between Source and Sink in a composition chain: a wrong-ordered
Swapper silently corrupts the entire pipeline at runtime. When composing connectors, prefer
`SwapConnectors.SwapSource` or `SwapConnectors.SwapSink` over calling a Swapper directly; call a
Swapper directly only when building or testing a standalone price query.

> **Beta notice:** `DeFiActions` is in beta on Testnet and Mainnet. Interfaces and event
> signatures may change before final release. Monitor
> [`onflow/FlowActions`](https://github.com/onflow/FlowActions) for breaking changes.

---

## Interface Definition

```cadence
// DeFiActions.cdc (canonical source)

/// Quote
///
/// Returned by quoteIn and quoteOut. A Quote with inAmount==outAmount==0.0 signals that
/// no price is available — callers must treat 0.0 values as an unavailable quote, not a
/// zero-cost swap.
///
access(all) struct interface Quote {
    access(all) let inType: Type    // pre-swap vault type
    access(all) let outType: Type   // post-swap vault type
    access(all) let inAmount: UFix64
    access(all) let outAmount: UFix64
}

/// Swapper
///
access(all) struct interface Swapper : IdentifiableStruct {

    // --- Type accessors ---

    /// Token type this Swapper consumes (token0 by convention).
    access(all) view fun inType(): Type

    /// Token type this Swapper produces (token1 by convention).
    access(all) view fun outType(): Type

    // --- Quote methods ---

    /// Returns how many input tokens are needed to receive `forDesired` output tokens.
    ///   reverse=false  ->  inType()  -> outType()  (normal direction)
    ///   reverse=true   ->  outType() -> inType()   (reverse direction)
    /// Returns inAmount==outAmount==0.0 when the price is unavailable.
    access(all) fun quoteIn(forDesired: UFix64, reverse: Bool): {Quote}

    /// Returns how many output tokens are delivered when `forProvided` input tokens are supplied.
    ///   reverse=false  ->  inType()  -> outType()
    ///   reverse=true   ->  outType() -> inType()
    /// Returns inAmount==outAmount==0.0 when the price is unavailable.
    access(all) fun quoteOut(forProvided: UFix64, reverse: Bool): {Quote}

    // --- Execution methods ---

    /// Consumes `inVault` (must be type inType()) and returns a vault of type outType().
    /// Pre-condition enforced by the interface:
    ///   inVault.getType() == self.inType()
    ///   (quote?.inType ?? inVault.getType()) == inVault.getType()
    /// Post-condition enforced by the interface:
    ///   result.getType() == self.outType()
    ///   emits DeFiActions.Swapped event
    /// NOTE: providing a Quote does NOT guarantee the result vault balance matches quote.outAmount.
    access(all) fun swap(quote: {Quote}?, inVault: @{FungibleToken.Vault}): @{FungibleToken.Vault}

    /// Consumes `residual` (must be type outType()) and returns a vault of type inType().
    /// This is the reverse direction; useful for returning unused tokens after a SwapSink partial fill.
    /// Pre-condition enforced by the interface:
    ///   residual.getType() == self.outType()
    /// Post-condition enforced by the interface:
    ///   result.getType() == self.inType()
    ///   emits DeFiActions.Swapped event
    ///
    /// **Note: `Swapped` is emitted unconditionally**, even when both `inAmount` and `outAmount`
    /// are `0.0`. This is asymmetric with `Withdrawn` (Source) and `Deposited` (Sink), which both
    /// suppress zero-amount events via conditional helpers. Consumers indexing `Swapped` events
    /// must filter zero-amount entries themselves if they want symmetry with the Source/Sink event
    /// streams.
    access(all) fun swapBack(quote: {Quote}?, residual: @{FungibleToken.Vault}): @{FungibleToken.Vault}
}
```

### Method summary

| Method | Mutating | Purpose |
|--------|----------|---------|
| `inType()` | no (view) | Token type the swapper accepts |
| `outType()` | no (view) | Token type the swapper produces |
| `quoteIn(forDesired:reverse:)` | yes | Estimate input needed to receive a desired output |
| `quoteOut(forProvided:reverse:)` | yes | Estimate output delivered for a given input |
| `swap(quote:inVault:)` | yes | Execute forward swap: inType → outType |
| `swapBack(quote:residual:)` | yes | Execute reverse swap: outType → inType |

Six methods total; all six are documented below.

---

## Token Ordering Rules

This is the most subtle aspect of the Swapper interface. Every Swapper has a fixed, statically
declared direction: `inType()` is the token it consumes and `outType()` is the token it produces.
This direction is **not interchangeable** — `swap()` panics at the interface level if the provided
vault type does not match `inType()`.

### The `reverse: Bool` parameter

`reverse` does **not** change which token `swap()` accepts. It only changes how the quote methods
interpret the pair:

| `reverse` | Quote `inType` | Quote `outType` | Meaning |
|-----------|----------------|-----------------|---------|
| `false` | `swapper.inType()` | `swapper.outType()` | Price the normal direction |
| `true`  | `swapper.outType()` | `swapper.inType()` | Price the reverse direction |

`swapBack()` already executes in the reverse direction unconditionally. `reverse: true` on
`quoteIn`/`quoteOut` lets callers price the reverse direction **before** calling `swapBack()`,
without needing a separate Swapper struct.

### Establishing the correct ordering before construction

When building a Swapper for an AMM pair that has a canonical token0/token1 ordering (as in
IncrementFi's pool contracts), the caller must determine whether the Source token matches token0
before constructing the Swapper. The safe pattern is:

```cadence
// Determine the pool's canonical pair ordering
let token0Type = /* pool.token0Type */
let token1Type = /* pool.token1Type */
let sourceType  = source.getSourceType()

// Reverse token arguments if the source produces token1 rather than token0
let reverse = sourceType != token0Type
let swapper = MyProtocolSwapper(
    inVault:  reverse ? token1Type : token0Type,
    outVault: reverse ? token0Type : token1Type,
    uniqueID: operationID
)
```

The `reverse` variable here is a **construction-time decision** unrelated to the `reverse: Bool`
parameter on quote methods. Once the Swapper is constructed, its token ordering is fixed.

### ✅ Correct: caller controls ordering

```cadence
// Source produces FUSD; pair is (FLOW, FUSD). Reverse so FUSD becomes inType.
let reverse = source.getSourceType() != token0Type   // true
let swapper = PoolSwapper(
    inVault:  reverse ? token1Type : token0Type,     // FUSD
    outVault: reverse ? token0Type : token1Type,     // FLOW
    uniqueID: operationID
)
// swapper.inType() == FUSD, swapper.outType() == FLOW
```

### ❌ Anti-pattern: Swapper silently reverses order internally

```cadence
// BAD: Swapper constructor detects source type and silently swaps token0/token1
access(all) struct AutoReverseSwapper : DeFiActions.Swapper {
    init(sourceType: Type, pool: ...) {
        // infers ordering from source — caller loses control
        if sourceType == pool.token0Type {
            self.inVault = pool.token0Type
            self.outVault = pool.token1Type
        } else {
            self.inVault = pool.token1Type   // silently flipped
            self.outVault = pool.token0Type
        }
    }
}
// Caller cannot reason about the ordering from the outside. Composition breaks when
// another caller passes a different source type to the same Swapper.
```

---

## Quote vs. Execution Guarantees

### What a Quote is

A `{DeFiActions.Quote}` is a **non-binding estimate**. The interface's pre- and post-conditions
enforce that types match, but they do **not** enforce that `result.balance >= quote.outAmount`.
The canonical contract comment states this explicitly: "providing a Quote does not guarantee the
fulfilled swap will enforce the quote's defined outAmount."

### Protecting against slippage

Because the Quote is advisory, callers must add their own slippage guard as a `post` condition on
the enclosing transaction:

```cadence
// In the Cadence transaction:
let startingBalance = self.targetVault.balance

execute {
    let result <- swapper.swap(quote: cachedQuote, inVault: <-inputVault)
    // enforce minimum output directly on the returned vault
    assert(
        result.balance >= self.minExpectedOut,
        message: "Swap output \(result.balance) is below minimum \(self.minExpectedOut)"
    )
    self.targetVault.deposit(from: <-result)
}

post {
    self.targetVault.balance >= startingBalance + self.minExpectedOut:
        "Post-swap balance below minimum acceptable output"
}
```

### Stale quote handling

Quotes are generated at a point in time against on-chain AMM state. By the time `swap()` is
called — even within the same transaction — pool state may have changed due to concurrent
transactions in the same block. Best practices:

1. Compute `minExpectedOut` from the quote with an acceptable slippage tolerance (e.g. 1%), then
   assert post-execution rather than relying on the quote alone.
2. When passing a cached quote to `swap()`, the implementation is free to ignore it (as
   `IncrementFiSwapConnectors.Swapper` does) and re-derive `amountOutMin` at execution time.
3. Never use a quote obtained in a separate transaction as a guarantee — it may be many blocks stale.

### Zero-quote signals unavailability

A quote with `inAmount == 0.0 && outAmount == 0.0` means the implementation could not produce an
estimate (e.g. pool liquidity was zero or the path was invalid). Callers must check for this:

```cadence
let q = swapper.quoteOut(forProvided: amount, reverse: false)
if q.inAmount == 0.0 || q.outAmount == 0.0 {
    // abort or try an alternate path
    panic("Swapper returned unavailable quote for amount \(amount)")
}
```

---

## Minimal Compliant Implementation — Fixed-Rate Swapper

This is a protocol-agnostic, constant-rate Swapper useful in tests and simple bridging scenarios.
It does not interact with any DEX or AMM; it converts at a fixed ratio.

```cadence
import "FungibleToken"
import "DeFiActions"
import "SwapConnectors"

/// FixedRateSwapper
///
/// Swaps from one FungibleToken vault type to another at a fixed exchange rate.
/// Useful as a test fixture or for wrapping a 1:1 token bridge.
///
access(all) struct FixedRateSwapper : DeFiActions.Swapper {

    /// Token this swapper accepts
    access(self) let inVault: Type
    /// Token this swapper produces
    access(self) let outVault: Type
    /// Output tokens delivered per input token (e.g. 2.5 means 1 inToken -> 2.5 outTokens)
    access(all) let rate: UFix64
    access(contract) var uniqueID: DeFiActions.UniqueIdentifier?

    init(
        inVault: Type,
        outVault: Type,
        rate: UFix64,
        uniqueID: DeFiActions.UniqueIdentifier?
    ) {
        pre {
            inVault.isSubtype(of: Type<@{FungibleToken.Vault}>()):
                "inVault must be a FungibleToken.Vault subtype"
            outVault.isSubtype(of: Type<@{FungibleToken.Vault}>()):
                "outVault must be a FungibleToken.Vault subtype"
            inVault != outVault:
                "inVault and outVault must be different types"
            rate > 0.0: "rate must be greater than 0.0"
        }
        self.inVault  = inVault
        self.outVault = outVault
        self.rate     = rate
        self.uniqueID = uniqueID
    }

    access(all) fun getComponentInfo(): DeFiActions.ComponentInfo {
        return DeFiActions.ComponentInfo(
            type: self.getType(),
            id: self.id(),
            innerComponents: []
        )
    }

    access(contract) view fun copyID(): DeFiActions.UniqueIdentifier? { return self.uniqueID }
    access(contract) fun setID(_ id: DeFiActions.UniqueIdentifier?)   { self.uniqueID = id }

    access(all) view fun inType(): Type  { return self.inVault }
    access(all) view fun outType(): Type { return self.outVault }

    /// quoteIn: how many input tokens are needed to produce `forDesired` output tokens?
    access(all) fun quoteIn(forDesired: UFix64, reverse: Bool): {DeFiActions.Quote} {
        let effectiveRate = reverse ? (1.0 / self.rate) : self.rate
        // inAmount = outDesired / rate
        let inAmount = forDesired / effectiveRate
        return SwapConnectors.BasicQuote(
            inType:    reverse ? self.outVault : self.inVault,
            outType:   reverse ? self.inVault  : self.outVault,
            inAmount:  inAmount,
            outAmount: forDesired
        )
    }

    /// quoteOut: how many output tokens are produced from `forProvided` input tokens?
    access(all) fun quoteOut(forProvided: UFix64, reverse: Bool): {DeFiActions.Quote} {
        let effectiveRate = reverse ? (1.0 / self.rate) : self.rate
        let outAmount = forProvided * effectiveRate
        return SwapConnectors.BasicQuote(
            inType:    reverse ? self.outVault : self.inVault,
            outType:   reverse ? self.inVault  : self.outVault,
            inAmount:  forProvided,
            outAmount: outAmount
        )
    }

    /// swap: convert inVault tokens to outVault tokens at the fixed rate.
    /// The caller is responsible for minting/unlocking outVault tokens in a real implementation.
    access(all) fun swap(quote: {DeFiActions.Quote}?, inVault: @{FungibleToken.Vault}): @{FungibleToken.Vault} {
        // In a real fixed-rate bridge, you would call the bridge contract here.
        // For a test fixture, return an empty vault of outType with the correct balance.
        // NOTE: The interface post-condition enforces result.getType() == self.outType().
        panic("FixedRateSwapper.swap: implement bridge call to produce \(self.outVault.identifier)")
    }

    /// swapBack: convert outVault tokens back to inVault tokens.
    access(all) fun swapBack(quote: {DeFiActions.Quote}?, residual: @{FungibleToken.Vault}): @{FungibleToken.Vault} {
        panic("FixedRateSwapper.swapBack: implement reverse bridge call to produce \(self.inVault.identifier)")
    }
}
```

**Note on `swap()` and `swapBack()` bodies:** A Swapper's execution methods must interact with an
external protocol to mint or unlock the output vault. The above struct shows the structure and
quote logic fully; the `panic` stubs in `swap`/`swapBack` mark where a caller would invoke their
bridge contract (e.g. `ERC4626`, an IncrementFi router, or a wrapped-FLOW contract). For a true
test fixture, replace the panic with a call to `FungibleToken.Vault.withdraw` from a pre-funded
test vault held in the struct.

---

## Composition Patterns

### Canonical flow: Source → Swapper → Sink

The standard pattern wraps the Swapper in either `SwapConnectors.SwapSource` or
`SwapConnectors.SwapSink` rather than calling it directly:

```cadence
// Source → SwapSource(Swapper + Source) → Sink
// SwapSource handles the withdrawAvailable → swap pipeline automatically.

let operationID = DeFiActions.createUniqueIdentifier()

let baseSource = MyProtocolSource(pool: poolRef, uniqueID: operationID)

// Token ordering: ensure swapper.inType() == baseSource.getSourceType()
let swapper = IncrementFiSwapConnectors.Swapper(
    path: ["A.xxx.TokenA", "A.xxx.TokenB"],
    inVault: Type<@TokenA.Vault>(),
    outVault: Type<@TokenB.Vault>(),
    uniqueID: operationID
)
// SwapSource enforces: source.getSourceType() == swapper.inType()
let swapSource = SwapConnectors.SwapSource(
    swapper: swapper,
    source: baseSource,
    uniqueID: operationID
)
// swapSource.getSourceType() == swapper.outType()  (TokenB)

let sink = MyProtocolSink(pool: poolRef, uniqueID: operationID)
// SwapSink enforces: swapper.outType() == sink.getSinkType()

// In execute:
let vault <- swapSource.withdrawAvailable(maxAmount: sink.minimumCapacity())
sink.depositCapacity(from: &vault as auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
assert(vault.balance == 0.0, message: "Residual after deposit")
destroy vault
```

### Type-check at composition boundary

`SwapConnectors.SwapSource` and `SwapSink` both assert type compatibility at construction:

```cadence
// SwapSource init pre-condition (from SwapConnectors.cdc):
pre {
    source.getSourceType() == swapper.inType():
    "Source outputs \(source.getSourceType().identifier) but Swapper takes \(swapper.inType().identifier)"
}

// SwapSink init pre-condition (from SwapConnectors.cdc):
pre {
    swapper.outType() == sink.getSinkType():
    "Swapper outputs \(swapper.outType().identifier) but Sink takes \(sink.getSinkType().identifier)"
}
```

These pre-conditions fire at **construction time** (in `prepare`), before any vault movement.
This is by design — it makes token-type mismatches fail early with a clear error rather than at
the point of vault transfer in `execute`.

### SequentialSwapper for multi-hop paths

When a single Swapper cannot serve the full route (e.g., TokenA → TokenB → TokenC), use
`SwapConnectors.SequentialSwapper`:

```cadence
let hop1 = IncrementFiSwapConnectors.Swapper(
    path: ["A.xxx.TokenA", "A.xxx.TokenB"],
    inVault: Type<@TokenA.Vault>(),
    outVault: Type<@TokenB.Vault>(),
    uniqueID: operationID
)
let hop2 = IncrementFiSwapConnectors.Swapper(
    path: ["A.xxx.TokenB", "A.xxx.TokenC"],
    inVault: Type<@TokenB.Vault>(),
    outVault: Type<@TokenC.Vault>(),
    uniqueID: operationID
)
// SequentialSwapper validates hop1.outType() == hop2.inType() at construction
let multiHop = SwapConnectors.SequentialSwapper(
    swappers: [hop1, hop2],
    uniqueID: operationID
)
// multiHop.inType() == TokenA.Vault, multiHop.outType() == TokenC.Vault
```

`SequentialSwapper` exposes the same `{DeFiActions.Swapper}` interface, so it can be passed
anywhere a single-hop Swapper is accepted.

**`MultiSwapper`** is the sibling of `SequentialSwapper` in `SwapConnectors.cdc`. Where `SequentialSwapper` chains multiple Swappers in sequence (token A → token B → token C), `MultiSwapper` holds multiple Swappers that each route the SAME `inType → outType` pair through different paths and picks the optimal one at quote time. Use `SequentialSwapper` when no single Swapper covers the pair you need; use `MultiSwapper` when several Swappers do cover it but their rates differ.

---

## Anti-Patterns and Common Pitfalls

### 1. ❌ Ignoring the token-type boundary when composing manually

```cadence
// BAD: composing without checking types
let vault <- source.withdrawAvailable(maxAmount: 100.0)
// vault is TokenA; swapper.inType() is TokenB — swap() will panic at the interface pre-condition
let out <- swapper.swap(quote: nil, inVault: <-vault)
```

```cadence
// GOOD: verify before calling, or use SwapSource which enforces this at construction
assert(
    source.getSourceType() == swapper.inType(),
    message: "Source/Swapper type mismatch"
)
```

### 2. ❌ Trusting a stale quote as an output guarantee

```cadence
// BAD: using quote.outAmount as a guarantee
let q = swapper.quoteOut(forProvided: 100.0, reverse: false)
let out <- swapper.swap(quote: q, inVault: <-inputVault)
// No assertion — out.balance may be less than q.outAmount due to slippage or pool movement
destroy out  // tokens lost silently
```

```cadence
// GOOD: assert minimum acceptable output after swap
let q = swapper.quoteOut(forProvided: 100.0, reverse: false)
let minOut = q.outAmount * 0.99   // 1% slippage tolerance
let out <- swapper.swap(quote: q, inVault: <-inputVault)
assert(out.balance >= minOut, message: "Output \(out.balance) below slippage floor \(minOut)")
```

### 3. ❌ Using `reverse: Bool` on quotes without matching direction in swap/swapBack

```cadence
// BAD: quoting in reverse but then calling swap() instead of swapBack()
let q = swapper.quoteIn(forDesired: 50.0, reverse: true)  // priced outType -> inType
// ...later...
let out <- swapper.swap(quote: q, inVault: <-outTypeVault)
// PANIC: swap() pre-condition requires inVault.getType() == swapper.inType()
// but outTypeVault.getType() == swapper.outType()
```

```cadence
// GOOD: match the swap call direction to the quote direction
let q = swapper.quoteIn(forDesired: 50.0, reverse: true)  // outType -> inType
let result <- swapper.swapBack(quote: q, residual: <-outTypeVault)  // correct
```

### 4. ❌ Constructing SwapSource or SwapSink with mismatched type boundaries

```cadence
// BAD: Swapper.outType() != Sink.getSinkType()
let swapper = MySwapper(inVault: TokenA, outVault: TokenB, ...)
let sink    = MySink(/* accepts TokenC */)
// This panics at SwapSink construction:
let swapSink = SwapConnectors.SwapSink(swapper: swapper, sink: sink, uniqueID: id)
// "Swapper outputs TokenB but Sink takes TokenC"
```

Fix: verify `swapper.outType() == sink.getSinkType()` before constructing `SwapSink`, or ensure
the Swapper and Sink are built from the same pool's token types.

### 5. ❌ Re-quoting between quoteIn and swap without re-validating the zero-quote case

```cadence
// BAD: quote is called in one part of prepare, swap in execute — pool may have moved
let q = swapper.quoteOut(forProvided: amount, reverse: false)
// ... many operations later in execute ...
let out <- swapper.swap(quote: q, inVault: <-vault)
// q is now stale; out.balance may be 0.0 or much less than q.outAmount
```

Fix: add a post-execution assertion on `out.balance` with your slippage tolerance. Never rely on
the Quote's `outAmount` as the actual received amount.

### 6. ❌ Sharing a single Swapper struct instance across multiple UniqueIdentifiers

Each composition chain should use a single `UniqueIdentifier` created via
`DeFiActions.createUniqueIdentifier()`. A Swapper struct holds a reference to one `UniqueIdentifier`
at a time. If you share a Swapper struct across two compositions by passing it the same struct
value, both chains share the identifier — `Swapped` events from one chain will be attributed to
the other.

```cadence
// BAD: two compositions sharing one Swapper struct value (and therefore one ID)
let id1 = DeFiActions.createUniqueIdentifier()
let id2 = DeFiActions.createUniqueIdentifier()
let sharedSwapper = MySwapper(..., uniqueID: id1)
let source1 = SwapConnectors.SwapSource(swapper: sharedSwapper, source: src1, uniqueID: id1)
let source2 = SwapConnectors.SwapSource(swapper: sharedSwapper, source: src2, uniqueID: id2)
// source2's Swapped events carry id1, not id2 — traceability is broken
```

Fix: construct a separate Swapper instance for each composition chain.

---

## Cross-references

- [source-interface.md](source-interface.md) — `getSourceType()`, `minimumAvailable()`,
  `withdrawAvailable()` weak-guarantee semantics and type validation requirements
- [sink-interface.md](sink-interface.md) — `getSinkType()`, `minimumCapacity()`,
  `depositCapacity()` liveness philosophy and residual-vault pattern
- [composition-patterns.md](composition-patterns.md) — canonical Source → Swapper → Sink
  composition, `SwapSource`/`SwapSink` wrappers, token-order reversal decision, and
  `minimumAvailable`/`minimumCapacity` sizing rules
