# Event Taxonomy in Cadence

Events are the contract's public API for off-chain observers. Indexers (FindLabs, Flowscan, custom GraphQL gateways), dashboards, auditing pipelines, and downstream automations all consume events as the canonical record of what happened on-chain. Bad event design is expensive to undo: once a contract is live, downstream consumers depend on every field name, type, and emission point. Renaming or restructuring an event later forces every indexer, subgraph, and dashboard to migrate in lockstep. Design events for the indexer, not for the writer.

## Naming Conventions

Cadence events are types — the event name becomes the type identifier (`A.<address>.<Contract>.<EventName>`) that indexers query against. Use **PascalCase** in the `<Subject><Verb>` form, past tense (the event records something that has already happened).

| ✅ Good | ❌ Bad | Why |
|---|---|---|
| `TokenMinted` | `MintToken` | Past-tense verb; events record completed state transitions |
| `VaultDeposited` | `DepositEvent` | Drop the `Event` suffix — the type kind is already `event` |
| `OrderFilled` | `fill_order` | snake_case breaks indexer codegen conventions |
| `PoolCreated` | `Pool` | Bare noun is ambiguous; pair with a verb |
| `RoleGranted` / `RoleRevoked` | `RoleChanged` | Separate verbs let indexers join/filter without inspecting payload |
| `FeeUpdated` | `Update` | Subject-less verbs collide across contracts |

Why these conventions matter:

- **Type identifiers feed indexer codegen.** FindLabs and similar tooling generate TypeScript/GraphQL types from event names. PascalCase maps cleanly to GraphQL `type` declarations and TypeScript interfaces.
- **Filter and join performance.** Indexers shard by event type. A single fat `Changed` event forces every consumer to filter in application code; separate verbs (`Granted`, `Revoked`) let the indexer push filtering to the storage layer.
- **Cross-contract correlation.** When every contract follows `<Subject><Verb>`, downstream tooling can build cross-contract dashboards (every `*Deposited` event across DeFi) without per-contract glue.
- **No `Event` suffix.** The Cadence keyword `event` already marks the kind; `TokenMintedEvent` is noise that bloats every query.

## Required Default Events

Every contract should emit a baseline event surface. Indexers treat the absence of these as a signal that the contract is opaque or experimental.

### Resource lifecycle

For each user-facing resource type, emit creation and destruction events:

```cadence
access(all) event NFTCreated(id: UInt64, kind: String)
access(all) event NFTDestroyed(id: UInt64)
```

Emit `Created` inside the function that wraps `create`, not inside the resource's `init()` — see Anti-patterns below. Emit `Destroyed` from a `burnCallback` (FT) or an explicit burn function, not from a `destroy` block (which is being deprecated).

### State transitions

For every state transition that changes user-observable behavior, emit a single past-tense verb event:

```cadence
access(all) event Transferred(id: UInt64, from: Address, to: Address)
access(all) event Approved(id: UInt64, owner: Address, spender: Address)
access(all) event Revoked(id: UInt64, owner: Address, spender: Address)
```

Prefer narrow verbs (`Approved` / `Revoked`) over broad ones (`PermissionChanged`). Each becomes its own indexer stream.

### Admin / configuration changes — emit before+after pairs

Admin actions are the most-audited part of any contract. Emit the previous and new values in the same event so auditors don't have to reconstruct history by walking prior events.

```cadence
access(all) event FeeUpdated(oldFee: UFix64, newFee: UFix64, admin: Address)
access(all) event AdminRotated(oldAdmin: Address, newAdmin: Address)
access(all) event PausedFlipped(oldPaused: Bool, newPaused: Bool, admin: Address)
```

The `admin` address (or `txOrigin` proxy) lets auditors join against signer history. For pause/unpause specifically, emit a single `PausedFlipped` event with both states rather than separate `Paused` / `Unpaused` events — it keeps the audit trail aligned with the storage slot.

## Success-Path vs Failure-Path Emission

Cadence transactions are all-or-nothing: if any `pre`/`post` condition fails, or any function panics, the entire transaction reverts and **every event emitted so far is wiped**. There is no observable failure event from a panicking handler.

```cadence
// ❌ Useless — the panic on the next line wipes this emit
emit OperationStarted(id: id)
panic("invalid state")
```

The off-chain consequence:

- Indexers must treat **absence of the expected success event** as the failure signal.
- A custom `Failed` event emitted just before a `panic` is never delivered — it is rolled back with the transaction.
- The only way to surface a failure on-chain is to **not panic**: return early, set a status field, and emit a `Failed` event in the success path of the wrapping transaction.

This matters most for scheduled transactions, where the scheduler optimistically flips status to `Executed` before calling the handler — see [scheduled-transactions.md](./scheduled-transactions.md) for the full failure-handling state machine.

For non-scheduled flows, the pattern is:

```cadence
// ✅ Failure is a non-panicking branch with its own emit
access(all) fun trySettle(orderId: UInt64): Bool {
    if !self.canSettle(orderId) {
        emit SettlementSkipped(orderId: orderId, reason: "stale price")
        return false
    }
    self.settle(orderId)
    emit OrderSettled(orderId: orderId)
    return true
}
```

## Argument Typing Best Practices

Event arguments must serialize to JSON for off-chain consumers. Treat the argument list as a database schema — every field becomes an indexed column.

**Flatten to scalars.** Stick to types the indexer can store directly: `Address`, `UInt64`, `UFix64`, `Int`, `String`, `Bool`, `Type`, plus arrays and dictionaries of those. These map 1:1 to GraphQL scalars.

```cadence
// ✅ Indexer-friendly scalars
access(all) event OrderFilled(
    orderId: UInt64,
    buyer: Address,
    seller: Address,
    amount: UFix64,
    price: UFix64,
    tokenType: Type
)
```

**Avoid optional structs.** Nested optional structs produce null-laden JSON that breaks SQL joins and confuses GraphQL non-null guarantees. Flatten the struct's fields into the event, or split into two events.

```cadence
// ❌ Optional struct: null fields confuse joins
access(all) event Filled(order: OrderInfo?)

// ✅ Flattened scalars
access(all) event Filled(orderId: UInt64, amount: UFix64, price: UFix64)
```

**Never put resource references in events.** References (`&T`) and resources (`@T`) do not serialize. The indexer cannot reconstruct them and the contract will fail to deploy if you try. Pass the resource's `uuid` or `id` instead.

```cadence
// ❌ Will not compile, and conceptually wrong
access(all) event NFTMoved(nft: &NFT)

// ✅ Pass the identifier
access(all) event NFTMoved(nftID: UInt64, owner: Address)
```

**Use `Type` for type identifiers; don't embed full metadata.** Cadence's `Type` value serializes to a stable identifier string (`A.f8d6e0586b0a20c7.MyToken.Vault`). Don't try to dump a full `RuntimeType` or include resolved metadata views in the event — those belong in a script consumers can call by id.

```cadence
// ✅ Just the type identifier
access(all) event PoolCreated(poolId: UInt64, tokenA: Type, tokenB: Type)
```

**Never use `AnyStruct` parameters.** See Anti-patterns.

## Decision Matrix

| Want to record | Use this event shape |
|---|---|
| Resource born | `<Type>Created(id, owner, ...)` emitted in the wrapping function, not `init()` |
| Resource burned | `<Type>Destroyed(id, ...)` emitted from `burnCallback` or explicit burn |
| Balance transfer (FT) | Rely on `FungibleToken.Withdrawn` / `FungibleToken.Deposited` (auto-emitted by interface post conditions) |
| Balance transfer (NFT) | Rely on `NonFungibleToken.Withdrawn` / `NonFungibleToken.Deposited` |
| Mint / burn (not covered by standard) | `<Token>Minted(amount, to)` / `<Token>Burned(amount, from)` |
| Permission grant | `<Capability>Granted(target, holder, by)` |
| Permission revoke | `<Capability>Revoked(target, holder, by)` |
| Admin field update | `<Field>Updated(oldValue, newValue, admin)` |
| Pause toggle | `PausedFlipped(oldPaused, newPaused, admin)` |
| Non-fatal failure on success path | `<Action>Skipped(id, reason)` then return false |
| Contract deploy / upgrade | `ContractInitialized()` (no args) emitted from contract `init()` |

The contract `init()` is the one exception where emission is allowed — it fires at deploy time and indexers explicitly listen for `ContractInitialized` to bootstrap their state.

## Testing Events

Asserting that the right events fired with the right payloads is the only way to verify the public contract surface end-to-end. See [../../cadence-testing/references/events-and-logs.md](../../cadence-testing/references/events-and-logs.md) for `Test.eventsOfType` patterns, payload matchers, and how to test the absence of an event (the standard failure signal).

## Anti-Patterns

### 1. Emitting from `init()` of structs or resources

Events emitted inside a struct or resource `init()` only fire during the construction call. They do not give the indexer a usable signal: structs are values (constructed in many places), and resource `init()` runs inside the creator function — which is the place that should emit.

```cadence
// ❌ Fires every construction, including in tests and scripts that never write to chain
access(all) resource NFT {
    init(id: UInt64) {
        self.id = id
        emit NFTCreated(id: id)
    }
}

// ✅ Emit from the function that mints and returns the resource
access(all) fun mintNFT(id: UInt64, recipient: Address): @NFT {
    emit NFTCreated(id: id, recipient: recipient)
    return <- create NFT(id: id)
}
```

The contract `init()` is the one allowed exception (`emit ContractInitialized()`) because it fires once at deploy and indexers depend on it.

### 2. `AnyStruct` parameters

`AnyStruct` is the type-erased escape hatch. Indexers cannot decode it without contract-specific glue, and the field shows up as opaque bytes in every dashboard.

```cadence
// ❌ Indexer-hostile
access(all) event Generic(payload: AnyStruct)

// ✅ Concrete typed fields
access(all) event PriceUpdated(symbol: String, oldPrice: UFix64, newPrice: UFix64)
```

If you genuinely have polymorphic payloads, split into multiple typed events rather than one `AnyStruct` event.

### 3. Silent state changes

Every state mutation that an off-chain consumer cares about must be paired with an emit in the same code path. A function that updates storage without emitting leaves the on-chain state correct but the indexer permanently desynced — and the only fix is a contract upgrade plus a full re-index.

```cadence
// ❌ Silent mutation
access(Admin) fun setFee(newFee: UFix64) {
    self.fee = newFee
}

// ✅ Mutation paired with event
access(Admin) fun setFee(newFee: UFix64) {
    let oldFee = self.fee
    self.fee = newFee
    emit FeeUpdated(oldFee: oldFee, newFee: newFee, admin: self.account.address)
}
```

Rule of thumb: if a getter would return a different value after the call, an event must fire.

### 4. Duplicating standard events

`FungibleToken` and `NonFungibleToken` interface post conditions auto-emit `Withdrawn` / `Deposited`. Emitting a custom `TokensWithdrawn` from inside `withdraw()` produces two events per transfer and confuses every standard indexer.

### 5. Events as control flow

Events are write-only from the contract's perspective — there is no on-chain reader. Do not design contract logic around events being consumed by another contract in the same transaction (they can't be). Use returned values, capabilities, or shared resources for in-contract signaling.

## Common Pitfalls

- **Renaming an event after deploy.** The fully qualified type identifier (`A.<addr>.<Contract>.<EventName>`) changes, breaking every consumer. Treat event names as a permanent API.
- **Reordering or retyping fields.** Field positions and types are part of the schema. Add new fields by emitting a new event, not by mutating an existing one.
- **Forgetting the admin address.** Auditors filter admin events by signer. Always include the acting principal (`admin: Address`) on configuration changes.
- **Emitting before the mutation.** If a `panic` in a later line of the same function reverts the tx, the emit is wiped — order doesn't save you, but emitting after the mutation makes the intent obvious during review.
- **Relying on event ordering across transactions.** Within one transaction, emit order is preserved. Across transactions, only block height + transaction index are ordered; events within a block from different txs interleave.
- **Skipping events on the failure branch of a non-panicking function.** If you return `false`, emit a `<Action>Skipped` event so the indexer can distinguish "not called" from "called and declined".
