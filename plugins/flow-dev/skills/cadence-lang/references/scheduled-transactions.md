# Scheduled Transactions (`FlowTransactionScheduler`)

Scheduled Transactions let a Cadence resource "wake itself up" at a future
timestamp and execute logic without any external trigger. They shipped with the
**Forte** network upgrade on Mainnet on **Oct 22, 2025** and are intended to
replace off-chain keepers, cron servers, and relayers for on-chain automation
patterns (recurring payments, scheduled liquidations, sweeps, DCA, vesting
unlocks, periodic rebalances). Reach for them when the trigger is *time*; reach
for events / polling when the trigger is *state*.

## When to choose a scheduled transaction

| Trigger type | Use |
|---|---|
| "Run X at timestamp T" or "every N seconds, on-chain" | **Scheduled transaction** |
| "Run X when on-chain state Y changes" (price tick, deposit, etc.) | Event subscription + off-chain submitter, or a hook inside the state-changing tx |
| "Run X when an external API says so" | Off-chain keeper (Gelato-style) or oracle push |
| "Run X exactly atomically with another action" | Same transaction (or Flow Actions composition), **not** a scheduled tx |

Scheduled txs are not free, not instant on the second they're scheduled for, and
not guaranteed except for `High` priority. They are guaranteed *eventually* (or
to fail loudly and refund nothing) — design around that.

## Contract addresses

- Emulator: `0xf8d6e0586b0a20c7` (service account)
- Testnet: `0x8c5303eaa26202d6`
- Mainnet: deployed to the service account (resolve via `flow.json` deps)

Import path (Cadence 1.0):

```cadence
import "FlowTransactionScheduler"
import "FlowTransactionSchedulerUtils" // optional, recommended Manager wrapper
```

## The `TransactionHandler` interface

Every scheduled transaction is backed by a **resource** in your storage that
conforms to `FlowTransactionScheduler.TransactionHandler`. The protocol calls
`executeTransaction` on this resource through an entitled capability you
created at schedule time.

```cadence
access(all) resource interface TransactionHandler: ViewResolver.Resolver {
    access(all) view fun getViews(): [Type] { return [] }
    access(all) fun resolveView(_ view: Type): AnyStruct? { return nil }

    /// Called by the FVM when the scheduled timestamp is reached.
    /// `id`   — the scheduled-tx id (useful for idempotency tracking).
    /// `data` — whatever AnyStruct you passed to schedule().
    access(Execute) fun executeTransaction(id: UInt64, data: AnyStruct?)
}
```

Minimal handler:

```cadence
import "FlowTransactionScheduler"
import "Counter"

access(all) contract MyAutomation {
    access(all) resource Handler: FlowTransactionScheduler.TransactionHandler {
        access(FlowTransactionScheduler.Execute)
        fun executeTransaction(id: UInt64, data: AnyStruct?) {
            // Keep this body idempotent and effort-bounded. See "Failure handling" below.
            Counter.increment()
        }
    }
    access(all) fun createHandler(): @Handler { return <- create Handler() }
}
```

Save the handler and issue **two** capabilities — one entitled `Execute` for
the scheduler, one un-entitled for public view resolution:

```cadence
let h <- MyAutomation.createHandler()
acct.storage.save(<-h, to: /storage/myHandler)

let execCap = acct.capabilities.storage
    .issue<auth(FlowTransactionScheduler.Execute) &{FlowTransactionScheduler.TransactionHandler}>(/storage/myHandler)

let publicCap = acct.capabilities.storage
    .issue<&{FlowTransactionScheduler.TransactionHandler}>(/storage/myHandler)
acct.capabilities.publish(publicCap, at: /public/myHandler)
```

## Scheduling a transaction

The contract-level entry point:

```cadence
access(all) fun schedule(
    handlerCap: Capability<auth(Execute) &{TransactionHandler}>,
    data: AnyStruct?,
    timestamp: UFix64,         // Unix seconds; fractional part is dropped
    priority: Priority,        // High | Medium | Low
    executionEffort: UInt64,   // CU budget for the callback
    fees: @FlowToken.Vault     // must cover the calculated fee
): @ScheduledTransaction
```

You receive a `@ScheduledTransaction` resource — store it (or use the `Manager`
wrapper below). It's the cancellation handle. Losing it means the transaction
cannot be canceled by anyone.

Recommended pattern via `FlowTransactionSchedulerUtils.Manager` (handles
storage, indexing by handler/timestamp, and cleanup):

```cadence
import "FlowTransactionScheduler"
import "FlowTransactionSchedulerUtils"
import "FlowToken"
import "FungibleToken"

transaction(delaySeconds: UFix64, effort: UInt64, feeAmount: UFix64) {
    prepare(acct: auth(BorrowValue, SaveValue, IssueStorageCapabilityController,
                       PublishCapability, GetStorageCapabilityController) &Account) {

        // 1. Ensure Manager exists.
        if !acct.storage.check<@{FlowTransactionSchedulerUtils.Manager}>(
            from: FlowTransactionSchedulerUtils.managerStoragePath) {
            acct.storage.save(<-FlowTransactionSchedulerUtils.createManager(),
                              to: FlowTransactionSchedulerUtils.managerStoragePath)
            let mgrCap = acct.capabilities.storage.issue<&{FlowTransactionSchedulerUtils.Manager}>(
                FlowTransactionSchedulerUtils.managerStoragePath)
            acct.capabilities.publish(mgrCap, at: FlowTransactionSchedulerUtils.managerPublicPath)
        }

        // 2. Borrow handler cap (already issued in a prior tx).
        let handlerCap = acct.capabilities.storage
            .getControllers(forPath: /storage/myHandler)[0]
            .capability as! Capability<auth(FlowTransactionScheduler.Execute)
                                       &{FlowTransactionScheduler.TransactionHandler}>

        // 3. Withdraw fees.
        let vault = acct.storage.borrow<auth(FungibleToken.Withdraw) &FlowToken.Vault>(
            from: /storage/flowTokenVault) ?? panic("no FLOW vault")
        let fees <- vault.withdraw(amount: feeAmount) as! @FlowToken.Vault

        // 4. Schedule.
        let mgr = acct.storage.borrow<auth(FlowTransactionSchedulerUtils.Owner)
                                      &{FlowTransactionSchedulerUtils.Manager}>(
            from: FlowTransactionSchedulerUtils.managerStoragePath)!
        let id = mgr.schedule(
            handlerCap: handlerCap,
            data: nil,
            timestamp: getCurrentBlock().timestamp + delaySeconds,
            priority: FlowTransactionScheduler.Priority.Medium,
            executionEffort: effort,
            fees: <-fees,
        )
        log("scheduled id=".concat(id.toString()))
    }
}
```

## Priority levels and slot model

Each Unix-second slot has a 17,500 CU total budget split as: 10,000 CU
reserved for High, 2,500 CU reserved for Medium, a 5,000 CU pool shared
between High and Medium, and 2,500 CU for Low. A Low-priority tx can be
pushed forward by *any* other tx whose effort would exceed the slot total
after exclusive reserves are filled. High and Medium do not displace each
other, but they compete for the shared 5,000 CU.

| Priority | Default slot cap (CU) | Multiplier | Scheduling guarantee |
|---|---|---|---|
| `High`   | 15,000 | **10x** | Executes at the requested timestamp's first block, or `schedule()` panics |
| `Medium` |  7,500 |  **5x** | If slot full, shifts forward one second until a slot has room |
| `Low`    |  2,500 |  **2x** | Same shift-forward behavior; opportunistic when block capacity allows |

Per-transaction caps (from contract defaults, governance-mutable):

- `maximumIndividualEffort` = **9,999** CU (the absolute per-tx ceiling)
- `minimumExecutionEffort`  = **100** CU
- `maxDataSizeMB`           = **0.001** MB (≈ 1 KB of `AnyStruct?` data)
- Per-collection cap: `collectionEffortLimit` = **500,000** CU,
  `collectionTransactionsLimit` = **150** txs

> Governance can change these; check `FlowTransactionScheduler.getConfig()`
> on the target network before quoting hard numbers in app UI.

Use `FlowTransactionScheduler.estimate(...)` before scheduling to surface error
strings ("priority slot full", "below minimum effort", etc.) without spending
gas on a panic.

For `priority=Low`, `estimate()` always returns a non-nil `error` string
("Cannot estimate for Low Priority...") even on success — check
`flowFee != nil` to determine whether scheduling is feasible.

## Fee model

The contract's `calculateFee` is:

```
fee = FlowFees.computeFees(inclusionEffort: 1.0,
                           executionEffort: effort / 100_000_000.0)
      * priorityFeeMultipliers[priority]
      + FlowStorageFees.storageCapacityToFlow(dataSizeMB)
      + 0.00001                                  // flat scheduled-tx inclusion fee
```

Fees are **paid upfront** in `@FlowToken.Vault` and held by the scheduler
contract account until execution or cancellation. Unused effort is **not**
refunded — pay only for the effort you reasonably expect to use.

### Worked example

Schedule a Medium-priority callback with `executionEffort = 1000` CU and no
data:

1. Base effort (in Flow's UFix64 effort units): `1000 / 100_000_000 = 0.00001`
2. `baseFee = FlowFees.computeFees(1.0, 0.00001)` returns **0.00001 FLOW
   flat** on emulator and on Mainnet at idle (where `surgeFactor=1.0` and
   `executionEffortCost ≈ 0`, so the inclusion component dominates and the
   execution component is below UFix64's 8-decimal precision for any
   effort < 9999).
3. Scale by Medium multiplier (5x): `0.00001 * 5 = 0.00005`
4. Storage fee: `0` (no data)
5. Add scheduled-tx inclusion fee `+ 0.00001`
6. **Total = 0.00006 FLOW** (Medium); 0.00011 FLOW (High); 0.00003 FLOW
   (Low) — measured.

Always call `estimate()` or `calculateFee()` from a script and pass the
returned amount (with a small headroom) into your scheduling transaction.

Under surge>1 the baseFee scales linearly but the inclusion/storage
components do not, so the total fee on a busy Mainnet scales sub-linearly
with surge.

## Cancellation

Two paths:

1. **Via the `ScheduledTransaction` resource** — pass it back to
   `FlowTransactionScheduler.cancel(scheduledTx: <-tx)` and receive a
   `@FlowToken.Vault` with the refunded portion of the fees.
2. **Via the Manager** — `manager.cancel(id: UInt64)` looks up the resource
   and forwards.

```cadence
let refund <- mgr.cancel(id: scheduledId)
vault.deposit(from: <-refund)
```

### Refund semantics

- `refundMultiplier` defaults to **0.5 (50%)** of the originally paid fee.
- The non-refunded half is deposited to `FlowFees` (node operator rewards),
  not burned.
- Cancellation is only valid while `status == Scheduled`. After
  `PendingExecution` (the slot timestamp has been reached and the tx is in
  the pending queue), the optimistic state transition to `Executed` has
  already happened — you can no longer cancel.
- The status transition rules are enforced by `TransactionData.setStatus`:
  `Scheduled -> Executed` and `Scheduled -> Canceled` are the only legal
  moves; finalized statuses are immutable.

### Ownership invariants

- Only the holder of the `@ScheduledTransaction` resource (or, via a Manager,
  the holder of the `auth(Owner) &Manager` reference) can cancel.
- The handler capability holder is **separate** from the cancellation
  capability holder. A contract that issues a handler capability to the
  scheduler does **not** thereby get cancel rights.
- Fees once deposited belong to the scheduler contract until `payAndRefundFees`
  is called. There is no way to "top up" or "modify" a scheduled tx — cancel
  and re-schedule instead.

## Per-tick / per-slot CU ceilings

Read these numbers as defaults; `getConfig()` is authoritative at runtime.

| Limit | Default | Meaning |
|---|---|---|
| `maximumIndividualEffort` | 9,999 CU | Single-tx cap. `executionEffort` greater than this is rejected at schedule time. |
| `priorityEffortLimit[High]` | 15,000 CU | Cumulative High-priority CU per slot |
| `priorityEffortLimit[Medium]` | 7,500 CU | Cumulative Medium per slot |
| `priorityEffortLimit[Low]` | 2,500 CU | Cumulative Low per slot |
| `slotSharedEffortLimit` | 5,000 CU | Shared pool between High and Medium per slot |
| `priorityEffortReserve` | `{High: 10_000, Medium: 2_500, Low: 0}` | Per-priority exclusive reserve per slot |
| `collectionEffortLimit` | 500,000 CU | Cumulative effort across a *collection* (≈ a block's worth of scheduled txs) |
| `collectionTransactionsLimit` | 150 | Cap on number of scheduled txs processed in one collection — additional ones spill to the next collection |

Implications:

- The biggest single callback you can schedule is **9,999 CU**, even on
  High priority (whose slot cap is 15,000). Two near-max High txs in the same
  second require splitting work across two seconds or two priorities.
- `priorityEffortLimit[High] = 15,000` is **not** an independent budget — it
  equals the 10,000 CU High reserve plus the 5,000 CU shared pool. The per-priority
  caps in the table do **not** sum to the 17,500 CU slot total; the shared pool
  is double-counted in High's 15,000 and Medium's 7,500.
- Low-priority txs can be **displaced (pushed to a later slot) by any higher-priority
  traffic** that exceeds the slot's available capacity after exclusive reserves are
  filled. The scheduler calls `rescheduleLowPriorityTransactions` to make room. Low
  txs do not pre-empt each other and have no shared pool access.
- Block-level back-pressure is governed by `collectionEffortLimit` /
  `collectionTransactionsLimit` — if a block is congested, the scheduler emits
  `CollectionLimitReached` and rolls the excess into the next block. Your tx is
  not dropped; it slides.

Read the live config in a script (note the intersection-type return):

```cadence
import "FlowTransactionScheduler"
access(all) fun main(): {FlowTransactionScheduler.SchedulerConfig} {
    return FlowTransactionScheduler.getConfig()
}
```

## Failure handling

This is the most important section and the part docs gloss over. The relevant
state machine, taken from the contract:

1. `schedule()` -> status = `Scheduled`, fees moved into scheduler account.
2. When `getCurrentBlock().timestamp >= scheduledTimestamp`, the FVM calls
   `SharedScheduler.process()`, which:
   - Removes already-executed entries (charging full fee, no refund),
   - Builds `pendingQueue`, emits `PendingExecution` per tx,
   - **Optimistically flips the status to `Executed` *before* the handler runs.**
3. The FVM then calls `SharedScheduler.executeTransaction(id)` in a **separate
   sub-transaction**, which borrows the handler and calls
   `handler.executeTransaction(id, data)`.

Why this matters:

- **The scheduler does not retry on panic.** The status was set to `Executed`
  in `process()` before your handler ran. If your handler reverts, the
  `Executed` event for that id will still *not* be emitted (because the
  emitting call panicked alongside it), but the tx will not be re-queued and
  the fees are kept.
- The optimistic-Executed flip is deliberate: it prevents the handler from
  re-entrantly seeing itself as `Scheduled` and from racing with concurrent
  block production. Don't fight it.
- A handler that panics consumes its full paid effort and forfeits the
  remaining 50% it would have gotten on a clean cancellation.

### Idempotent handler pattern

✅ Treat `executeTransaction(id, data)` as "may be called at most once, may
fail." Store a small flag in your contract for any side effect that absolutely
must not happen twice if you ever re-schedule the same logical job:

```cadence
access(all) resource Handler: FlowTransactionScheduler.TransactionHandler {
    access(FlowTransactionScheduler.Execute)
    fun executeTransaction(id: UInt64, data: AnyStruct?) {
        // (1) Defensive: bail out cheaply if pre-conditions aren't satisfied.
        if !MyApp.canRunNow() { return }   // do NOT panic — that wastes the fee
                                            // and emits no Executed event

        // (2) Wrap risky inner work so a partial failure doesn't blow up
        //     the whole callback.
        let ok = MyApp.tryStep()           // returns Bool, never panics
        if !ok { MyApp.markFailed(id: id) }

        // (3) Re-schedule the next tick *last*. If self-rescheduling, do it
        //     after the side effects, with fresh fees pulled from a vault
        //     this resource owns.
        if MyApp.shouldContinue() {
            MyApp.scheduleNext(handlerCap: MyApp.ownHandlerCap())
        }
    }
}
```

❌ Anti-patterns:

```cadence
// ❌ panic inside executeTransaction — wastes fees, no retry, no Executed event
panic("bad input")

// ❌ assume the callback fires exactly at `scheduledTimestamp` for Medium/Low
let nowMustEqualScheduled = ...           // it MAY have slipped forward

// ❌ try to cancel the in-flight callback from within itself
manager.cancel(id: id)                    // panic: Invalid ID: <id> transaction not found
                                          // (post-execution entry is removed entirely;
                                          //  borrowTransaction returns nil)
```

### Retry / backoff pattern

The scheduler will not retry, so your handler must do it. Two idioms:

- **Self-reschedule on partial failure** — at the end of `executeTransaction`,
  if work remains, call `FlowTransactionScheduler.schedule(...)` again with a
  later timestamp and exponential backoff. Pull fresh fees from a vault the
  handler owns (the handler resource can hold a `@FlowToken.Vault` reserve).
- **External monitor** — emit a custom event from your contract on partial
  failure; an off-chain (or another scheduled) watcher submits a remedial tx.

For recurring jobs (cron-style), the canonical pattern is: the handler
re-schedules itself at the bottom of every successful execution. Be sure to
fence that with a "kill switch" boolean in your contract so an upgrade or
emergency can stop the chain reaction.

## Events

The scheduler emits four lifecycle events you can subscribe to:

| Event | Fired when | Key fields |
|---|---|---|
| `Scheduled` | `schedule()` succeeds | `id, priority, timestamp, executionEffort, fees, transactionHandlerOwner, transactionHandlerTypeIdentifier, transactionHandlerUUID, transactionHandlerPublicPath` |
| `PendingExecution` | Slot reached, queued in current collection | `id, priority, executionEffort, fees, transactionHandlerOwner, transactionHandlerTypeIdentifier` (note: type identifier intentionally empty here to keep `process()` panic-proof) |
| `Executed` | Handler call returned normally | `id, priority, executionEffort, transactionHandlerOwner, transactionHandlerTypeIdentifier, transactionHandlerUUID, transactionHandlerPublicPath` |
| `Canceled` | `cancel()` called while `Scheduled` | `id, priority, feesReturned, feesDeducted, transactionHandlerOwner, transactionHandlerTypeIdentifier` |

Operational events: `CollectionLimitReached`, `RemovalLimitReached`,
`ConfigUpdated`, `CriticalIssue` — useful for indexers and dashboards.

> Important: `Executed` is **not** emitted if the handler panics. Use the
> *absence* of `Executed` for a `PendingExecution` id as the failure signal.

> Note: `transactionHandlerPublicPath` is `nil` unless the handler implements
> `resolveView(Type<PublicPath>())`. Handler-emitted events from a panicking
> `executeTransaction` are rolled back along with the `Executed` event —
> indexers should not assume "if I see my custom event, the handler
> succeeded"; confirm `Executed` was emitted with a matching id.

## Flow CLI helpers

The CLI ships a `flow schedule` subcommand (CLI ≥ v2.17):

```bash
flow schedule setup --signer my-account               # one-time Manager install
flow schedule list   my-account --network testnet     # list scheduled txs
flow schedule get    <transaction-id>                 # details by scheduler id
flow schedule cancel <transaction-id> --signer my-account
```

The setup transaction it runs is equivalent to the `prepare` block of
`schedule_transaction.cdc` in
[`onflow/flow-core-contracts/transactions/transactionScheduler/`](https://github.com/onflow/flow-core-contracts/tree/master/transactions/transactionScheduler).

As of Flow CLI v2.17.1, `flow schedule setup` creates the Manager resource
but does not publish the public capability, so `flow schedule list` will
fail until you also `acct.capabilities.publish(...)` against
`FlowTransactionSchedulerUtils.managerPublicPath`. Track upstream issue or
use a custom setup tx.

`flow schedule list / get / cancel` only operate on transactions scheduled
via the Manager. Direct `FlowTransactionScheduler.schedule(...)` results are
invisible to the CLI.

`flow schedule get` takes the numeric scheduler id (UInt64), not a Flow
transaction hash. The CLI `--help`'s `0x1234...` example is misleading.

## Common pitfalls

- **Forgetting the `Execute` entitlement on the handler capability.** The
  scheduler will reject the capability at `schedule()` time. Issue with
  `auth(FlowTransactionScheduler.Execute) &{...TransactionHandler}`.
- **Issuing only one capability.** You need a separate un-entitled public
  capability for view resolution / indexers; otherwise tooling can't read
  your handler's display metadata.
- **Paying with a vault that doesn't have enough FLOW.** `schedule()` asserts
  `fees.balance >= estimate.flowFee`. Call `estimate()` first.
- **Scheduling past-or-present timestamps.** `schedule()` panics with
  "timestamp is in the past." Use `getCurrentBlock().timestamp + N`.
- **Putting too much data in `data: AnyStruct?`.** Default cap is **0.001 MB**.
  Pass small structs; for large payloads, store on-chain and pass an ID.
- **Calling `panic` in the handler for ordinary "skip this tick" logic.** That
  costs the full fee and leaves no `Executed` event. Use early `return`
  instead, and reserve `panic` for unrecoverable invariant violations.

> See canonical treatment in [../cadence-audit/references/forte-anti-patterns.md](../cadence-audit/references/forte-anti-patterns.md) A6.
> This entry is a context-specific summary; updates to the underlying behavior should land in the canonical file first.
- **Assuming retries.** The scheduler never retries. Re-schedule from within
  the handler if you want recurrence.
- **Trying to cancel during `executeTransaction`.** The post-execution entry
  is removed entirely from `self.transactions`, so `borrowTransaction(id:)`
  returns nil and `cancel(id:)` panics with `Invalid ID: <id> transaction not
  found` — not a status-mismatch error.
- **Hard-coding the 50% refund or 9,999-CU cap in business logic.** Both are
  governance-mutable. Read `getConfig()` if it matters.
- **Relying on `scheduledTimestamp` equaling your requested time for
  Medium/Low.** The estimate result tells you the real assigned timestamp —
  use that as the source of truth.
- **High-priority over-confidence.** High guarantees *first block at the
  requested second*, but only if the slot wasn't full at `schedule()` time.
  The 15,000-CU High pool can be exhausted by other apps; estimate first.

## Decision matrix: scheduled vs alternatives

| Need | Pick | Why |
|---|---|---|
| Cron-style "every N seconds" job, fully on-chain | Scheduled tx, self-rescheduling | No off-chain infra; auditable |
| One-shot delayed action ("auction ends in 24h") | Scheduled tx, single High-pri schedule | Cheapest, deterministic time |
| State-triggered ("when oracle price < X") | Off-chain keeper or in-tx hook | Scheduler can't watch state |
| Atomic multi-step within one tx | Flow Actions (same tx) | Scheduling is async by definition |
| Best-effort "soon-ish" cleanup | Low priority | Cheapest multiplier (2x) |
| Tight deadline ("settle at exactly T") | High priority | Only level with timestamp guarantee |

## References

- Contract: `onflow/flow-core-contracts/contracts/FlowTransactionScheduler.cdc`
- Manager: `onflow/flow-core-contracts/contracts/FlowTransactionSchedulerUtils.cdc`
- Example txs: `onflow/flow-core-contracts/transactions/transactionScheduler/`
- Tutorial: `developers.flow.com/blockchain-development-tutorials/forte/scheduled-transactions/scheduled-transactions-introduction`
- API page: `developers.flow.com/build/cadence/advanced-concepts/scheduled-transactions`
- FLIP 330 (Scheduled Callbacks design) — `onflow/flips`
- Forte announcement (Oct 22, 2025) — `flow.com/post/the-forte-network-upgrade-now-live-on-flow`
- Related references in this skill: `cadence-lang/references/cu-optimization.md`
  (currently in open PR #34 — assume effectively in main) for tuning
  `executionEffort` budgets.
