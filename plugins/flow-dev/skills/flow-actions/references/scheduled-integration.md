# Integration with Scheduled Transactions

The combination of DeFiActions composition and `FlowTransactionScheduler` is the canonical
pattern for **on-chain intent executors**: a resource that wakes up on a timer, runs a full
Source → Swapper → Sink pipeline, and immediately re-schedules itself for the next tick —
with no off-chain keeper, cron server, or relayer required. This is the time-based complement
to event-driven strategies: reach for it when the trigger is "every N seconds" rather than
"when price crosses threshold." Each tick is an independent, atomic Cadence transaction; the
DeFiActions framework guarantees that vault movement is all-or-nothing within the tick, and the
scheduler guarantees eventual execution at the agreed priority level.

> **Beta notice:** `DeFiActions` is in beta on Testnet and Mainnet. Interfaces may change before
> final release. Monitor [`onflow/FlowActions`](https://github.com/onflow/FlowActions) for
> breaking changes.

---

## How the Pattern Fits Together

```
┌─────────────────────────────────────────────────────────┐
│  FlowTransactionScheduler tick N                        │
│                                                         │
│  handler.executeTransaction(id: N, data: nil)           │
│    1. Generate fresh UniqueIdentifier (tickOpID)        │
│    2. Build Source / Swapper / Sink connectors          │
│    3. Source.withdrawAvailable → Swapper.swap → Sink    │
│    4. Assert residual vault balance == 0.0              │
│    5. Emit TickExecuted event (schedulerID + tickOpID)  │
│    6. Schedule tick N+1 (if enabled, reserve adequate)  │
└─────────────────────────────────────────────────────────┘
```

Each tick carries its own `DeFiActions.UniqueIdentifier`. Cross-tick traceability is preserved
through the contract's `TickExecuted` event, which records both the scheduler `id` (unique per
scheduled run) and the composition `tickOpID` (unique per DeFiActions pipeline).

---

## UniqueIdentifier per Tick vs. per Handler Lifetime

Every call to `DeFiActions.createUniqueIdentifier()` produces a monotonically increasing,
globally unique `UInt64` id. The rule is:

- **One `UniqueIdentifier` per scheduled run** — created at the top of `executeTransaction`,
  passed to every connector constructed in that run.
- **Never one `UniqueIdentifier` per handler lifetime** — reusing the same `uniqueID` across
  multiple ticks conflates all `Withdrawn`, `Swapped`, and `Deposited` events into a single
  logical trace, making indexers unable to distinguish tick N from tick N+1.

```cadence
// ✅ Correct: fresh ID each tick
access(FlowTransactionScheduler.Execute)
fun executeTransaction(id: UInt64, data: AnyStruct?) {
    let tickOpID = DeFiActions.createUniqueIdentifier()  // new ID every tick
    // ... build connectors with tickOpID ...
}

// ❌ Wrong: ID stored at handler init and reused
access(all) resource Handler : FlowTransactionScheduler.TransactionHandler {
    let sharedID: DeFiActions.UniqueIdentifier  // set once in init — never do this
    // All ticks share the same ID; Withdrawn/Swapped/Deposited events are indistinguishable.
}
```

---

## Canonical Contract Template: `DCAExecutor`

The following contract implements a Dollar-Cost-Averaging (DCA) executor: it buys a fixed
amount of a target token from a USDC vault every N seconds, depositing the proceeds into
a separate output vault. It is the worked reference for the entire integration pattern.

```cadence
import "FungibleToken"
import "DeFiActions"
import "FungibleTokenConnectors"
import "SwapConnectors"
import "IncrementFiSwapConnectors"
import "FlowTransactionScheduler"
import "FlowTransactionSchedulerUtils"
import "FlowToken"

/// DCAExecutor
///
/// On-chain Dollar-Cost-Averaging intent executor. Every `intervalSeconds` the handler
/// withdraws up to `amountPerTick` USDC from a source vault, swaps it for a target token,
/// and deposits the output. The handler self-reschedules until paused or the fee reserve
/// falls below a minimum threshold.
///
/// Storage layout (deployer account):
///   /storage/dcaExecutorHandler   — the Handler resource
///   /storage/dcaManager           — FlowTransactionSchedulerUtils.Manager
///
access(all) contract DCAExecutor {

    // -------------------------------------------------------------------------
    // Events
    // -------------------------------------------------------------------------

    /// Emitted at the start of every tick before any vault movement.
    access(all) event TickStarted(
        schedulerID: UInt64,   // FlowTransactionScheduler id for this tick
        tickOpID: UInt64       // DeFiActions UniqueIdentifier for this tick's pipeline
    )

    /// Emitted after a successful Source → Swapper → Sink composition.
    access(all) event TickExecuted(
        schedulerID: UInt64,
        tickOpID: UInt64,
        amountIn: UFix64,
        amountOut: UFix64
    )

    /// Emitted when a tick is skipped (source empty or sink full) without swapping.
    access(all) event TickSkipped(schedulerID: UInt64, tickOpID: UInt64, reason: String)

    /// Emitted when the handler decides not to reschedule.
    access(all) event ChainStopped(reason: String)

    // -------------------------------------------------------------------------
    // Configuration (mutable by admin)
    // -------------------------------------------------------------------------

    access(all) var enabled: Bool
    access(all) var intervalSeconds: UFix64
    access(all) var amountPerTick: UFix64
    access(all) var executionEffort: UInt64      // CU budget per tick
    access(all) var schedulerPriority: FlowTransactionScheduler.Priority
    access(all) let minFeeReserveBalance: UFix64  // refuse to reschedule below this

    // -------------------------------------------------------------------------
    // Handler resource
    // -------------------------------------------------------------------------

    // NOTE: swapPath field
    // IncrementFiSwapConnectors.Swapper requires a `path: [String]` as its first parameter.
    // The path is an array of pool identifier strings (typically contract identifiers in the
    // form "A.<address>.<ContractName>") that the IncrementFi router uses to sequence hops.
    // For a single-pool swap it contains exactly two elements (inToken address, outToken
    // address); multi-hop paths have more. path.length >= 2 is enforced by the constructor's
    // pre-condition via _validateSwapperInitArgs. Omitting path is a Cadence compile error
    // ("missing argument for parameter 'path'") and is a common mistake in examples found
    // in the wild. The path must be stored on the Handler at deploy time because it encodes
    // the operator's choice of routing, not a per-tick runtime value.
    access(all) resource Handler : FlowTransactionScheduler.TransactionHandler {

        /// IncrementFi pool route for the swap. Must have at least 2 elements.
        /// Example (USDC → FLOW on mainnet):
        ///   ["A.b19436aae4d94622.USDC", "A.1654653399040a61.FlowToken"]
        access(self) let swapPath: [String]

        /// Capability to withdraw USDC from the source vault.
        access(self) let usdcWithdrawCap:
            Capability<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>

        /// Capability to deposit the target token into the output vault.
        access(self) let outputDepositCap: Capability<&{FungibleToken.Vault}>

        /// Fee reserve — holds FLOW to fund future reschedules.
        access(self) var feeReserve: @FlowToken.Vault

        /// The handler's own Execute-entitled capability (used for self-rescheduling).
        access(self) let handlerCap:
            Capability<auth(FlowTransactionScheduler.Execute)
                       &{FlowTransactionScheduler.TransactionHandler}>

        /// The Manager used to schedule and index ticks.
        access(self) let managerRef:
            Capability<auth(FlowTransactionSchedulerUtils.Owner)
                       &{FlowTransactionSchedulerUtils.Manager}>

        // -----------------------------------------------------------------
        // ViewResolver stubs (required by TransactionHandler)
        // -----------------------------------------------------------------
        access(all) view fun getViews(): [Type] { return [] }
        access(all) fun resolveView(_ view: Type): AnyStruct? { return nil }

        // -----------------------------------------------------------------
        // Core callback
        // -----------------------------------------------------------------

        access(FlowTransactionScheduler.Execute)
        fun executeTransaction(id: UInt64, data: AnyStruct?) {

            // 1. Create a fresh UniqueIdentifier for this tick's pipeline.
            let tickOpID = DeFiActions.createUniqueIdentifier()
            emit DCAExecutor.TickStarted(schedulerID: id, tickOpID: tickOpID.id)

            // 2. Kill-switch and reserve check — early return (NOT panic) to preserve
            //    fee costs and ensure the Executed event fires.
            if !DCAExecutor.enabled {
                emit DCAExecutor.TickSkipped(schedulerID: id, tickOpID: tickOpID.id, reason: "disabled")
                return
            }
            if self.feeReserve.balance < DCAExecutor.minFeeReserveBalance {
                emit DCAExecutor.ChainStopped(reason: "fee reserve below minimum")
                return  // do NOT reschedule; chain terminates here
            }

            // 3. Build connectors — all with the same tickOpID.
            let source = FungibleTokenConnectors.VaultSource(
                min: nil,
                withdrawVault: self.usdcWithdrawCap,
                uniqueID: tickOpID
            )
            let swapper = IncrementFiSwapConnectors.Swapper(
                // path: stored at Handler init time — encodes the IncrementFi pool route.
                // Must have >= 2 elements; validated by IncrementFiSwapConnectors constructor.
                path: self.swapPath,
                // token ordering: USDC is inType, targetToken is outType
                // (caller verified this matches the pool's canonical pair ordering at deploy time)
                inVault: self.usdcWithdrawCap.borrow()!.getType(),
                outVault: self.outputDepositCap.borrow()!.getType(),
                uniqueID: tickOpID
            )
            let sink = FungibleTokenConnectors.VaultSink(
                max: nil,
                depositVault: self.outputDepositCap,
                uniqueID: tickOpID
            )

            // 4. Sizing: use the lesser of (amountPerTick, source available, sink capacity).
            let available = source.minimumAvailable()
            let capacity  = sink.minimumCapacity()
            let maxAmount = DCAExecutor.amountPerTick

            // If source is empty or sink is full, skip without swapping.
            if available == 0.0 || capacity == 0.0 {
                let reason = available == 0.0 ? "source empty" : "sink full"
                emit DCAExecutor.TickSkipped(schedulerID: id, tickOpID: tickOpID.id, reason: reason)
                // Fall through to reschedule — the next tick may have tokens.
            } else {
                let withdrawMax = available < capacity ? available : capacity
                let actualMax   = withdrawMax < maxAmount ? withdrawMax : maxAmount

                // 5. Execute the pipeline.
                let inVault <- source.withdrawAvailable(maxAmount: actualMax)
                let amountIn = inVault.balance

                // Swap: Source token → target token.
                let outVault <- swapper.swap(quote: nil, inVault: <-inVault)
                let amountOut = outVault.balance

                // Deposit into Sink.
                sink.depositCapacity(
                    from: &outVault as auth(FungibleToken.Withdraw) &{FungibleToken.Vault}
                )
                // Assert no residual. A non-zero balance here is a logic error.
                assert(outVault.balance == 0.0, message: "DCA: residual vault after deposit")
                destroy outVault

                emit DCAExecutor.TickExecuted(
                    schedulerID: id,
                    tickOpID: tickOpID.id,
                    amountIn: amountIn,
                    amountOut: amountOut
                )
            }

            // 6. Self-reschedule — always at the end, after side effects.
            self.scheduleNext()
        }

        // -----------------------------------------------------------------
        // Self-rescheduling helper
        // -----------------------------------------------------------------

        access(self) fun scheduleNext() {
            if !DCAExecutor.enabled { return }

            // Calculate the fee for the next tick.
            let nextTimestamp = getCurrentBlock().timestamp + DCAExecutor.intervalSeconds
            let estimateResult = FlowTransactionScheduler.estimate(
                handlerCap: self.handlerCap,
                data: nil,
                timestamp: nextTimestamp,
                priority: DCAExecutor.schedulerPriority,
                executionEffort: DCAExecutor.executionEffort
            )
            if estimateResult.error != nil {
                // Slot may be full; try again at the next available second.
                // For simplicity this template stops the chain — a robust implementation
                // would retry with timestamp + 1.0. See "Pitfall: slot contention".
                emit DCAExecutor.ChainStopped(reason: estimateResult.error!)
                return
            }
            let feeAmount = estimateResult.flowFee!

            // Guard: refuse to reschedule if the reserve is insufficient.
            if self.feeReserve.balance < feeAmount + DCAExecutor.minFeeReserveBalance {
                emit DCAExecutor.ChainStopped(reason: "fee reserve insufficient for next tick")
                return
            }

            let fees <- self.feeReserve.withdraw(amount: feeAmount) as! @FlowToken.Vault
            let mgr = self.managerRef.borrow()
                ?? panic("DCAExecutor: Manager capability broken — cannot reschedule")

            let _ = mgr.schedule(
                handlerCap: self.handlerCap,
                data: nil,
                timestamp: nextTimestamp,
                priority: DCAExecutor.schedulerPriority,
                executionEffort: DCAExecutor.executionEffort,
                fees: <-fees
            )
        }

        // -----------------------------------------------------------------
        // Admin: top up fee reserve
        // -----------------------------------------------------------------

        access(all) fun topUpReserve(from: @FlowToken.Vault) {
            self.feeReserve.deposit(from: <-from)
        }

        init(
            swapPath: [String],
            usdcWithdrawCap: Capability<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>,
            outputDepositCap: Capability<&{FungibleToken.Vault}>,
            initialFees: @FlowToken.Vault,
            handlerCap: Capability<auth(FlowTransactionScheduler.Execute)
                                   &{FlowTransactionScheduler.TransactionHandler}>,
            managerRef: Capability<auth(FlowTransactionSchedulerUtils.Owner)
                                   &{FlowTransactionSchedulerUtils.Manager}>
        ) {
            pre {
                swapPath.length >= 2: "DCAExecutor: swap path must have at least 2 elements"
                usdcWithdrawCap.check(): "DCAExecutor: invalid USDC withdraw capability"
                outputDepositCap.check(): "DCAExecutor: invalid output deposit capability"
                handlerCap.check(): "DCAExecutor: invalid handler capability"
                managerRef.check(): "DCAExecutor: invalid manager capability"
            }
            self.swapPath         = swapPath
            self.usdcWithdrawCap  = usdcWithdrawCap
            self.outputDepositCap = outputDepositCap
            self.feeReserve       = <-initialFees
            self.handlerCap       = handlerCap
            self.managerRef       = managerRef
        }
    }

    // -------------------------------------------------------------------------
    // Contract init
    // -------------------------------------------------------------------------

    init(intervalSeconds: UFix64, amountPerTick: UFix64, executionEffort: UInt64) {
        self.enabled              = true
        self.intervalSeconds      = intervalSeconds
        self.amountPerTick        = amountPerTick
        self.executionEffort      = executionEffort
        self.schedulerPriority    = FlowTransactionScheduler.Priority.Medium
        self.minFeeReserveBalance = 0.001  // keep at least 0.001 FLOW as cushion
    }

    // -------------------------------------------------------------------------
    // Admin entrypoints (call from a transaction authorized by the contract account)
    // -------------------------------------------------------------------------

    access(account) fun setEnabled(_ v: Bool) { self.enabled = v }
    access(account) fun setInterval(_ s: UFix64) { self.intervalSeconds = s }
    access(account) fun setAmountPerTick(_ a: UFix64) { self.amountPerTick = a }
    access(account) fun setExecutionEffort(_ e: UInt64) { self.executionEffort = e }
}
```

---

## Cancellation Invariants for Scheduled Compositions

### Cancelling a future tick

A tick that has already been scheduled (status = `Scheduled`) can be cancelled via
`manager.cancel(id: schedulerID)`. The caller receives 50% of the originally paid fee
as a `@FlowToken.Vault`. The non-refunded 50% is paid to `FlowFees` (node operator rewards).

Cancelling tick N does **not** affect tick N-1:
- If tick N-1 ran successfully it has already settled, emitted `Executed`, and the
  composition's side effects are committed on-chain. The USDC has been swapped; the
  output tokens are in the sink vault.
- If tick N-1 panicked, it is already finalized as a failed execution (no `Executed` event,
  fees consumed). Cancelling tick N does not change that outcome.

### The cancellation window

The scheduler enforces a strict two-state model. Once the slot timestamp has been reached,
`process()` optimistically flips the status from `Scheduled` to `Executed` **before** the
handler runs. From that moment:

- Cancellation is no longer possible (status is no longer `Scheduled`).
- Calling `manager.cancel(id:)` will panic: `Invalid ID: <id> transaction not found` because
  the post-execution entry is removed from `self.transactions` entirely.

The practical boundary:

```
Timeline:
  t=0   tick N scheduled (status = Scheduled)
  t=N   slot reached → process() flips to Executed → cancel window CLOSED
  t=N+  handler runs → composition executes (or panics)
  t=N+  if handler returns normally: Executed event emitted
  t=N+  if handler panics: Executed event NOT emitted; fees consumed
```

An admin attempting to cancel a tick that is mid-execution cannot do so: by the time the
handler is running, the status has already been flipped and the cancel target is gone.

### Refund semantics

```
refund = 0.50 × original_fee
```

This is the default `refundMultiplier` in `FlowTransactionScheduler`. It is governance-mutable;
read `FlowTransactionScheduler.getConfig()` rather than hard-coding 0.5 in UI or business logic.

---

## Failure Handling: Panic-as-Rollback Inside a Tick

### The scheduler's optimistic state machine

Per the `FlowTransactionScheduler` specification:

1. `schedule()` → status = `Scheduled`; fees moved into the scheduler account.
2. When `getCurrentBlock().timestamp >= scheduledTimestamp`, `process()` flips status to
   `Executed` **before** the handler runs.
3. `executeTransaction(id)` is called in a sub-transaction. Your handler runs.

If the handler panics:
- The Cadence runtime rolls back the sub-transaction atomically — all vault movements,
  storage writes, and event emissions within `executeTransaction` are reverted.
- The status was already flipped to `Executed` in step 2, which is in a separate transaction
  from step 3. That status flip is **not** rolled back.
- The `Executed` event is **not** emitted (it is part of the handler's sub-transaction).
- Fees are **consumed in full** — there is no partial refund on handler panic.
- **The composition's residual state is fully reverted.** Any partially swapped vault, any
  vault withdrawn from the source but not yet deposited to the sink — all of it rolls back.
  Cadence atomicity within the sub-transaction is unconditional.

### Detecting failure by event absence

The absence of `Executed` for a known `PendingExecution` id is the failure signal:

| Event emitted | Meaning |
|---|---|
| `PendingExecution` + `Executed` | Handler ran to completion normally |
| `PendingExecution` only (no `Executed`) | Handler panicked; sub-transaction rolled back |
| `Canceled` | Admin cancelled before the slot timestamp |

### ✅ Panic-as-rollback: when to use it

Use `panic` only for unrecoverable invariant violations — conditions that indicate a logic
error, not a transient market condition. The next tick will reschedule fresh; let it retry.

```cadence
// ✅ Panic on hard invariants (logic bugs, broken contract state)
assert(outVault.balance == 0.0, message: "Residual vault after deposit — logic error")

// ✅ Panic on missing required capability (indicates misconfiguration, not market condition)
let pool = self.poolCap.borrow()
    ?? panic("DCAExecutor: pool capability revoked — contract requires admin intervention")
```

### ❌ Anti-pattern: Panic for skippable conditions

```cadence
// ❌ Panics on an empty source — wastes the tick fee, no Executed event, no reschedule possible
access(FlowTransactionScheduler.Execute)
fun executeTransaction(id: UInt64, data: AnyStruct?) {
    let available = source.minimumAvailable()
    assert(available > 0.0, message: "Source is empty")  // panics and kills the tick
    // ...
}

// ✅ Return early instead — preserves Executed event and allows chain to continue
access(FlowTransactionScheduler.Execute)
fun executeTransaction(id: UInt64, data: AnyStruct?) {
    let available = source.minimumAvailable()
    if available == 0.0 {
        emit DCAExecutor.TickSkipped(...)
        self.scheduleNext()  // still reschedule for next tick
        return
    }
    // ...
}
```

---

## Per-Tick CU Budget

Every scheduled tick is a transaction and is subject to the hard **9,999 CU ceiling**.
A Source → Swapper → Sink composition uses Cadence CU for each step:

| Step | Approximate CU cost | Notes |
|---|---|---|
| `source.minimumAvailable()` | ~5–15 CU | Storage borrow + arithmetic |
| `source.withdrawAvailable(maxAmount:)` | ~20–40 CU | Vault withdraw, post-condition event |
| `swapper.swap(quote:inVault:)` | ~100–300 CU | Depends on DEX routing complexity |
| `sink.minimumCapacity()` | ~5–15 CU | Storage borrow |
| `sink.depositCapacity(from:)` | ~20–40 CU | Vault deposit, post-condition event |
| Self-reschedule (`mgr.schedule(...)`) | ~50–100 CU | Manager indexing overhead |
| Handler overhead, event emission | ~30–60 CU | UniqueIdentifier creation, events |

A simple USDC → token swap with self-reschedule runs approximately **250–600 CU** in total.
A multi-hop SequentialSwapper adds ~100 CU per hop. Budget conservatively and validate
empirically against the emulator before setting `executionEffort`.

> Cross-link: [cu-optimization.md](../../cadence-lang/references/cu-optimization.md) for the
> CU sweep measurement methodology and high-cost pattern catalogue.

### CrossVM compositions: quadratic scaling bites hard

If the Swapper routes through an EVM contract via a CrossVM call (`coa.call`), the budget
constraint is critical. State-mutating EVM calls cost **~200–500 CU** for the Cadence bridge
overhead alone (ABI encoding + decoding + COA borrow), and that cost scales with the size of
the return data. A multi-hop CrossVM swap can easily consume 1,000–3,000+ CU, leaving little
headroom for the DeFiActions pipeline overhead and self-reschedule.

> Cross-link: [flow-crossvm/references/cu-ceiling.md](../../flow-crossvm/references/cu-ceiling.md)
> for the full CrossVM CU measurement table and the multi-tx escrow pattern for work that
> exceeds 9,999 CU.

Do not schedule a CrossVM DCA executor at `executionEffort` values below your measured actual
cost, or the tick will be over-budget and fail. Always measure first:

```cadence
// Measure tick CU from the emulator before committing to executionEffort
// flow transactions send dca_tick.cdc --compute-limit 9999 --network emulator
// Then read FlowFees.FeesDeducted.amount from the sealed transaction events.
```

---

## Admin Operations: Pause, Parameter Change, Cancel Future Ticks

```cadence
// Transaction: pause the DCA chain
transaction {
    prepare(acct: auth(BorrowValue) &Account) {
        // No further tick will be rescheduled once enabled = false.
        // Already-scheduled ticks may still fire; cancel them explicitly via the Manager.
        DCAExecutor.setEnabled(false)
    }
}

// Transaction: cancel the already-scheduled next tick and receive the fee refund
transaction(nextTickSchedulerID: UInt64) {
    prepare(acct: auth(BorrowValue) &Account) {
        let mgr = acct.storage.borrow<
            auth(FlowTransactionSchedulerUtils.Owner) &{FlowTransactionSchedulerUtils.Manager}
        >(from: FlowTransactionSchedulerUtils.managerStoragePath)
            ?? panic("Manager not found")

        let refund <- mgr.cancel(id: nextTickSchedulerID)
        // Deposit refund to the operator's FLOW vault (50% of original fee).
        let flowVault = acct.storage.borrow<
            auth(FungibleToken.Withdraw) &FlowToken.Vault
        >(from: /storage/flowTokenVault)!
        flowVault.deposit(from: <-refund)
    }
}

// Transaction: change parameters mid-chain (takes effect on the next tick)
transaction(newInterval: UFix64, newAmount: UFix64) {
    prepare(acct: auth(BorrowValue) &Account) {
        DCAExecutor.setInterval(newInterval)
        DCAExecutor.setAmountPerTick(newAmount)
    }
}

// Transaction: top up the fee reserve inside the handler
transaction(topUpAmount: UFix64) {
    prepare(acct: auth(BorrowValue) &Account) {
        let flowVault = acct.storage.borrow<
            auth(FungibleToken.Withdraw) &FlowToken.Vault
        >(from: /storage/flowTokenVault)!
        let top <- flowVault.withdraw(amount: topUpAmount) as! @FlowToken.Vault

        let handler = acct.storage.borrow<&DCAExecutor.Handler>(
            from: /storage/dcaExecutorHandler)
            ?? panic("Handler not found")
        handler.topUpReserve(from: <-top)
    }
}
```

---

## Initial Setup Transaction

The following transaction deploys the Handler and schedules the first tick. It assumes
`DCAExecutor` is already deployed to the deployer account.

```cadence
import "FungibleToken"
import "FlowToken"
import "FlowTransactionScheduler"
import "FlowTransactionSchedulerUtils"
import "DCAExecutor"

transaction(
    swapPath: [String],           // IncrementFi pool route, e.g. ["A.b19436aae4d94622.USDC", "A.1654653399040a61.FlowToken"]
    usdcStoragePath: StoragePath,
    outputStoragePath: StoragePath,
    intervalSeconds: UFix64,
    initialFeeTopUp: UFix64,
    executionEffort: UInt64
) {
    prepare(acct: auth(
        BorrowValue, SaveValue,
        IssueStorageCapabilityController,
        PublishCapability,
        GetStorageCapabilityController
    ) &Account) {

        // 1. Create Manager if needed.
        if !acct.storage.check<@{FlowTransactionSchedulerUtils.Manager}>(
            from: FlowTransactionSchedulerUtils.managerStoragePath) {
            acct.storage.save(
                <-FlowTransactionSchedulerUtils.createManager(),
                to: FlowTransactionSchedulerUtils.managerStoragePath
            )
            let mgrCap = acct.capabilities.storage.issue<&{FlowTransactionSchedulerUtils.Manager}>(
                FlowTransactionSchedulerUtils.managerStoragePath
            )
            acct.capabilities.publish(mgrCap, at: FlowTransactionSchedulerUtils.managerPublicPath)
        }
        let ownerMgrCap = acct.capabilities.storage.issue<
            auth(FlowTransactionSchedulerUtils.Owner) &{FlowTransactionSchedulerUtils.Manager}
        >(FlowTransactionSchedulerUtils.managerStoragePath)

        // 2. Issue capabilities for the source and sink vaults.
        let usdcCap = acct.capabilities.storage.issue<
            auth(FungibleToken.Withdraw) &{FungibleToken.Vault}
        >(usdcStoragePath)
        let outputCap = acct.capabilities.storage.issue<&{FungibleToken.Vault}>(outputStoragePath)

        // 3. Withdraw initial fee top-up for the handler reserve.
        let flowVault = acct.storage.borrow<auth(FungibleToken.Withdraw) &FlowToken.Vault>(
            from: /storage/flowTokenVault) ?? panic("No FLOW vault")
        let initialFees <- flowVault.withdraw(amount: initialFeeTopUp) as! @FlowToken.Vault

        // 4. Issue a placeholder handlerCap — will be set properly after save.
        //    (Pattern: save first, then issue capability against /storage/dcaExecutorHandler.)
        //    To avoid the chicken-and-egg problem, pass a self-referencing capability stored
        //    at a known path. The handler init validates the cap via check().
        let handlerPath: StoragePath = /storage/dcaExecutorHandler
        let handlerExecCap = acct.capabilities.storage.issue<
            auth(FlowTransactionScheduler.Execute)
            &{FlowTransactionScheduler.TransactionHandler}
        >(handlerPath)

        // 5. Create and save the Handler.
        let handler <- create DCAExecutor.Handler(
            swapPath: swapPath,
            usdcWithdrawCap: usdcCap,
            outputDepositCap: outputCap,
            initialFees: <-initialFees,
            handlerCap: handlerExecCap,
            managerRef: ownerMgrCap
        )
        acct.storage.save(<-handler, to: handlerPath)

        // 6. Publish a public (un-entitled) capability for view resolution and tooling.
        let publicCap = acct.capabilities.storage.issue<
            &{FlowTransactionScheduler.TransactionHandler}
        >(handlerPath)
        acct.capabilities.publish(publicCap, at: /public/dcaExecutorHandler)

        // 7. Estimate fee and schedule the first tick.
        let firstTimestamp = getCurrentBlock().timestamp + intervalSeconds
        let estimate = FlowTransactionScheduler.estimate(
            handlerCap: handlerExecCap,
            data: nil,
            timestamp: firstTimestamp,
            priority: FlowTransactionScheduler.Priority.Medium,
            executionEffort: executionEffort
        )
        assert(estimate.error == nil, message: "Cannot schedule first tick: ".concat(estimate.error!))

        let tickFees <- flowVault.withdraw(amount: estimate.flowFee!) as! @FlowToken.Vault
        let mgr = acct.storage.borrow<
            auth(FlowTransactionSchedulerUtils.Owner) &{FlowTransactionSchedulerUtils.Manager}
        >(from: FlowTransactionSchedulerUtils.managerStoragePath)!

        let _ = mgr.schedule(
            handlerCap: handlerExecCap,
            data: nil,
            timestamp: firstTimestamp,
            priority: FlowTransactionScheduler.Priority.Medium,
            executionEffort: executionEffort,
            fees: <-tickFees
        )
    }
}
```

---

## Common Pitfalls

### Quadratic CU scaling for CrossVM swappers

A CrossVM Swapper (one that calls an EVM DEX) encodes ABI calldata and decodes the EVM result
on the Cadence side. The CU cost of ABI decoding is super-linear in the size of the return
data. On a scheduled tick with a hard 9,999 CU budget, a multi-pool EVM swap path that returns
a large `bytes[]` response can exhaust the budget entirely, leaving no room for the Sink deposit
and self-reschedule. Verify CU consumption with `flow transactions send --compute-limit 9999`
on the emulator **before** deploying to testnet.

### Fee depletion terminates the chain silently

The handler withdraws from its internal `feeReserve` vault on every reschedule. When that
vault runs low the handler stops rescheduling. If `minFeeReserveBalance` is too small, the
last reschedule may succeed while leaving insufficient balance for one more tick, causing the
chain to terminate with no event other than `ChainStopped`. Monitor `feeReserve.balance`
off-chain via scripts and top up before it hits the minimum.

### UniqueIdentifier per tick, not per handler

See the top of this document. Using one `UniqueIdentifier` stored at handler init and reused
across ticks makes `Withdrawn`, `Swapped`, and `Deposited` events indistinguishable in an
indexer. Always call `DeFiActions.createUniqueIdentifier()` at the top of `executeTransaction`.

### Panic instead of early return for skippable conditions

Panicking on a skippable condition (empty source, full sink, disabled flag) costs the full
tick fee, suppresses the `Executed` event, and prevents the self-reschedule at the bottom of
`executeTransaction` from running — terminating the chain. Use early `return` for all
non-error conditions; reserve `panic` for logic bugs and unrecoverable state violations.

### Not calling `FlowTransactionScheduler.estimate()` before rescheduling

A slot that is full at reschedule time causes `schedule()` to panic — which happens inside
`executeTransaction`, rolling back the entire tick including the composition side effects that
already ran. Always call `estimate()` first; if the slot is full, emit `ChainStopped` and
return cleanly rather than panicking.

### Trying to cancel the in-flight tick from within itself

Once `executeTransaction(id:data:)` is running, the scheduler has already removed the entry
for `id` from its internal map. Calling `manager.cancel(id:)` inside the handler panics with
`Invalid ID: <id> transaction not found`. The only valid cancellation target from within the
handler is the **next** tick's `id` (returned by `mgr.schedule(...)` at the bottom of
`scheduleNext()`).

---

## Cross-links

- [composition-patterns.md](composition-patterns.md) — canonical Source → Swapper → Sink
  composition, SwapSource/SwapSink wrappers, and token-order reversal rules
- [source-interface.md](source-interface.md) — `minimumAvailable`, `withdrawAvailable`,
  weak-guarantee semantics
- [sink-interface.md](sink-interface.md) — `minimumCapacity`, `depositCapacity`, residual-vault
  pattern
- [swapper-interface.md](swapper-interface.md) — `quoteIn`/`quoteOut`/`swap`/`swapBack`,
  token ordering, SequentialSwapper for multi-hop
- [price-oracle-interface.md](price-oracle-interface.md) — optional oracle integration for
  minimum-output guards before the swap step
- [../../cadence-lang/references/scheduled-transactions.md](../../cadence-lang/references/scheduled-transactions.md)
  — canonical FlowTransactionScheduler API: `TransactionHandler` interface, fee model,
  priority levels, cancellation, failure state machine, CU ceiling table
- [../../cadence-lang/references/cu-optimization.md](../../cadence-lang/references/cu-optimization.md)
  — CU budgeting methodology, high-cost patterns, sweep measurement technique
- [../../flow-crossvm/references/cu-ceiling.md](../../flow-crossvm/references/cu-ceiling.md)
  — CrossVM CU costs per operation; quadratic scaling of EVM state-mutating calls; applies
  directly to CrossVM Swappers inside scheduled handlers
- [../../cadence-audit/references/forte-anti-patterns.md](../../cadence-audit/references/forte-anti-patterns.md)
  — scheduled-tx anti-patterns: unbounded self-rescheduling (A1), handler panic as silent
  failure (A6), reserve depletion termination; the `DCAExecutor` pattern above is designed
  to avoid all three
