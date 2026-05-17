# Forte / Scheduled-Transaction Anti-Patterns

Scheduled transactions (`FlowTransactionScheduler`, Forte upgrade, October 2025) introduce a failure surface that ordinary Cadence audits do not cover: the scheduler is asynchronous, never retries, optimistically flips `status = Executed` *before* the handler runs, and refunds only 50% on cancel. Any contract that relies on a handler firing exactly once at exactly its requested timestamp is wrong; any contract that holds state across ticks without a state machine is race-prone. Cross-reference [scheduled-transactions.md](../../cadence-lang/references/scheduled-transactions.md) for the canonical API.

---

## A1 — Unbounded self-rescheduling (Critical)

Handler reschedules itself with no kill switch, no counter, no deadline. Chain runs until the fee reserve drains.

### Bad

```cadence
// BAD
access(FlowTransactionScheduler.Execute)
fun executeTransaction(id: UInt64, data: AnyStruct?) {
    RecurringJob.doWork()
    let fees <- RecurringJob.reserve.withdraw(amount: 0.00006) as! @FlowToken.Vault
    FlowTransactionScheduler.schedule(
        handlerCap: RecurringJob.handlerCap, data: nil,
        timestamp: getCurrentBlock().timestamp + 60.0,
        priority: FlowTransactionScheduler.Priority.Medium,
        executionEffort: 1000, fees: <-fees,
    )
}
```

### Why it's bad

A self-rescheduling chain has no operator brake. A bug in `doWork()` keeps firing until governance can upgrade or the reserve empties. Canceling tick N does not stop tick N+1 — tick N+1 hasn't been scheduled yet.

The reserve vault is a silent terminator. The handler funds each reschedule by withdrawing from a finite reserve (`RecurringJob.reserve` above); when that reserve drops below the next tick's fee the handler will `panic` inside `withdraw`. That panic is an instance of A6 (handler panic = silent data loss) but with a more concrete cause: the fees of the *current* tick were already paid, so the chain breaks **inside the handler** — no `Executed` event, no retry, the work that should have happened this tick is lost. Empirically verified on emulator: a deposit handler funded with 1000 FLOW drained after a handful of multi-hundred-FLOW ticks and subsequent scheduled deposits panicked. Reserves are themselves a state machine that can hit zero.

### Correct

```cadence
// GOOD
access(all) contract RecurringJob {
    access(all) var enabled: Bool          // admin kill switch
    access(all) var tickCount: UInt64
    access(all) let maxTicks: UInt64       // hard ceiling
    access(all) let minReserveBalance: UFix64  // refuse to reschedule below this

    access(FlowTransactionScheduler.Execute)
    fun executeTransaction(id: UInt64, data: AnyStruct?) {
        if !RecurringJob.enabled { return }
        if RecurringJob.tickCount >= RecurringJob.maxTicks { return }
        // Refuse to drain the reserve below the next tick's funding need.
        // Returns cleanly (no panic) so this tick's Executed event fires.
        if RecurringJob.reserve.balance < RecurringJob.minReserveBalance { return }
        RecurringJob.doWork()
        RecurringJob.tickCount = RecurringJob.tickCount + 1
        if RecurringJob.enabled && RecurringJob.tickCount < RecurringJob.maxTicks {
            RecurringJob.scheduleNext()
        }
    }
    access(account) fun setEnabled(_ v: Bool) { self.enabled = v }
}
```

### Detection hint

Grep `FlowTransactionScheduler.schedule(` inside any `executeTransaction`. Require (a) a bool/admin gate read before the reschedule, (b) a monotonically advancing counter or deadline, and (c) a balance check on the funding reserve that exits cleanly (early `return`) when the reserve is below the next tick's fee. Missing any of the three is the smell.

---

## A2 — Shared state across ticks without a state machine (High)

Resource holds funds or partial work between ticks but has no phase ordering. Manual transactions interleave with scheduled ticks and observe torn state.

### Bad

```cadence
// BAD
access(all) resource Escrow {
    access(self) var vault: @FlowToken.Vault
    access(all) fun deposit(funds: @FlowToken.Vault) { self.vault.deposit(from: <-funds) }
    access(all) fun claim(): @FlowToken.Vault {
        return <- self.vault.withdraw(amount: self.vault.balance) as! @FlowToken.Vault
    }
}
```

### Why it's bad

Nothing prevents `claim()` between two scheduled ticks or twice in the same block via different paths. The scheduler is not a mutex: a manual tx at height H and a scheduled tick at H execute in collection-order, not handler-order. The escrow drains, the handler then sees `balance = 0` and either panics (A6) or silently no-ops.

### Correct

```cadence
// GOOD
access(all) enum Phase: UInt8 {
    access(all) case Funding
    access(all) case Processing
    access(all) case Claimable
    access(all) case Closed
}
access(all) resource Escrow {
    access(self) var vault: @FlowToken.Vault
    access(self) var phase: Phase

    access(all) fun deposit(funds: @FlowToken.Vault) {
        pre { self.phase == Phase.Funding: "deposit only in Funding" }
        self.vault.deposit(from: <-funds)
    }
    access(FlowTransactionScheduler.Execute) fun advance() {
        pre { self.phase == Phase.Processing: "advance only in Processing" }
        self.phase = Phase.Claimable
    }
    access(Claim) fun claim(): @FlowToken.Vault {
        pre { self.phase == Phase.Claimable: "not claimable yet" }
        self.phase = Phase.Closed
        return <- self.vault.withdraw(amount: self.vault.balance) as! @FlowToken.Vault
    }
}
```

### Detection hint

For every resource touched inside `executeTransaction`, ask: "can a user transaction call these same methods?" If yes, the resource must carry an explicit phase enum (or a version counter, A3) and every mutation must phase-gate via pre-condition.

---

## A3 — Assuming serialisation between ticks (High)

Scheduling two callbacks against the same resource at adjacent timestamps and assuming the first tick's effects are observable to the second. Two or more ticks at the **same** timestamp T always land in the same collection (verified on emulator v2.17.1). Even ticks at T+1 may slip into the T collection if the T collection had headroom and the T+1 slot is congested — see priority displacement in T01-scheduled-transactions.md. Ordering with manual transactions in that block is not handler-controllable. The scheduler is not a mutex.

### Bad

```cadence
// BAD
access(all) resource Counter {
    access(self) var n: UInt64
    access(FlowTransactionScheduler.Execute)
    fun tick() {
        if self.n % 2 == 0 { self.n = self.n + 1 } else { self.n = self.n * 2 }
    }
}
```

### Why it's bad

Don't rely on tick-ordering for correctness. The handler reads in-memory state, but cannot observe whether unrelated manual transactions in the same block have already mutated `self.n` between scheduled-tick boundaries. The scheduler provides no serialisation guarantee against non-handler transactions.

Same-slot tick order is also **non-monotonic and non-submission-ordered** — the queue insertion ordering is opaque from outside. Verified on emulator v2.17.1: four ticks submitted in order `{11, 22, 33, 44}` (scheduled IDs 7-10) executed in order `{33, 22, 44, 11}`, leaving the naive resource at `lastTick = 11`. Neither schedule-order nor id-order predicts execution order.

### Correct

```cadence
// GOOD — version counter passed through data
access(all) resource Counter {
    access(self) var n: UInt64
    access(self) var version: UInt64

    access(FlowTransactionScheduler.Execute)
    fun tick(data: AnyStruct?) {
        let expected = (data as? UInt64) ?? panic("missing version")
        pre { self.version == expected: "stale tick" }
        self.n = self.n + 1
        self.version = self.version + 1
    }
}
```

The schedule call passes the expected version through `data`. A version mismatch exits via pre-condition — observable, audit-friendly, and forces an explicit reschedule.

The version counter must be passed through `data` and compared with `==`, not `>`. A `>` check would tolerate gaps but still re-execute stale ticks if state was already advanced by another tick or manual tx.

### Detection hint

In any handler that reads-then-writes shared state, look for `expectedVersion`, `expectedNonce`, or `expectedPhase` checked against current state before mutation. Absence, combined with multiple ticks-per-resource-per-block, is the smell. Inspect every `schedule()` call site for whether the same `handlerCap` is being called more than once per Unix-second; if yes, the resource being mutated must carry a version counter passed through `data`.

---

## A4 — Assuming 100% fee refund on cancel (Medium)

`refundMultiplier` defaults to **0.5 (50%)**. The other half goes to `FlowFees` (node operator rewards), not burned, and is unrecoverable. Refund is only possible while `status == Scheduled`; after `PendingExecution` it's too late. See [scheduled-transactions.md § Refund semantics](../../cadence-lang/references/scheduled-transactions.md).

### Bad

```cadence
// BAD
let refund <- manager.cancel(id: oldId)         // returns 50% only
let newId = manager.schedule(
    handlerCap: cap, data: nil,
    timestamp: getCurrentBlock().timestamp + 60.0,
    priority: FlowTransactionScheduler.Priority.Medium,
    executionEffort: 1000,
    fees: <-refund as! @FlowToken.Vault,        // half-funded -> schedule() panics
)
```

`schedule()` asserts `fees.balance >= estimate.flowFee` and panics — surfaces as a confusing "insufficient fees" error after a clean-looking cancel.

### Correct

```cadence
// GOOD
let refund <- manager.cancel(id: oldId)
let topup <- userVault.withdraw(amount: estimatedFee - refund.balance)
refund.deposit(from: <-topup)
let newId = manager.schedule(..., fees: <-refund as! @FlowToken.Vault)
```

Better: read `refundMultiplier` from `FlowTransactionScheduler.getConfig()` at runtime — governance can change it.

### Detection hint

Grep for `cancel(` followed (in the same function) by `schedule(` reusing the returned vault. Verify the code tops up from another source rather than assuming the vault is sufficient.

---

## A5 — Privilege escalation via handler capability (Critical)

Handler resource holds an `auth(...)` capability to admin functions or unrelated state. Because `executeTransaction` is callable by anyone with an `auth(Execute) &{TransactionHandler}` cap — the scheduler, plus anyone who leaks, re-issues, or publishes that capability — wider entitlements escape.

### Bad

```cadence
// BAD — handler carries an admin entitlement
access(all) resource Handler: FlowTransactionScheduler.TransactionHandler {
    access(self) let adminCap: Capability<auth(Protocol.Admin) &Protocol.Vault>
    access(FlowTransactionScheduler.Execute)
    fun executeTransaction(id: UInt64, data: AnyStruct?) {
        let admin = self.adminCap.borrow() ?? panic("admin cap broken")
        admin.rotateOwner(to: (data as? Address) ?? panic("addr"))
    }
}
```

### Why it's bad

The handler is callable by anyone holding an `auth(Execute) &{TransactionHandler}` capability. If that capability is ever leaked, re-issued after a key compromise, or accidentally published, the attacker gets every entitlement the handler carries — with `data` as the attacker payload.

### Correct

```cadence
// GOOD — narrowest authority
access(all) resource Handler: FlowTransactionScheduler.TransactionHandler {
    access(self) let tickCap: Capability<auth(Protocol.Tick) &Protocol.Vault>
    access(FlowTransactionScheduler.Execute)
    fun executeTransaction(id: UInt64, data: AnyStruct?) {
        let vault = self.tickCap.borrow() ?? return
        vault.advanceTick()                       // single scoped operation
    }
}
```

Define a dedicated `entitlement Tick` on the target resource. The handler authorises only that one thing.

### Detection hint

For each handler, enumerate every `Capability` field. Verify each entitlement set is the minimum needed for the per-tick work — never `Admin`, never multi-purpose, and never a capability with `auth(BorrowValue, SaveValue, ...) &Account`.

---

## A6 — Handler panic = silent data loss (High)

Code assumes a panicking handler is retried, requeued, or refunded. None of that happens. The scheduler flips `status = Executed` *before* the handler runs (empirically confirmed in T01-VERIFICATION); on panic the sub-transaction reverts, no `Executed` event is emitted, the `transactions` map entry is removed, and fees are kept. The work is lost.

### Bad

```cadence
// BAD
access(FlowTransactionScheduler.Execute)
fun executeTransaction(id: UInt64, data: AnyStruct?) {
    let order = OrderBook.get(id: id) ?? panic("missing order")
    assert(order.canFill(), message: "cannot fill")
    OrderBook.fill(id: id)
}
```

### Why it's bad

The scheduler never retries. A panicking handler forfeits 100% of paid fees (no 50% refund — that's cancel-only), emits no `Executed` event, removes itself from `self.transactions` (so subsequent `cancel()` panics with `Invalid ID: <id> transaction not found`), and rolls back its own pre-panic events along with `Executed`.

### Correct

```cadence
// GOOD
access(FlowTransactionScheduler.Execute)
fun executeTransaction(id: UInt64, data: AnyStruct?) {
    let order = OrderBook.get(id: id) ?? return
    if !order.canFill() {
        OrderBook.markRetryNeeded(id: id)            // observable failure event
        MyContract.scheduleRetry(orderId: id)        // explicit reschedule
        return
    }
    OrderBook.fill(id: id)
}
```

Reserve `panic` for unrecoverable invariant violations where you genuinely want side effects rolled back.

### Detection hint

Count `panic(`, `assert(`, force-unwrap `!`, and `as!` casts in each handler body. Each is a no-retry death point. Many panics + no recoverable returns is the smell.

---

## A7 — Optimistic `Executed` flip misread (High)

Auditor or indexer reads `getStatus(id: N)`, sees `Executed`, concludes the handler ran. In reality `status = Executed` is set in `SharedScheduler.process()` *before* the handler is invoked — a panicking handler leaves `status = Executed` with **no** `Executed` event emitted.

### Bad

```cadence
// BAD — on-chain checker / off-chain indexer
let status = FlowTransactionScheduler.getStatus(id: id)
if status == FlowTransactionScheduler.Status.Executed {
    Dashboard.markSettled(id: id)                  // wrong: handler may have panicked
}
```

### Why it's bad

`status` is set optimistically. If the handler panicked: no `Executed` event was emitted, all handler-emitted events were rolled back, and the `transactions` map entry is gone (`getTransactionData(id:)` returns nil). The only reliable success signal is the `Executed` event with the matching id.

### Correct

```text
- Treat status == Executed as "the scheduler made an attempt".
- Confirm success via FlowTransactionScheduler.Executed event with same id.
- Treat (PendingExecution emitted) AND (no Executed emitted) as failure.
```

In Cadence-side audits, any contract that reads `getStatus(id:)` to decide whether a tick "ran" is buggy.

### Detection hint

Grep for `getStatus(` and `Status.Executed`. For each, verify the code path also confirms the `Executed` event, or treats status as sufficient. If sufficient: bug.

---

## A8 — Leaking COA / capability via publicly callable handler (Critical)

Handler holds a `Capability` to user funds, an EVM COA (`@EVM.CadenceOwnedAccount`), or any privileged resource — and selects its target or amount based on attacker-controlled `data: AnyStruct?`. The scheduler does not validate `data`. Any caller who invokes `executeTransaction` once has the same power as the user.

### Bad

```cadence
// BAD — destination chosen from data
access(all) resource Handler: FlowTransactionScheduler.TransactionHandler {
    access(self) let coa: Capability<auth(EVM.Call) &EVM.CadenceOwnedAccount>
    access(FlowTransactionScheduler.Execute)
    fun executeTransaction(id: UInt64, data: AnyStruct?) {
        let target = (data as? EVM.EVMAddress) ?? panic("no target")
        let amount = (data as? UInt256) ?? panic("no amount")
        let coa = self.coa.borrow() ?? panic("coa")
        coa.call(to: target, data: [], gasLimit: 100_000,
                 value: EVM.Balance(attoflow: amount))
    }
}
```

### Why it's bad

`data: AnyStruct?` is opaque and attacker-controlled. If the handler dispatches business logic off `data` — destination, amount, capability selection — then anyone who reaches the handler with a crafted schedule call can drain user funds.

### Correct

```cadence
// GOOD — destination + amount baked in at construction
access(all) resource Handler: FlowTransactionScheduler.TransactionHandler {
    access(self) let coa: Capability<auth(EVM.Call) &EVM.CadenceOwnedAccount>
    access(self) let allowedTarget: EVM.EVMAddress
    access(self) let maxAmount: UInt256

    access(FlowTransactionScheduler.Execute)
    fun executeTransaction(id: UInt64, data: AnyStruct?) {
        let coa = self.coa.borrow() ?? return
        coa.call(to: self.allowedTarget, data: [], gasLimit: 100_000,
                 value: EVM.Balance(attoflow: self.maxAmount))
    }
}
```

`data` may carry per-tick metadata, but never destination, amount, or capability selection. Revoke and recreate the handler if those need to change.

### Detection hint

For each `Capability` held by a handler, ask: "does its borrow/call/withdraw target depend on `data`?" If yes — escalation surface. Also flag any handler whose storage path has a publicly published `auth(Execute)` cap; that is almost always wrong.

---

## A9 — Race between scheduled tick and manual transaction (Medium)

A vault, balance, or queue is mutated by both a scheduled tick (recurring deposit) and a manual transaction (user withdraw). **On the Flow emulator v2.17.1, manual transactions in a block consistently execute before the scheduled-tick system collection for that block — meaning a manual withdraw racing a scheduled deposit will see the pre-deposit balance and may fail.** On testnet and mainnet the relative order may differ; **either way it is not handler-controllable.**

### Bad

```cadence
// BAD
access(all) resource Savings {
    access(self) var vault: @FlowToken.Vault
    access(FlowTransactionScheduler.Execute)
    fun depositWeekly() {
        let funds <- ExternalSource.draw()
        self.vault.deposit(from: <-funds)
    }
    access(Withdraw) fun withdraw(amount: UFix64): @FlowToken.Vault {
        return <- self.vault.withdraw(amount: amount) as! @FlowToken.Vault
    }
}
```

A user `withdraw(amount: 100)` and a scheduled `+50` deposit at the same timestamp may collide. If the withdraw runs first with only 50 available, it panics — even though the user "knew" the deposit was queued for the same block.

### Why it's bad

Block-level ordering between manual and scheduled txs is not handler-controllable, and on the emulator the manual-first ordering means the withdraw observes stale balance — guaranteed failure for the unwary user, not a 50/50 race. A defensive contract must tolerate both orderings.

### Correct

```cadence
// GOOD — defensive pre-condition; phase machine if ordering is semantic
access(Withdraw) fun withdraw(amount: UFix64): @FlowToken.Vault {
    pre {
        amount <= self.vault.balance:
            "requested ".concat(amount.toString())
            .concat(" exceeds available ").concat(self.vault.balance.toString())
    }
    return <- self.vault.withdraw(amount: amount) as! @FlowToken.Vault
}
```

For protocols where ordering matters semantically ("deposit must precede withdraw"), make the dependency explicit via a phase machine (A2) or version counter (A3). Never rely on the scheduler firing first.

### Detection hint

For each shared resource, partition entry points into "called by handler" vs "called by manual tx". For each pair touching the same field, verify correctness under either ordering. Unconditional `withdraw` or `assert(balance == X)` in a manual tx that depends on a scheduled tick's prior execution is the smell.

---

## A10 — Indexer / monitor relying on `flow blocks get` to see scheduled-tx execution (High)

Indexer, monitoring dashboard, archive-node ingester, or audit tool walks blocks via `flow blocks get <n>` (or the equivalent gRPC `GetBlockByHeight` collections listing) and assumes the user-visible collections list contains every transaction that ran in that block — including scheduled-tick callbacks. It does not. Scheduled-tick callbacks fire in a **system collection** that is NOT surfaced in `flow blocks get <n>`'s collections list.

### Bad

```text
// BAD — indexer logic
for block in blocks {
    for collection in block.collections {       // user-tx collections only
        for tx in collection.transactions {
            indexer.record(tx)                  // scheduled ticks: silently missed
        }
    }
}
```

### Why it's bad

Verified on emulator v2.17.1: block 153 reported `Total Collections = 1` containing only the manual user tx, while the scheduled tick that ran in that same block executed in a separate system collection that does not appear in the `flow blocks get` output. An indexer or monitor that walks block collections will silently miss every scheduled-tick callback. Downstream tools that count "txs per block", reconcile balances by replaying collection contents, or alert on "no activity in block N" will all be wrong. Auditors using block-walking tooling to verify "did this tick actually run" will get a false negative.

### Correct

```text
- Treat block.collections as user-tx collections only.
- For scheduled-tx execution, subscribe to FlowTransactionScheduler.Executed
  events (id matching the scheduled id) rather than walking block collections.
- For success/failure signal, follow A7: Executed event present == success;
  PendingExecution without subsequent Executed == failure.
- If you need a per-block count of scheduled-tick executions, aggregate
  Executed events by their emitting block height — not by collection walk.
```

### Detection hint

In indexer / dashboard / archive code, grep for `block.collections`, `GetBlockByHeight`, `flow blocks get`, or equivalent collection-walking patterns. For each, verify the consumer is NOT trying to enumerate scheduled-tick executions from that surface. If it is — bug. The `Executed` event stream is the only authoritative source.

---

## Audit checklist for code using `FlowTransactionScheduler`

For any contract or transaction that schedules, handles, or cancels scheduled transactions, the auditor must answer **yes** to every question — or document a deliberate exception.

### Handler design
- [ ] Does the handler hold *only* the narrowest capabilities needed for one tick (no admin caps, no `auth(...) &Account`)?
- [ ] Does `executeTransaction` use `return` (not `panic`) for all recoverable conditions?
- [ ] Are all `panic`, `assert`, force-unwrap, and `as!` paths genuinely unrecoverable invariants?
- [ ] Is `data: AnyStruct?` treated as untrusted input and validated before use?
- [ ] Is `data` informational only — never used to select capabilities, destinations, or amounts?

### Recurrence and termination
- [ ] If the handler reschedules itself, is there a kill switch checked before each reschedule?
- [ ] Is there a hard iteration cap or absolute deadline enforced by the handler?
- [ ] Are next-tick fees drawn from an explicit reserve, with reserve exhaustion handled (not panicked)?

### State machine and concurrency
- [ ] For every resource touched in `executeTransaction`, is there a phase enum or version counter guarding mutations?
- [ ] Are manual-tx entry points to the same resource protected by the same phase/version checks?
- [ ] Are pre-conditions robust under **both** orderings of scheduled-vs-manual transactions in one block, including the emulator's manual-first ordering and any reordering testnet/mainnet block builders may apply?

### Fees and cancellation
- [ ] Does any path that calls `cancel()` and re-schedules account for the 50% refund (not assume 100%)?
- [ ] Is the refund multiplier read from `getConfig()` when business logic depends on it?
- [ ] Is `cancel()` only called when `status == Scheduled`, and is the post-execution `Invalid ID` panic handled gracefully?

### Capability hygiene
- [ ] Is the handler's `auth(FlowTransactionScheduler.Execute)` capability stored privately, never published?
- [ ] Are storage capability controllers tracked so the capability can be revoked if the handler is replaced?
- [ ] Is the un-entitled `&{TransactionHandler}` capability the only publicly published cap on the handler's storage path?

### Observability
- [ ] Does indexer/dashboard logic confirm success via the `Executed` event, not via `getStatus(id:)`?
- [ ] Are handler-emitted events (which may roll back on panic) paired with `Executed` before being treated as authoritative?
- [ ] Are failure signals (`PendingExecution` without subsequent `Executed`) observable to operators?

### Scheduling boundary conditions
- [ ] Does every `schedule()` call pass a timestamp **strictly greater than** `getCurrentBlock().timestamp`? `timestamp == now` is rejected with an `"Invalid timestamp: X is in the past, current timestamp: X"` panic (verified on emulator v2.17.1).

### Configuration drift
- [ ] Does business logic that quotes "9,999 CU per tx", "50% refund", or "17,500 CU per slot" read those from `FlowTransactionScheduler.getConfig()` rather than hard-coding them?

## See also

- [scheduled-transactions.md](../../cadence-lang/references/scheduled-transactions.md) — canonical API reference (slot model, fee formula, failure handling, optimistic-Executed flip)
- `cadence-lang/references/cu-optimization.md` — sizing `executionEffort`
- `cadence-audit/references/audit-checklist.md` — general Cadence audit checklist (apply in addition to this file)
