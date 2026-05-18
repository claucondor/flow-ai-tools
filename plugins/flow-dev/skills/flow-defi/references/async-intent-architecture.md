# Async Intent Architecture Overview

An **async intent** on Flow is a signed, on-chain declaration of a desired
outcome — "swap 100 FLOW for maximum USDC over the next 24 hours with no more
than 1% slippage" — that separates the moment of user consent from the moment
of execution. The user signs once, encoding their terms into a resource. An
executor (automated or off-chain searcher) later fulfils those terms by
borrowing the authority the intent grants, executing a strategy, and releasing
settlement events when the conditions are met or returning funds when they are
not. This model composes cleanly with Flow's native scheduling primitive
(recurring intents that execute every N seconds), with CrossVM execution
(intents that route through Solidity contracts on Flow EVM), and with
trust-minimization goals (the user's funds never leave escrow until a strategy
that clears the user's stated criteria actually executes).

The pattern emerges from combining several Cadence-native primitives that each
solve one part of the problem: resource state machines enforce lifecycle
integrity, multi-transaction escrow enforces custody, strategy registries
enforce trust, scheduled transactions enforce timing, and DeFiActions
composition enforces atomic execution. No single primitive is the "async
intent system" — the architecture is the composition of all of them.

This document is the **map**. Every component described here links to the
reference that explains how to build it. Read this overview to understand
which primitives you need and why; follow the cross-links for implementation
detail.

---

## Intent Lifecycle

An async intent moves through five named states. The state is an on-chain
field on the Intent resource — it is never inferred from event history. See
[resource-state-machines.md](../../../cadence-lang/references/resource-state-machines.md)
for the discipline of `access(self) var phase` with `pre`-guarded entitled
transitions; see
[multi-tx-escrow.md](../../../cadence-lang/references/multi-tx-escrow.md)
for the escrow sub-machine that mirrors these states.

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │  User signs                                                          │
  │  intent terms                                                        │
  │     │                                                                │
  │     ▼                                                                │
  │  ┌─────────┐   deposit +     ┌─────────┐   strategy      ┌─────────┐│
  │  │ Signed  │ ─── arm() ───►  │ Funded  │ ──  executes ─► │ Active  ││
  │  └─────────┘                 └─────────┘                 └─────────┘│
  │                                   │                           │      │
  │                          cancel() │                 deadline  │      │
  │                         (pre-arm) │                  passes   │      │
  │                                   ▼                           ▼      │
  │                              ┌─────────┐                ┌─────────┐ │
  │                              │Cancelled│                │ Expired │ │
  │                              └─────────┘                └─────────┘ │
  │                                                              │       │
  │  Strategy satisfies                                 depositor│       │
  │  intent terms                                        refund()│       │
  │     │                                                        ▼      │
  │     ▼                                                   ┌─────────┐ │
  │  ┌─────────┐                                            │ Refunded│ │
  │  │ Settled │  ◄──── settlement event(s) emitted         └─────────┘ │
  │  └─────────┘                                                        │
  └──────────────────────────────────────────────────────────────────────┘
```

### State-to-component mapping

| State | Resource holding it | Entitlement to leave | Event emitted on entry | References |
|---|---|---|---|---|
| `Signed` | Intent resource (no funds yet) | `auth(Deposit)` to call `deposit()` + `arm()` | `IntentCreated` | [resource-state-machines.md](../../../cadence-lang/references/resource-state-machines.md) |
| `Funded` / `Claimable` | Escrow resource, funds inside | `auth(Claim)` held by executor | `Armed` (escrow) | [multi-tx-escrow.md](../../../cadence-lang/references/multi-tx-escrow.md) |
| `Active` | Intent resource + funded escrow | Strategy execution or deadline | `TickStarted` (if scheduled) | [scheduled-integration.md](../../../flow-actions/references/scheduled-integration.md) |
| `Settled` | Escrow drained, intent closed | `close()` (terminal) | `Settled` / `TickExecuted` | [multi-tx-escrow.md](../../../cadence-lang/references/multi-tx-escrow.md) |
| `Expired` | Escrow in `Expired` phase | `auth(Refund)` to call `refund()` | `Expired` transition | [multi-tx-escrow.md](../../../cadence-lang/references/multi-tx-escrow.md) |
| `Cancelled` | Escrow drained (pre-arm cancel) | Terminal — no further mutation | `Cancelled` | [multi-tx-escrow.md](../../../cadence-lang/references/multi-tx-escrow.md) |
| `Refunded` | Escrow closed, funds returned | Terminal | `Refunded` | [multi-tx-escrow.md](../../../cadence-lang/references/multi-tx-escrow.md) |

The `Funded`/`Active` distinction reflects whether the strategy has started
executing. For a one-shot intent these collapse to a single execution
transaction. For a recurring intent (scheduled DCA, periodic rebalance) the
escrow stays `Claimable` across multiple ticks and the intent remains `Active`
until the deadline or the strategy marks completion.

---

## Architecture Components

The following seven components together implement the full async intent pattern.
Each is a distinct concern; their interfaces are deliberately narrow so they can
be composed independently.

### 1. Intent Resource

The Intent resource is the user's signed statement of desired outcome. It holds
immutable terms (input token type and amount, output token type, minimum output,
deadline, slippage tolerance), a mutable `phase` field, and references (not the
funds themselves) to the escrow and strategy.

```cadence
// Minimal intent terms struct — stored immutably inside the Intent resource.
// The resource's phase field is access(self) and mutable only via entitled methods.
access(all) struct IntentTerms {
    access(all) let inputType:    Type
    access(all) let inputAmount:  UFix64
    access(all) let outputType:   Type
    access(all) let minOutput:    UFix64
    access(all) let maxSlippage:  UFix64   // fraction, e.g. 0.01 for 1%
    access(all) let deadline:     UFix64   // Unix seconds
}
```

Design principles:

- ✅ Separate the Intent struct (terms, immutable) from the Intent resource
  (lifecycle state, mutable) — the struct can be passed to strategy `run()`
  without the caller acquiring lifecycle authority.
- ❌ Do not embed strategy execution state inside the Intent resource — the
  strategy resource owns its own state; the intent only holds a reference (a
  capability) to it.
- ✅ Store `deadline` in the resource at creation time and make it immutable.
  The escrow references the same deadline; they must agree.
- ❌ Do not derive the intent's callability from event history — always read
  `intent.getPhase()` from the resource.

References:
[resource-state-machines.md](../../../cadence-lang/references/resource-state-machines.md),
[multi-tx-escrow.md](../../../cadence-lang/references/multi-tx-escrow.md)

---

### 2. Escrow

The escrow holds user funds across the gap between deposit and execution. It is
the custody primitive — the only component that actually owns a
`@{FungibleToken.Vault}` at rest. The escrow's phase machine mirrors the
intent's lifecycle states; both agree on deadline and transition together.

Three actors, three entitlements:

| Actor | Entitlement | Phase | Action |
|---|---|---|---|
| Depositor | `auth(Deposit)` | `Funding` | `deposit()`, `arm()`, `cancel()` |
| Executor | `auth(Claim)` | `Claimable` | `claim()` (pulls funds to execute strategy) |
| Depositor | `auth(Refund)` | `Expired` | `refund()` (returns funds after deadline) |

The escrow does not know about strategies or scheduling — it only enforces
custody and phase transitions. Any actor (keeper, scheduled tx, executor) can
call the `access(all)` `expireIfStale()` to drive the `Claimable → Expired`
transition once the deadline passes.

For EVM-side custody (when the intent routes through an EVM contract), the same
state machine applies but the "vault" is replaced by a COA capability — see
component 7 below and
[coa-lifecycle.md](../../../flow-crossvm/references/coa-lifecycle.md).

Full worked implementation including the anti-patterns (storage-path-per-txID,
events-as-state) and the Manager dictionary pattern:
[multi-tx-escrow.md](../../../cadence-lang/references/multi-tx-escrow.md)

---

### 3. Strategy Registry

The strategy registry is an admin-curated whitelist of approved executors. It
holds `Capability<auth(Execute) &{Strategy}>` values, each checked at
registration time via `cap.check()`. Users and executors look up a strategy by
ID; the registry guarantees the capability resolves to a conforming
implementation and that the strategy is in `Active` phase before calling
`run()`.

The registry enforces the trust boundary between the user and the executor:

- ✅ Validate `cap.check()` at `addStrategy()` time — not at `run()` time.
  A mismatched capability discovered mid-execution means user funds are already
  mid-flight.
- ✅ Model strategy lifecycle with four phases (`Active`, `Paused`,
  `Deprecated`, `Revoked`) — a boolean `active` field cannot distinguish
  "temporarily paused for investigation" from "permanently revoked."
- ❌ Do not store strategies as raw addresses — raw addresses give no typed
  guarantee about the code on the other side.

The `Strategy` interface declares the entitled `run(intent: &IntentTerms)`
entry point that every approved strategy implements:

```cadence
// From strategy-registry.md — the shape every strategy must conform to.
access(all) resource interface Strategy {
    access(all) view fun describe(): String
    // Execute entitlement is what the registry holds; run() may only be called
    // by a holder of auth(StrategyRegistry.Execute) — the registry itself.
    access(StrategyRegistry.Execute) fun run(intent: &IntentTerms): @{FungibleToken.Vault}
}
```

Multi-sig admin emerges naturally: issue one `auth(Admin) &Registry` capability
per admin account; revoking one admin is a single controller delete.

Full implementation with worked examples, anti-patterns, and the four lifecycle
phases:
[strategy-registry.md](../../../cadence-lang/references/strategy-registry.md)

---

### 4. Strategy Implementation

A strategy implementation conforms to `{Strategy}` and contains the actual
execution logic — calling a DEX, routing through a lending protocol, or
executing a multi-hop swap. The strategy resource is the only component that
touches the underlying DeFi protocol. It is deliberately isolated from the
intent lifecycle so protocols can evolve strategies independently.

A strategy implementation's `run()` method should:
1. Validate `intent.inputType`, `intent.outputType`, and `intent.minOutput`
   against what the strategy can actually deliver.
2. Call the DeFiActions composition pipeline (Source → Swapper → Sink) to
   execute the trade — see component 5.
3. Assert the output vault's balance satisfies `intent.minOutput` before
   returning it.
4. Never directly access the escrow — the executor extracts funds from the
   escrow and passes them to the strategy as a vault argument.

```cadence
// Pattern: strategy validates terms first, then executes.
access(StrategyRegistry.Execute)
fun run(intent: &IntentTerms): @{FungibleToken.Vault} {
    pre {
        intent.inputType == Type<@FlowToken.Vault>(): "strategy only handles FLOW input"
        intent.outputType == Type<@USDC.Vault>(): "strategy only handles USDC output"
    }
    // ... DeFiActions composition here (see component 5)
    // post: assert returned vault.balance >= intent.minOutput
}
```

References:
[strategy-registry.md](../../../cadence-lang/references/strategy-registry.md),
[composition-patterns.md](../../../flow-actions/references/composition-patterns.md)

---

### 5. DeFiActions Composition

DeFiActions is the atomic execution primitive — a Source → Swapper → Sink
pipeline that executes entirely within one Cadence transaction. Because all
steps are in a single transaction, the pipeline is all-or-nothing: no EVM-style
partial commit where one hop succeeds and the next reverts.

Every pipeline uses a single `UniqueIdentifier` threaded through all
connectors, binding every `Withdrawn`, `Swapped`, and `Deposited` event into
one traceable operation:

```cadence
// Inside strategy.run() or inside a scheduled handler's executeTransaction():
let operationID = DeFiActions.createUniqueIdentifier()
let source  = FungibleTokenConnectors.VaultSource(
    min: nil, withdrawVault: inputVaultCap, uniqueID: operationID)
let swapper = IncrementFiSwapConnectors.Swapper(
    path: swapPath, inVault: inType, outVault: outType, uniqueID: operationID)
let sink    = FungibleTokenConnectors.VaultSink(
    max: nil, depositVault: outputVaultCap, uniqueID: operationID)
// execute phase:
let inVault <- source.withdrawAvailable(maxAmount: intent.inputAmount)
let outVault <- swapper.swap(quote: nil, inVault: <-inVault)
assert(outVault.balance >= intent.minOutput, message: "slippage exceeded")
sink.depositCapacity(from: &outVault as auth(FungibleToken.Withdraw) &{FungibleToken.Vault})
assert(outVault.balance == 0.0, message: "residual vault after deposit")
destroy outVault
```

For intents requiring multi-hop routes, `SwapConnectors.SequentialSwapper`
chains multiple Swappers while maintaining the same `{DeFiActions.Swapper}`
interface — the composition pattern is identical.

Pattern selection (Source→Sink, SwapSource→Sink, Source→SwapSink, direct
chaining, multi-hop) and the six critical safety rules:
[composition-patterns.md](../../../flow-actions/references/composition-patterns.md)

---

### 6. Scheduled Handler

The scheduled handler is the automation layer — a resource conforming to
`FlowTransactionScheduler.TransactionHandler` that wakes up at a configured
timestamp, pulls funds from the escrow (via the executor's `auth(Claim)` cap),
runs a DeFiActions composition, deposits the output, and re-schedules itself
for the next tick if the intent has not yet settled.

This component is optional for one-shot intents (where an off-chain executor
submits the settlement transaction). It is required for recurring intents — DCA
strategies, periodic rebalances, scheduled liquidations — where the user wants
the system to execute on its own schedule with no further interaction.

Key invariants from the scheduler integration:

- ✅ Create a fresh `DeFiActions.UniqueIdentifier` at the top of every
  `executeTransaction()` call — never reuse one across ticks.
- ✅ Use early `return` for skippable conditions (source empty, sink full,
  disabled flag); never `panic` for ordinary skip logic — a panic costs the
  full tick fee and prevents self-rescheduling.
- ✅ Self-reschedule at the bottom of `executeTransaction()`, after all side
  effects, by calling `FlowTransactionScheduler.schedule()` with fees drawn
  from a vault the handler owns.
- ✅ Call `FlowTransactionScheduler.estimate()` before rescheduling — a full
  slot at schedule time causes `schedule()` to panic, which rolls back the
  entire tick including the already-executed composition.

The 9,999 CU per-tick ceiling is the hardest constraint. A simple
Source → Swapper → Sink composition runs approximately 250–600 CU. A CrossVM
swap that encodes ABI data and decodes EVM return values can consume 1,000–
3,000+ CU on top of that. Measure on the emulator before committing to an
`executionEffort` value.

Scheduler API (TransactionHandler interface, priority model, fee model,
cancellation, failure state machine):
[scheduled-transactions.md](../../../cadence-lang/references/scheduled-transactions.md)

Canonical `DCAExecutor` contract template, per-tick CU budget table, failure
handling, admin operations, and initial setup transaction:
[scheduled-integration.md](../../../flow-actions/references/scheduled-integration.md)

---

### 7. COA (Cross-VM, Optional)

When an intent routes through an EVM contract — an EVM DEX, an ERC-20 lending
protocol, or a Solidity-based AMM — the execution requires a Cadence Owned
Account (COA). The COA is a resource (`@EVM.CadenceOwnedAccount`) whose `uuid`
deterministically derives an EVM address. Custody follows Cadence's linear
ownership rules: whoever holds the `auth(EVM.Call, EVM.Withdraw)` capability
controls the EVM address.

For cross-VM intents, the COA plays the role the `@{FungibleToken.Vault}` plays
in Cadence-only intents: it is the fund-holding primitive on the EVM side. The
escrow holds a capability to the depositor's COA (not the COA resource itself)
and exercises it only during the `Claimable` phase.

```cadence
// Cross-VM escrow pattern: the escrow holds a narrow auth cap, not the COA.
access(self) let depositorCOA:
    Capability<auth(EVM.Call, EVM.Withdraw) &EVM.CadenceOwnedAccount>

// In claim():
let coa = self.depositorCOA.borrow() ?? panic("COA cap revoked")
let result = coa.call(to: evmDexAddress, data: abiCalldata, gasLimit: 300_000,
                      value: EVM.Balance(attoflow: 0))
assert(result.status == EVM.Status.successful,
    message: "EVM claim call failed: ".concat(result.errorMessage))
```

Critical: always assert `result.status == EVM.Status.successful` immediately
after `coa.call`. A failed EVM call does NOT automatically revert the
surrounding Cadence transaction — the escrow would record `Settled` while the
EVM payment silently failed.

After the escrow closes, the depositor deletes the capability controller to
revoke the COA authority. Revocation is the cross-VM equivalent of draining a
Cadence vault: capabilities, not vaults, carry authority here.

COA entitlements (Call, Withdraw, Deploy, Validate, Owner), canonical storage
paths, funding from Cadence, withdrawing back to Cadence, custody pattern,
anti-patterns, and capability revocation:
[coa-lifecycle.md](../../../flow-crossvm/references/coa-lifecycle.md)

---

## Component Interaction Diagram

The following shows how the seven components wire together for a typical
scheduled intent execution:

```
  User signs intent (Tx 1)
    │
    ▼
  Intent resource created (Signed)
  Escrow created (Funding)
  Escrow funded + armed (Claimable)
  Scheduled handler registered
    │
    ▼
  FlowTransactionScheduler fires tick N
    │
    ├── handler.executeTransaction(id: N, data: nil)
    │     │
    │     ├── 1. Generate tickOpID (DeFiActions.createUniqueIdentifier)
    │     │
    │     ├── 2. Borrow escrow.claim() via auth(Claim) cap
    │     │       → escrow transitions Claimable → Settled
    │     │       → funds released as @{FungibleToken.Vault}  [component 2]
    │     │
    │     ├── 3. Look up strategy in registry by ID
    │     │       → assert strategy phase == Active           [component 3]
    │     │
    │     ├── 4. Call strategy.run(intent: &terms)
    │     │       → Source → Swapper → Sink pipeline runs     [components 4+5]
    │     │       → assert output.balance >= intent.minOutput
    │     │
    │     ├── 5. Deposit output to user's receiver
    │     │       → emit TickExecuted(schedulerID, tickOpID)  [component 6]
    │     │
    │     └── 6. Self-reschedule (if recurring intent + deadline not passed)
    │
    ▼
  Intent resource transitions Active → Settled
  Escrow transitions Settled → Closed
  Settlement event(s) emitted
```

For cross-VM intents, step 4 routes through a COA (component 7):
the strategy calls `coa.call(...)` with ABI-encoded calldata and the
Swapper wraps the EVM DEX call.

---

## Decision Matrix: Async Intent vs Alternatives

| Pattern | User interaction | Execution trust | Gas payer | Recurring? | CrossVM? | Choose when |
|---|---|---|---|---|---|---|
| **Async intent** (this architecture) | Sign once, executor settles | Strategy registry whitelist | Executor (or scheduled handler) | Yes — self-rescheduling handler | Yes — COA in claim() | User wants delayed or recurring execution with guaranteed refund on failure |
| **Synchronous swap** (user submits each swap) | Sign every swap | Direct — user controls the tx | User | No — user must submit each time | Yes — single atomic tx | User is online and interactive; latency matters more than automation |
| **Limit order** (executor picks off at target price) | Sign once, executor watches | Order book contract | Executor | No — single fill per order | Possible — same architecture, price trigger instead of time trigger | User wants price-conditional execution, not time-conditional |
| **Traditional aggregator** (no keeper) | Sign every time | Aggregator route | User every time | No | Depends on aggregator | User prefers to stay in control of timing; no intent system deployed |

Additional decision criteria:

- ✅ Use async intent when the user wants a **guaranteed refund path** if
  execution fails or the deadline passes — the escrow's `Expired` phase and
  `auth(Refund)` entitlement enforce this without trusting the executor.
- ✅ Use async intent when execution spans **more than one transaction** —
  either a recurring schedule or a CrossVM flow that exceeds the 9,999 CU
  single-tx ceiling.
- ❌ Avoid async intent for **low-value, time-sensitive, interactive** trades
  where the overhead of deploying an escrow and strategy is disproportionate
  to the swap amount or latency requirement.
- ❌ Avoid async intent if the **strategy set is unbounded** — the registry
  model requires curated whitelisting; permissionless strategy registration
  undermines the trust model.

---

## Trust and Security Properties

The async intent model provides these on-chain guarantees:

**1. Custody integrity.** User funds live inside the escrow resource at a
single storage path from deposit until either settlement or refund. They cannot
be moved by any actor without the escrow-gated entitlement. The escrow does not
trust the executor — it only releases funds when the executor holds
`auth(Claim)` and calls `claim()` in the `Claimable` phase before the deadline.

**2. Strategy trust boundary.** The strategy registry enforces that only
admin-approved strategies are callable. `cap.check()` at registration time
verifies the capability resolves to a conforming `{Strategy}` resource. Revoke
is available from day one — a strategy that was honest at registration can be
paused or revoked if it is upgraded with malicious logic. See the anti-pattern
"no revocation path" in
[strategy-registry.md](../../../cadence-lang/references/strategy-registry.md).

**3. Slippage protection.** The intent terms include `minOutput` and
`maxSlippage`. The strategy's `run()` method asserts these terms before
returning the output vault. The DeFiActions pipeline also enforces that the
residual vault is zero after `depositCapacity` — no tokens can be silently
retained by the strategy.

**4. Atomicity within a tick.** DeFiActions compositions are all-or-nothing
within a single Cadence transaction. If the swap reverts (slippage, broken
pool, insufficient liquidity), the entire tick reverts — the escrow's `claim()`
is rolled back, the funds return to `Claimable`, and the next tick can retry.

**5. Refund guarantee.** The escrow's `deadline` is set at creation time and
is immutable. Once the deadline passes, any party can call `expireIfStale()` to
transition to `Expired`, and the depositor can call `refund()` using their
`auth(Refund)` capability without any cooperation from the executor. A
scheduled transaction at `deadline + epsilon` can automate this if desired.

**6. CrossVM atomicity caveat.** When the strategy calls `coa.call()`, a
failed EVM call does not automatically revert the Cadence transaction. The
strategy implementation must assert `result.status == EVM.Status.successful`
and panic on failure to enforce all-or-nothing semantics. See
[coa-lifecycle.md](../../../flow-crossvm/references/coa-lifecycle.md) and
[protocol-architecture.md](protocol-architecture.md) for the full failure mode
taxonomy.

---

## Design Principles: Dos and Don'ts

✅ **Separate Intent terms from Strategy state.** The `IntentTerms` struct is
immutable and passed by reference to `strategy.run()`. The strategy owns its
own mutable fields (router path, fee tier, etc.). Mixing them couples the
lifecycle of one to the other.

❌ **Do not embed strategy execution state in the Intent resource.** An Intent
that tracks "how many of my ticks have executed" conflates the user's consent
record with the strategy's operational bookkeeping.

✅ **One escrow per intent.** The escrow's `uuid` is the canonical intent ID.
A Manager dictionary (one per user account) indexes escrows by `uuid`, giving
atomic insert, enumeration, and a single storage path.

❌ **Do not use a storage path derived from a transaction ID as the escrow
location.** Two concurrent flows can derive the same path; `save` panics on
collision; `load` + re-save loses the resource on interleaved failure. See
anti-pattern 1 in
[multi-tx-escrow.md](../../../cadence-lang/references/multi-tx-escrow.md).

✅ **Emit settlement events for indexers, but never read events as state.**
The Intent resource's `phase` field is the source of truth. Indexers read
events; contract logic reads `getPhase()`.

❌ **Do not skip the `close()` terminal state.** A `Settled` or `Cancelled`
escrow that never transitions to `Closed` stays in storage indefinitely,
accumulating storage fees on the holder's account. Call `close()` in the same
transaction as the final `claim()`/`refund()`/`cancel()`, or let a scheduled
sweep auto-prune terminal escrows.

✅ **Issue one narrow capability per consumer.** The executor holds
`auth(Claim)`. The depositor holds `auth(Deposit, Refund)`. The admin holds
`auth(AdminRecover)`. Never issue a combined `auth(Deposit, Claim, Refund)`
capability — a single leaked reference can settle the escrow in either
direction.

---

## Cross-links to Every Primitive

This overview synthesizes the following references. Each architectural
component above links to one or more of these; the links below are the
canonical starting points for implementation depth.

| Reference | Component | What it covers |
|---|---|---|
| [resource-state-machines.md](../../../cadence-lang/references/resource-state-machines.md) | Intent resource lifecycle | `access(self) var phase`, entitled transition methods, `pre`/`post` discipline, terminal phases |
| [multi-tx-escrow.md](../../../cadence-lang/references/multi-tx-escrow.md) | Escrow | Funding/Claimable/Settled/Expired/Closed state machine, three-entitlement model, Manager dictionary, anti-patterns |
| [strategy-registry.md](../../../cadence-lang/references/strategy-registry.md) | Strategy registry + implementation | `Strategy` interface, `cap.check()` at registration, four lifecycle phases, admin-via-capability, revocation patterns |
| [composition-patterns.md](../../../flow-actions/references/composition-patterns.md) | DeFiActions composition | Source → Swapper → Sink patterns, `UniqueIdentifier`, atomicity guarantees, token-order rules, six safety rules |
| [scheduled-integration.md](../../../flow-actions/references/scheduled-integration.md) | Scheduled handler | `DCAExecutor` template, per-tick CU budget, self-rescheduling, panic-vs-return discipline, admin ops |
| [scheduled-transactions.md](../../../cadence-lang/references/scheduled-transactions.md) | Scheduler API | `TransactionHandler` interface, priority model, fee model, CU ceiling, failure state machine, cancellation |
| [coa-lifecycle.md](../../../flow-crossvm/references/coa-lifecycle.md) | COA (CrossVM) | Entitlements, canonical paths, deposit/withdraw, custody pattern, narrow-cap-per-consumer, revocation |
| [protocol-architecture.md](protocol-architecture.md) | System context | Flow's structural advantages (MEV-free EVM, CrossVM atomicity), COA use cases, cross-VM failure modes |

---

## CU Budget Reference

For scheduled async intents, the 9,999 CU per-tx ceiling constrains what a
single tick can do. The table below is a planning reference; always measure
empirically on the emulator before committing to `executionEffort`.

| Component | Approximate CU | Notes |
|---|---|---|
| Handler overhead + event emission | 30–60 CU | UniqueIdentifier creation, TickStarted event |
| Escrow `claim()` | 20–50 CU | Phase check, vault withdraw, event |
| Registry `run()` lookup | 10–30 CU | Phase check, capability borrow |
| DeFiActions Source.withdrawAvailable | 20–40 CU | Vault withdraw, post-condition event |
| DeFiActions Swapper.swap (single-pool) | 100–300 CU | DEX routing complexity |
| DeFiActions Swapper.swap (multi-hop) | +100 CU per hop | SequentialSwapper overhead |
| DeFiActions Sink.depositCapacity | 20–40 CU | Vault deposit, event |
| COA `coa.call` (state-mutating EVM) | 200–500 CU | ABI encode + decode + bridge overhead |
| COA EVM return data (large `bytes[]`) | +200–800 CU | Scales super-linearly with return size |
| Self-reschedule `mgr.schedule()` | 50–100 CU | Manager indexing overhead |

A Cadence-only DCA tick (Source → single-hop Swapper → Sink + self-reschedule)
runs approximately **270–620 CU** total — well within the 9,999 ceiling. A
CrossVM tick that routes through an EVM DEX and decodes a complex response can
easily reach **1,500–4,000+ CU** and must be measured before deployment.

For work that genuinely exceeds 9,999 CU per tick, use the multi-transaction
escrow pattern to split execution across multiple ticks rather than attempting
to pack everything into one handler call.
