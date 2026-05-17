# Time Mocking for Scheduled Transactions in Tests

Testing scheduled transactions is non-obvious because the trigger is *time*, not a transaction submission. There is no `Test.runScheduledTransactions()` method to fire callbacks on demand; instead, the Cadence Testing Framework's `Test.moveTime(by:)` is wired all the way through to the same `SharedScheduler.process()` path the FVM uses on a live network. When you advance the wall clock past a scheduled timestamp, the framework runs the callback for you as part of the next block the runtime produces — including all the failure-mode behaviour described in [`scheduled-transactions.md`](./scheduled-transactions.md) (optimistic `Executed` flip, no retries, fees fully consumed on panic).

This reference is specific to scheduler callbacks. The general primitives (`Test.moveTime`, `Test.commitBlock`, and the `getCurrentBlock()` staleness gotcha) are covered in [`blockchain-emulation.md`](./blockchain-emulation.md); cross-reference it for anything not specific to the scheduler.

## Trigger semantics: `moveTime` runs the callback

`Test.moveTime(by:)` advances the in-process blockchain's wall clock by a `Fix64` number of seconds. When the new time passes a scheduled timestamp, the runtime processes the scheduler queue and invokes `executeTransaction` on each ready handler. **No extra Test.* method is required** — the same `moveTime` you use for vesting or expiry tests also fires scheduled callbacks.

```cadence
// Schedule "now + 2s".
let scheduleResult = Test.executeTransaction(scheduleTx)
Test.expect(scheduleResult, Test.beSucceeded())

// Advance past the scheduled timestamp. This is what fires the callback.
Test.moveTime(by: 3.0 as Fix64)

// Read the post-callback state via a script (see staleness gotcha below).
let count = readCounter()
Test.assertEqual(1 as UInt64, count)
```

The empirically verified shape from the upstream
[`onflow/scheduledtransactions-scaffold` tests](https://github.com/onflow/scheduledtransactions-scaffold/blob/main/cadence/tests/CounterTransactionHandler_test.cdc):
`schedule` -> `Test.moveTime(by: delay + 1.0)` -> assert state via a script. `Test.commitBlock()` is not called between `moveTime` and the assertion in the scaffold; the runtime produces the blocks needed to drain the scheduler queue internally.

You should still call `Test.commitBlock()` when a contract reads `getCurrentBlock().height` (the scheduler keys on Unix-second timestamps; block height is not advanced by `moveTime` alone). See "Moving time vs. committing blocks" below.

## Reading state after the callback: always via a script

Direct calls to `getCurrentBlock().timestamp` and `getCurrentBlock().height` from inside a test function return **stale** values, which is critical when the test is asserting on a timestamp the handler observed. The fix — always read current time/height through `Test.executeScript` — is documented once in [`blockchain-emulation.md`](./blockchain-emulation.md#time-manipulation); the same rule applies to *every* contract field a scheduled callback mutated.

❌ Reading state directly from the test runtime returns pre-callback values:

```cadence
Test.moveTime(by: 3.0)
// WRONG: Counter.count read this way may be stale.
Test.assertEqual(1 as UInt64, Counter.count)
```

✅ Read through a script:

```cadence
access(all) fun readCount(): UInt64 {
    let result = Test.executeScript(
        "import \"Counter\"\naccess(all) fun main(): UInt64 { return Counter.count }",
        []
    )
    Test.expect(result, Test.beSucceeded())
    return result.returnValue! as! UInt64
}
```

The script runs against the post-callback blockchain state and returns a fresh value. The same rule applies to scheduler-facing reads like `FlowTransactionScheduler.getStatus(id:)`.

## Happy path: schedule, advance time, assert

The minimal end-to-end test for a Counter-incrementing handler:

```cadence
import Test
import "FlowTransactionScheduler"

// The Flow service account holds the bootstrap FLOW balance in tests,
// which makes it the path of least resistance for fee withdrawals.
access(all) let signer = Test.serviceAccount()

access(all) fun setup() {
    var err = Test.deployContract(
        name: "Counter",
        path: "../contracts/Counter.cdc",
        arguments: []
    )
    Test.expect(err, Test.beNil())
}

access(all) fun testHandlerIncrementsCounterOnTick() {
    // Arrange: install the handler (saves the resource, issues both caps).
    let installResult = Test.executeTransaction(Test.Transaction(
        code: Test.readFile("../transactions/install_counter_handler.cdc"),
        authorizers: [signer.address],
        signers: [signer],
        arguments: []
    ))
    Test.expect(installResult, Test.beSucceeded())

    // Arrange: schedule a Medium-priority callback 2s from now,
    //          effort 1000 CU, fee 0.001 FLOW (covers measured 0.00006).
    let scheduleResult = Test.executeTransaction(Test.Transaction(
        code: Test.readFile("../transactions/schedule_simple.cdc"),
        authorizers: [signer.address],
        signers: [signer],
        arguments: [
            2.0 as UFix64,    // delaySeconds
            1 as UInt8,       // priority: Medium
            1000 as UInt64,   // executionEffort
            0.001 as UFix64   // feeAmount
        ]
    ))
    Test.expect(scheduleResult, Test.beSucceeded())

    // Confirm the Scheduled event fired.
    let scheduledBefore = Test.eventsOfType(Type<FlowTransactionScheduler.Scheduled>()).length

    // Act: advance the wall clock past the scheduled timestamp.
    //      moveTime(2.0) would put us *at* the slot; +1.0 of headroom
    //      ensures the slot has been processed.
    Test.moveTime(by: 3.0 as Fix64)

    // Assert: handler ran exactly once.
    Test.assertEqual(1 as UInt64, readCount())

    // Assert: the lifecycle events fired in order.
    let scheduledAfter = Test.eventsOfType(Type<FlowTransactionScheduler.Scheduled>()).length
    let executed = Test.eventsOfType(Type<FlowTransactionScheduler.Executed>())
    Test.assertEqual(scheduledBefore, scheduledAfter)        // no new Scheduled
    Test.expect(executed, Test.haveElementCount(1))           // one Executed
}
```

The handler resource implements `FlowTransactionScheduler.TransactionHandler` and is installed by `install_counter_handler.cdc`; the transaction itself is straight from the scheduled-transactions reference. The test asserts on both state (the counter incremented) and events (`Executed` fired exactly once) — both observables, both worth checking.

## Cancellation

A cancellation test schedules a callback, cancels it before the timestamp lands, and asserts the `Canceled` event fired with the 50% refund split and that the handler does *not* run after time advances past the original timestamp.

```cadence
access(all) fun testCancelBeforeExecutionPreventsCallback() {
    installHandler()
    scheduleAt(delay: 10.0, fee: 0.005)        // helper wrapping schedule_simple.cdc

    let cancelResult = Test.executeTransaction(Test.Transaction(
        code: Test.readFile("../transactions/cancel_saved.cdc"),
        authorizers: [signer.address], signers: [signer], arguments: []
    ))
    Test.expect(cancelResult, Test.beSucceeded())

    let canceled = Test.eventsOfType(Type<FlowTransactionScheduler.Canceled>())
    Test.expect(canceled, Test.haveElementCount(1))
    let evt = canceled[0] as! FlowTransactionScheduler.Canceled
    Test.assertEqual(0.00250 as UFix64, evt.feesReturned)   // 50% of 0.005
    Test.assertEqual(0.00250 as UFix64, evt.feesDeducted)

    // Advance past the would-be slot — handler must NOT run.
    Test.moveTime(by: 15.0 as Fix64)
    Test.assertEqual(0 as UInt64, readCount())
    Test.expect(
        Test.eventsOfType(Type<FlowTransactionScheduler.Executed>()),
        Test.haveElementCount(0)
    )
}
```

Cancellation *after* the callback has fired panics with `Invalid ID: <id> transaction not found` (not a status-mismatch). Test that path by advancing time first, then attempting the cancel and asserting `Test.beFailed()` with a substring check on the error message.

## Expiry / "missed window" semantics

Scheduled transactions do not expire as a first-class concept — once the slot's timestamp passes, the callback runs (or, for Low priority, may be slid forward by congestion). To test what an application does when its *business* deadline is missed: schedule the callback, advance time past the application's deadline (but before the scheduler timestamp), mutate some app-level state that the handler treats as "deadline passed," then advance the rest of the way. The handler must take its no-op branch with an early `return` — never `panic`, which consumes the full fee and emits no `Executed` event (see [`scheduled-transactions.md`](./scheduled-transactions.md) failure-handling). Assert that `Executed` fired (handler ran) *and* the side effect did not happen (`readCount() == 0`).

## Multi-tick / fan-out scenarios

Two flavors: a single self-rescheduling chain (3+ ticks in series) and multiple independent schedules that fan out across a slot.

### Self-rescheduling chain

The `SelfTick` contract from the verifier's workdir increments a counter and re-schedules itself for `now + 3s`, stopping after `maxTicks`. The test plays the entire chain by advancing time enough to cover every link:

```cadence
access(all) fun testSelfReschedulingChainStopsAfterMaxTicks() {
    installSelfTick(maxTicks: 3)
    seedFeeReserve(amount: 0.01 as UFix64)

    // Kick off the first tick at t+3.
    kickSelfTick()

    // Each tick re-schedules at +3s; three ticks total = +9s.
    // Add headroom so the third tick's slot has surely been processed.
    Test.moveTime(by: 10.0 as Fix64)

    // Inspect via script.
    Test.assertEqual(3 as UInt64, readTickCount())

    // After maxTicks, the handler stops rescheduling — no Scheduled events
    // beyond the chain's natural length.
    let scheduled = Test.eventsOfType(Type<FlowTransactionScheduler.Scheduled>())
    Test.expect(scheduled, Test.haveElementCount(3))   // kick + 2 reschedules
}
```

For a chain where you want to assert intermediate state (e.g. counter is exactly 2 after the second tick), step time forward one link at a time and read the script after each step:

```cadence
kickSelfTick()
Test.moveTime(by: 4.0 as Fix64)   // covers t=3 first tick
Test.assertEqual(1 as UInt64, readTickCount())

Test.moveTime(by: 3.0 as Fix64)   // covers t=6 second tick
Test.assertEqual(2 as UInt64, readTickCount())

Test.moveTime(by: 3.0 as Fix64)   // covers t=9 third tick
Test.assertEqual(3 as UInt64, readTickCount())
```

### Fan-out within one slot

Schedule three Medium-priority callbacks for the same future timestamp, advance past it, and assert all three fired in a single processing pass. Useful for testing that handlers tolerate sibling traffic in the same slot and for asserting on slot-saturation behaviour.

```cadence
access(all) fun testThreeMediumCallbacksFireInSameSlot() {
    installHandler()
    let targetDelay = 2.0 as UFix64
    for _ in [1, 2, 3] {
        Test.expect(
            scheduleAt(delay: targetDelay, effort: 1000, fee: 0.001),
            Test.beSucceeded()
        )
    }
    let scheduledBefore = Test.eventsOfType(Type<FlowTransactionScheduler.Scheduled>()).length
    Test.assertEqual(3, scheduledBefore)

    Test.moveTime(by: 3.0 as Fix64)
    Test.assertEqual(3 as UInt64, readCount())
    Test.expect(
        Test.eventsOfType(Type<FlowTransactionScheduler.Executed>()),
        Test.haveElementCount(3)
    )
}
```

To exercise the "Medium shifts forward when slot is full" behaviour (see [`scheduled-transactions.md`](./scheduled-transactions.md) slot model), schedule two Medium callbacks each at 4000 CU effort for the same timestamp. The second one will land at `requested + 1s` (verified empirically). Assert by reading the `Scheduled` event's `timestamp` field on each:

```cadence
let events = Test.eventsOfType(Type<FlowTransactionScheduler.Scheduled>())
let a = events[0] as! FlowTransactionScheduler.Scheduled
let b = events[1] as! FlowTransactionScheduler.Scheduled
Test.assertEqual(a.timestamp + 1.0, b.timestamp)
```

## Fee accounting in tests

Scheduled transactions cost FLOW. In a test, the signer of the *scheduling* transaction is what pays — the scheduler contract account holds the fees in escrow until execution or cancellation. Two practical patterns:

✅ **Use `Test.serviceAccount()` as the signer.** The framework's service account is bootstrapped with a positive FLOW balance, so withdrawing from `/storage/flowTokenVault` works out of the box. This is the path the upstream `scheduledtransactions-scaffold` test takes and is the simplest setup for most callback tests.

✅ **Fund a fresh account before scheduling.** When the test needs a distinct signer (for access-control or multi-user scenarios), mint FLOW into the new account first. The account allocated by `Test.createAccount()` has *no* FLOW vault until something deposits to it; trying to schedule before that fails with "missing FlowToken vault" or zero balance.

```cadence
access(all) let user = Test.createAccount()

access(all) fun setup() {
    // ... deploy contracts ...
    // Mint 1.0 FLOW from the service account into the user account.
    let mintResult = Test.executeTransaction(Test.Transaction(
        code: Test.readFile("../transactions/mint_flow.cdc"),
        authorizers: [Test.serviceAccount().address],
        signers: [Test.serviceAccount()],
        arguments: [user.address, 1.0 as UFix64]
    ))
    Test.expect(mintResult, Test.beSucceeded())
}
```

Fee sizing (from `T01` Test 9 measurements): Low = 0.00003, Medium = 0.00006, High = 0.00011 FLOW for any effort under 9,999 CU with no data payload. Pass a fee that comfortably covers the priority — 0.001 FLOW is generous for any single Medium callback and avoids needing to call `estimate()` from the test.

Refunds on cancellation arrive in the cancelling transaction as a `@FlowToken.Vault` (50% of fee). For a test that asserts the refund landed back in the signer's vault, run a balance script before and after the cancel transaction:

```cadence
let balBefore = readFlowBalance(of: signer.address)
let cancelResult = Test.executeTransaction(cancelTx)
Test.expect(cancelResult, Test.beSucceeded())
let balAfter = readFlowBalance(of: signer.address)
Test.assertEqual(balBefore + 0.00250 as UFix64, balAfter)
```

## Moving time vs. committing blocks

The two operations look interchangeable but advance different clocks:

| Operation | Advances `timestamp` | Advances `height` | Triggers scheduler |
|---|---|---|---|
| `Test.moveTime(by: x)` | yes (by x seconds) | no | yes, when crossing a slot |
| `Test.commitBlock()` | no | yes (by 1) | no |
| Submitting a transaction (`Test.executeTransaction`) | yes (advances a small amount) | yes (commits one block) | yes, if the tx's block crosses a slot |

The scheduler's internal sorted map keys on Unix seconds, so **only `moveTime` makes callbacks fire**. `commitBlock` on its own moves the block height but not the wall clock, so a contract that reads `getCurrentBlock().timestamp` sees the same value across many committed blocks — and the scheduler will sit on its queue indefinitely until time advances.

❌ Anti-pattern — committing blocks expecting callbacks to fire:

```cadence
// WRONG: scheduled tx will never fire because time hasn't moved.
Test.executeTransaction(scheduleTx)
for _ in [1, 2, 3, 4, 5] { Test.commitBlock() }
Test.assertEqual(1 as UInt64, readCount())   // fails — handler never ran
```

✅ Correct — advance time:

```cadence
Test.executeTransaction(scheduleTx)
Test.moveTime(by: 3.0 as Fix64)
Test.assertEqual(1 as UInt64, readCount())
```

If the contract reads both `getCurrentBlock().timestamp` and `getCurrentBlock().height`, advance time first and then commit a block:

```cadence
Test.moveTime(by: 3.0 as Fix64)
Test.commitBlock()
```

## Test isolation around scheduled callbacks

The snapshot-and-reset pattern from [`patterns.md`](./patterns.md) composes cleanly with scheduled tx tests *as long as* the reset height is taken after every account that will ever sign a schedule was created. Accounts that don't exist at the reset height cannot sign transactions; calls fail with "account public key not found." The safe shape:

```cadence
access(all) let signer = Test.serviceAccount()   // created at framework init
access(all) var setupHeight: UInt64 = 0

access(all) fun setup() {
    // Deploy contracts.
    // Install the handler.
    // Snapshot AFTER everything that future tests need is in place.
    setupHeight = readBlockHeight()              // via script — see staleness gotcha
}

access(all) fun beforeEach() {
    Test.reset(to: setupHeight)
}
```

Each test then re-schedules whatever it needs from a clean baseline. **Do not** snapshot before `install_handler.cdc` runs and expect each test to install its own handler — the reset rewinds the install, and the new install runs into "path already stores an object" or capability-collision issues on the second test.

## Common pitfalls

- **Asserting state directly without a script.** Reading `Counter.count` or `getCurrentBlock().timestamp` from inside a test function after `Test.moveTime` returns stale values. Always go through `Test.executeScript` — see [`blockchain-emulation.md`](./blockchain-emulation.md#time-manipulation).
- **Confusing `moveTime` with `commitBlock`.** Only `moveTime` triggers scheduler callbacks; `commitBlock` advances height but not time. Schedulers key on Unix seconds.
- **Insufficient headroom on `moveTime`.** `Test.moveTime(by: delay)` puts the clock exactly on the slot boundary; a small amount of slop (`delay + 1.0`) is the safe default to guarantee the slot has been processed before the assertion runs.
- **Using `Test.createAccount()` as the signer without funding.** Fresh test accounts have no FLOW vault. Fund them from `Test.serviceAccount()` before scheduling, or use the service account directly.
- **Snapshot before account/handler setup.** `Test.reset(to: heightBeforeSetup)` invalidates any account or capability created after that height. Always snapshot *after* the installs you want to persist across tests.
- **Asserting `getStatus(id:) == Executed` to confirm success.** Status flips to `Executed` *before* the handler runs (optimistic flip — see [`T01-scheduled-transactions.md`](./scheduled-transactions.md) failure-handling). A handler that panicked still shows `Executed`. The reliable success signal is the presence of an `Executed` *event* with the matching id.
- **Expecting `Test.executeScript`-style methods to fire callbacks.** Scripts are read-only and never affect block production. Only `moveTime` (and transactions that themselves cross a slot boundary) trigger the scheduler.
- **Forgetting that events accumulate across the file.** `Test.eventsOfType(...)` returns *every* event emitted since the file started, including from prior tests and from `setup()`. Snapshot the length before each act-and-assert pair and subtract, or call `Test.reset(to:)` in `beforeEach` to clear the slate.
