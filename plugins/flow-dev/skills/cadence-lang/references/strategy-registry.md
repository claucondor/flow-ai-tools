# Strategy Registry Pattern

A **strategy registry** is a contract that maintains a whitelist of approved
"strategy" capabilities — each pointing to a resource that conforms to a shared
`Strategy` interface and is entitled to execute one well-defined action (swap,
unwind, rebalance, fulfill an intent). The registry decides which strategies
are valid; users (or an executor) opt in to call them. This pattern underpins
modular DeFi (pluggable AMM routers, lending unwinders), intent systems
(multiple competing fulfillers for the same outcome), and governance over an
*executable set* without hardcoding addresses into every consumer. It is hard
to build correctly without Cadence's capability primitives because raw
addresses give you no way to verify the *shape* of the code on the other side
— capabilities give you a typed handle the registry can borrow-check before
storing.

Cross-links:

- Admin-via-capability (multi-sig admins) — [admin-via-capability.md](admin-via-capability.md)
- Event taxonomy (flat scalar arguments) — [event-taxonomy.md](event-taxonomy.md)
- Resource state machines (Active / Paused / Deprecated / Revoked) — [resource-state-machines.md](resource-state-machines.md)
- Capability fundamentals (`check`, `borrow`, controller revocation) — [capabilities.md](capabilities.md)
- Entitlements on strategy entry points — [entitlements.md](entitlements.md)

## Four ingredients

1. **A `Strategy` resource interface.** Every approved strategy conforms;
   declares the entitled `run` entry point and a `view` `describe()`.
2. **A registry resource** with `access(self) var strategies: {UInt64: Entry}`
   keyed by an internal strategy ID. Each `Entry` stores the capability, the
   provider address, the typed interface name, and the lifecycle phase.
3. **Admin-only mutations** (`addStrategy`, `revokeStrategy`, `freezeStrategy`,
   `deprecateStrategy`) guarded by entitlements, with capability-type
   validation *at registration time* — the central security property.
4. **Lifecycle events** emitted on every mutation, flat scalar arguments only
   (no nested structs) so indexers can filter cheaply.

## Worked example: intent-based executor

A user submits an `Intent` ("swap 100 FLOW for at least 95 USDC"). The
registry holds capabilities to vetted strategies — `NoOpStrategy` (sanity /
testing), `SwapStrategy` (calls a DEX), and so on. An executor borrows the
registry, picks a strategy by ID, and calls `strategy.run(intent: ...)`.

```cadence
import "FungibleToken"

access(all) contract StrategyRegistry {

    access(all) entitlement Admin       // add / revoke / freeze
    access(all) entitlement Execute     // call run() on a Strategy ref

    access(all) enum Phase: UInt8 {
        access(all) case Active        // executor may call
        access(all) case Paused        // temporary freeze; can return to Active
        access(all) case Deprecated    // no new calls; existing users finishing up
        access(all) case Revoked       // terminal; entry is dead
    }

    access(all) resource interface Strategy {
        access(all) view fun describe(): String
        access(Execute) fun run(intent: &Intent): @{FungibleToken.Vault}
    }

    access(all) struct Intent {
        access(all) let inputType: Type
        access(all) let inputAmount: UFix64
        access(all) let outputType: Type
        access(all) let minOutput: UFix64
        init(inputType: Type, inputAmount: UFix64, outputType: Type, minOutput: UFix64) {
            self.inputType = inputType
            self.inputAmount = inputAmount
            self.outputType = outputType
            self.minOutput = minOutput
        }
    }

    access(all) struct Entry {
        access(all) let id: UInt64
        access(all) let provider: Address
        access(all) let interfaceName: String   // human-readable type id
        access(all) let cap: Capability<auth(Execute) &{Strategy}>
        access(all) var phase: Phase
        init(
            id: UInt64,
            provider: Address,
            interfaceName: String,
            cap: Capability<auth(Execute) &{Strategy}>
        ) {
            self.id = id
            self.provider = provider
            self.interfaceName = interfaceName
            self.cap = cap
            self.phase = Phase.Active
        }
        access(contract) fun setPhase(_ p: Phase) { self.phase = p }
    }

    // Lifecycle events — flat scalar args (see event-taxonomy.md)
    access(all) event StrategyAdded(id: UInt64, provider: Address, interfaceName: String)
    access(all) event StrategyRevoked(id: UInt64, provider: Address, interfaceName: String)
    access(all) event StrategyFrozen(id: UInt64, provider: Address, interfaceName: String)
    access(all) event StrategyResumed(id: UInt64, provider: Address, interfaceName: String)
    access(all) event StrategyDeprecated(id: UInt64, provider: Address, interfaceName: String)
    access(all) event StrategyExecuted(id: UInt64, provider: Address, caller: Address, output: UFix64)

    access(all) resource Registry {
        access(self) var nextID: UInt64
        access(self) var strategies: {UInt64: Entry}
        init() { self.nextID = 0; self.strategies = {} }

        // cap.check() validates the capability resolves to a resource
        // conforming to {Strategy}. Without this, an admin could register
        // a capability that satisfies the type slot syntactically but does
        // the wrong thing at runtime.
        access(Admin) fun addStrategy(
            cap: Capability<auth(Execute) &{Strategy}>,
            interfaceName: String
        ): UInt64 {
            pre {
                cap.check(): "strategy capability does not resolve to a valid &{Strategy}"
                interfaceName.length > 0: "interfaceName required"
            }
            let id = self.nextID
            self.nextID = self.nextID + 1
            self.strategies[id] = Entry(
                id: id,
                provider: cap.address,
                interfaceName: interfaceName,
                cap: cap
            )
            emit StrategyAdded(id: id, provider: cap.address, interfaceName: interfaceName)
            return id
        }

        access(Admin) fun freezeStrategy(id: UInt64) {
            let e = (&self.strategies[id] as &Entry?) ?? panic("strategy \(id) not registered")
            assert(e.phase == Phase.Active, message: "strategy \(id) not Active")
            e.setPhase(Phase.Paused)
            emit StrategyFrozen(id: id, provider: e.provider, interfaceName: e.interfaceName)
        }

        access(Admin) fun resumeStrategy(id: UInt64) {
            let e = (&self.strategies[id] as &Entry?) ?? panic("strategy \(id) not registered")
            assert(e.phase == Phase.Paused, message: "strategy \(id) not Paused")
            e.setPhase(Phase.Active)
            emit StrategyResumed(id: id, provider: e.provider, interfaceName: e.interfaceName)
        }

        access(Admin) fun deprecateStrategy(id: UInt64) {
            let e = (&self.strategies[id] as &Entry?) ?? panic("strategy \(id) not registered")
            assert(
                e.phase == Phase.Active || e.phase == Phase.Paused,
                message: "strategy \(id) cannot deprecate from phase \(e.phase.rawValue)"
            )
            e.setPhase(Phase.Deprecated)
            emit StrategyDeprecated(id: id, provider: e.provider, interfaceName: e.interfaceName)
        }

        // Revocation is terminal; always emit even if already revoked,
        // so off-chain consumers see a definitive lifecycle close.
        access(Admin) fun revokeStrategy(id: UInt64) {
            let e = (&self.strategies[id] as &Entry?) ?? panic("strategy \(id) not registered")
            e.setPhase(Phase.Revoked)
            emit StrategyRevoked(id: id, provider: e.provider, interfaceName: e.interfaceName)
        }

        access(all) view fun getEntry(id: UInt64): Entry? { return self.strategies[id] }
        access(all) view fun listIDs(): [UInt64] { return self.strategies.keys }

        // The single source of truth for "is this callable" is Entry.phase.
        // Do NOT re-derive callability from events.
        access(all) fun run(id: UInt64, intent: &Intent, caller: Address): @{FungibleToken.Vault} {
            let e = self.strategies[id] ?? panic("strategy \(id) not registered")
            assert(e.phase == Phase.Active,
                message: "strategy \(id) not Active (phase=\(e.phase.rawValue))")
            let ref = e.cap.borrow() ?? panic("strategy \(id) capability no longer borrowable")
            let out <- ref.run(intent: intent)
            let amount = out.balance
            emit StrategyExecuted(id: id, provider: e.provider, caller: caller, output: amount)
            return <-out
        }
    }

    access(all) let RegistryStoragePath: StoragePath
    access(all) let RegistryPublicPath:  PublicPath

    init() {
        self.RegistryStoragePath = /storage/strategyRegistry
        self.RegistryPublicPath  = /public/strategyRegistry
        let reg <- create Registry()
        self.account.storage.save(<-reg, to: self.RegistryStoragePath)
        // Public cap exposes only the un-entitled &Registry — no Admin leak.
        let pubCap = self.account.capabilities.storage
            .issue<&Registry>(self.RegistryStoragePath)
        self.account.capabilities.publish(pubCap, at: self.RegistryPublicPath)
    }
}
```

### Two example strategies

```cadence
import "FungibleToken"
import "FlowToken"
import "StrategyRegistry"

// 1. No-op — exercises the registry pipeline without touching a real DEX.
access(all) contract NoOpStrategy {
    access(all) resource Impl: StrategyRegistry.Strategy {
        access(all) view fun describe(): String { return "noop" }
        access(StrategyRegistry.Execute) fun run(
            intent: &StrategyRegistry.Intent
        ): @{FungibleToken.Vault} {
            return <- FlowToken.createEmptyVault(vaultType: Type<@FlowToken.Vault>())
        }
    }
    init() {
        self.account.storage.save(<-create Impl(), to: /storage/noopStrategy)
    }
}

// 2. Stubbed swap — wires into a real DEX router in production. The point
//    is the *shape*: implements {Strategy}, entitled to swap on intent flow.
access(all) contract SwapStrategy {
    access(all) resource Impl: StrategyRegistry.Strategy {
        access(all) view fun describe(): String { return "FLOW->USDC via DEX X" }
        access(StrategyRegistry.Execute) fun run(
            intent: &StrategyRegistry.Intent
        ): @{FungibleToken.Vault} {
            // Production: borrow router cap, swap intent.inputAmount,
            // assert post-balance >= intent.minOutput, return output vault.
            return <- FlowToken.createEmptyVault(vaultType: Type<@FlowToken.Vault>())
        }
    }
    init() {
        self.account.storage.save(<-create Impl(), to: /storage/swapStrategy)
    }
}
```

### Admin transaction — add a strategy

```cadence
import "StrategyRegistry"

transaction(strategyStoragePath: StoragePath, interfaceName: String) {
    prepare(admin: auth(BorrowValue, IssueStorageCapabilityController) &Account) {
        let cap = admin.capabilities.storage
            .issue<auth(StrategyRegistry.Execute) &{StrategyRegistry.Strategy}>(
                strategyStoragePath
            )
        let reg = admin.storage.borrow<auth(StrategyRegistry.Admin) &StrategyRegistry.Registry>(
            from: StrategyRegistry.RegistryStoragePath
        ) ?? panic("registry not found")
        let id = reg.addStrategy(cap: cap, interfaceName: interfaceName)
        log("registered strategy id=".concat(id.toString()))
    }
}
```

### User-facing executor transaction

```cadence
import "FungibleToken"
import "FlowToken"
import "StrategyRegistry"

transaction(strategyID: UInt64, inputAmount: UFix64, minOutput: UFix64) {
    prepare(user: auth(BorrowValue) &Account) {
        let intent = StrategyRegistry.Intent(
            inputType: Type<@FlowToken.Vault>(),
            inputAmount: inputAmount,
            outputType: Type<@FlowToken.Vault>(),
            minOutput: minOutput
        )
        let reg = getAccount(0xCAFE).capabilities
            .borrow<&StrategyRegistry.Registry>(StrategyRegistry.RegistryPublicPath)
            ?? panic("registry not reachable")
        let out <- reg.run(id: strategyID, intent: &intent, caller: user.address)
        let receiver = user.capabilities
            .borrow<&{FungibleToken.Receiver}>(/public/flowTokenReceiver)
            ?? panic("no receiver")
        receiver.deposit(from: <-out)
    }
}
```

## Capability type validation at registration time

The single most important rule:

```cadence
pre { cap.check(): "strategy capability does not resolve to a valid &{Strategy}" }
```

`Capability<auth(Execute) &{Strategy}>` is validated at *registration time*
via `cap.check()`. Without this, an admin can register a capability whose
stored target does not actually conform to `{Strategy}` — the type slot is
satisfied syntactically because Cadence does not enforce target conformance on
the capability *value* until you `borrow()` it. A registry that never
`check()`s at add-time will only discover the mismatch the first time someone
calls `run(id:)`, which is exactly the wrong place to fail. Pair this with
storing `interfaceName` so off-chain auditors can compare the declared
interface against what was actually whitelisted.

## Multi-sig admin emerges naturally

The registry's `Admin` entitlement does not need a singleton admin resource.
Issue an `auth(Admin) &Registry` capability to each admin account — multiple
administrators each hold their own capability. Revoking one admin is a
one-line controller delete; adding one is one-line capability issuance. See
[admin-via-capability.md](admin-via-capability.md) for threshold logic and
admin audit trails.

```cadence
let adminCap = deployer.capabilities.storage
    .issue<auth(StrategyRegistry.Admin) &StrategyRegistry.Registry>(
        StrategyRegistry.RegistryStoragePath
    )
// Deliver via inbox or signed tx — never publish.
```

A 3-of-5 multi-sig is *not* enforced at the Cadence layer; it is enforced by
requiring the admin transaction to be signed by a multi-key account whose
weights sum correctly, or by routing admin actions through a governance
contract that holds the `auth(Admin)` cap and gates calls behind on-chain
voting.

## Anti-pattern: storing strategies as raw addresses

```cadence
// ❌ Trusts addresses at execute time, never verifies shape
access(all) resource RegistryBad {
    access(self) var strategies: {UInt64: Address}
    access(Admin) fun addStrategy(addr: Address): UInt64 { /* just stores addr */ }
    access(all) fun run(id: UInt64) {
        let addr = self.strategies[id]!
        // Hope /public/strategy exists, hope it conforms, hope the entitlement
        // is set up. Three hopes, zero checks.
        let cap = getAccount(addr).capabilities.get<auth(Execute) &{Strategy}>(/public/strategy)
        let ref = cap.borrow() ?? panic("no strategy at addr")
        // ...
    }
}
```

Problems: (1) no type check at add-time — the provider can swap out the target
resource later without re-registering; (2) mismatched-interface bugs surface
during a real intent execution, the worst possible time because user funds are
mid-flight; (3) hardcoded path coupling removes the provider's freedom to use
a different storage path. The fix is to take a
`Capability<auth(Execute) &{Strategy}>` directly and `cap.check()` it in
`addStrategy`.

## Anti-pattern: no revocation path

```cadence
// ❌ Once added, forever active — a single hacked provider compromises every user.
access(Admin) fun addStrategy(cap: ...) { /* ... no revoke / freeze */ }
```

Cadence contracts **can be updated by their deployer** subject to upgrade
rules (storage layout must remain compatible). A strategy that was honest at
registration may host malicious code one upgrade later. A registry that
cannot revoke is one upgrade away from draining every user that opted in.
Always ship `revokeStrategy(id:)` from day one, *and* a `freezeStrategy(id:)`
fast-pause for cases where you suspect the strategy but want to investigate
before destroying capability metadata.

## Anti-pattern: "active" as the only state

```cadence
// ❌ Boolean lifecycle: can't distinguish "paused for investigation" from
//    "deprecated, migrate off" from "revoked, dead forever".
access(all) struct Entry { access(all) var active: Bool }
```

A real strategy lifecycle has four meaningfully different phases:

| Phase        | Executor can call? | User signal                                  |
|--------------|--------------------|----------------------------------------------|
| `Active`     | Yes                | Normal operation                              |
| `Paused`     | No                 | Temporary; expect a return to Active          |
| `Deprecated` | No                 | Migrate to a newer strategy; old in wind-down |
| `Revoked`    | No                 | Terminal; do not re-add at the same ID        |

A `Bool` collapses all four into one. Indexers and frontends cannot give users
useful guidance. Always model the registry entry as a state machine — see
[resource-state-machines.md](resource-state-machines.md) for the
`access(self) var phase` discipline and `pre`-gated transitions.

## Anti-pattern: events as state

An off-chain script that replays `StrategyAdded` / `StrategyRevoked` history
to decide "is strategy 7 still active?" lags on-chain truth and misses
out-of-order delivery. Events are observability only; the authoritative
source is `Entry.phase`, read via `getEntry(id:).phase` in a script. See
[event-taxonomy.md](event-taxonomy.md) for the broader rule.

## Common pitfalls

- **Forgetting `cap.check()` in `addStrategy`.** This is the entire security
  property of the pattern. A test that registers a deliberately-mismatched
  capability and asserts the `pre` fires is mandatory.
- **Publishing the entitled registry cap.** `auth(Admin) &Registry` must
  *never* be published at a public path. Issue privately and deliver via
  inbox or signed transaction. Publish only `&Registry` (no entitlements) for
  read views.
- **Leaving `cap` on the Entry as `access(all) let`.** Owned values are fully
  entitled; if an `Entry` value escapes through a public getter, a copied
  `auth(Execute)` cap lets anyone call `execute` out-of-band. Return only a
  redacted view, or make the capability field `access(contract)`.
- **No `interfaceName` field.** Without an on-chain record of *which*
  interface the cap was checked against, future audits cannot distinguish a
  `SwapStrategy` from a `LendingStrategy` when both implement `{Strategy}`.
- **Reusing strategy IDs after revocation.** New strategies get fresh IDs
  from `nextID`. Re-using a revoked ID breaks event-history reasoning.
- **Calling `run` against a `Deprecated` strategy.** The `assert` in
  `run()` must require `Phase.Active` — not "phase != Revoked". A
  deprecated strategy is *intentionally* uncallable; route to a newer one.
- **Strategy doesn't double-check the intent.** `Strategy.run` should
  also `pre`-check `intent.outputType == Type<@OutputVault>()` inside the
  strategy. Defense in depth — the registry is not the only type guard.
- **Skipping events on idempotent revoke.** Emit even if the entry is already
  revoked. Indexers want a definitive close signal; a silent no-op breaks
  downstream alerting.
