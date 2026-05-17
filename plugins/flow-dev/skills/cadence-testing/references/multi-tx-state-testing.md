# Testing Multi-Transaction State-Machine Resources

Resources that walk a state graph across many transactions — escrows, vesting contracts, async intent flows, auction lockups — are non-obvious to test for three connected reasons. First, the test runner sees a *stale* view of contract fields when read directly from inside a test function; only scripts get the post-transaction view. Second, a state machine that takes five transactions to drive end-to-end is only really tested when one test walks the full transition graph, not when five separate tests each cover one edge. Third, the failure paths (revert mid-flight, race between two claimants) are where the contract earns its keep, and they only surface in tests that explicitly model the interleaving.

This reference is the testing complement to two cadence-lang references:
- [resource-state-machines.md](../../cadence-lang/references/resource-state-machines.md) — the state-machine pattern itself (T03).
- [multi-tx-escrow.md](../../cadence-lang/references/multi-tx-escrow.md) — the canonical escrow shape this file's worked example matches (T07).

General mechanics (Test API, accounts, deploys, `moveTime`, `commitBlock`, the staleness gotcha) live in [setup-and-basics.md](./setup-and-basics.md) and [blockchain-emulation.md](./blockchain-emulation.md); pattern-level conventions (Arrange/Act/Assert, snapshot isolation, event-as-API) live in [patterns.md](./patterns.md). This file does not re-derive those — it shows how they compose when the contract under test is a state machine driven by N transactions.

## The Worked Example: `IntentEscrow`

Throughout this reference, the contract under test is `IntentEscrow` — a resource that holds a depositor's vault, arms when both parties confirm, becomes claimable once a release condition is satisfied, and closes on either claim or timeout-refund. The state graph:

```
Funding ─ deposit ─▶ Funding ─ arm ─▶ Active ─ release ─▶ Claimable ─ claim ─▶ Closed
                                       │
                                       └─ timeoutRefund (after expiry) ─▶ Refunded
```

The contract exposes `getStatus(id: UInt64): UInt8` returning the enum case (0 = Funding, 1 = Active, 2 = Claimable, 3 = Closed, 4 = Refunded) and `getBalance(id: UInt64): UFix64`. Both are read-only and intended to be called from scripts.

## Read state via scripts, not direct field access

The single most common bug in multi-tx state tests is asserting on a contract field directly from the test function and getting a stale value. The framework's import-and-call path resolves against an internal snapshot that does **not** advance with each `executeTransaction` call inside a test body. Scripts execute against the live post-transaction state and always return fresh values.

This is the same staleness rule documented for `getCurrentBlock()` in [blockchain-emulation.md § Time manipulation](./blockchain-emulation.md#time-manipulation), but it applies to **every** field a transaction mutated, not just block-level reads. Status enums, balances, expiry timestamps, dictionary lengths — all of them must be read through a script after the first mutating transaction in the test.

Define one helper per observable. The helpers stay short and the test bodies read like prose:

```cadence
import Test
import "IntentEscrow"

access(all) fun readStatus(id: UInt64): UInt8 {
    let result = Test.executeScript(
        "import \"IntentEscrow\"\n"
            .concat("access(all) fun main(id: UInt64): UInt8 {\n")
            .concat("    return IntentEscrow.getStatus(id: id)\n")
            .concat("}"),
        [id]
    )
    Test.expect(result, Test.beSucceeded())
    return result.returnValue! as! UInt8
}

access(all) fun readBalance(id: UInt64): UFix64 {
    let result = Test.executeScript(
        "import \"IntentEscrow\"\n"
            .concat("access(all) fun main(id: UInt64): UFix64 {\n")
            .concat("    return IntentEscrow.getBalance(id: id)\n")
            .concat("}"),
        [id]
    )
    Test.expect(result, Test.beSucceeded())
    return result.returnValue! as! UFix64
}
```

The anti-pattern this exists to replace:

```cadence
// WRONG — reads pre-transaction view of IntentEscrow.statuses
Test.executeTransaction(armTx)
Test.assertEqual(1 as UInt8, IntentEscrow.statuses[id]!)   // stale, often == 0
```

Replace with:

```cadence
// CORRECT — script runs against post-transaction state
Test.executeTransaction(armTx)
Test.assertEqual(1 as UInt8, readStatus(id: id))
```

The cost is one extra `executeScript` per assertion. The benefit is that every observation actually reflects what the chain would show an off-chain reader at the same point.

## Walk the full state graph in a single test

A state machine is only meaningfully tested when one test drives it from initial state to terminal state. Sharding "Funding → Active" into one test and "Active → Claimable" into another loses the most valuable property of the suite: that the *composition* of transitions still arrives at the expected terminal state. Each test in such a sharded suite passes individually while the contract is silently broken at the seam.

The canonical happy-path test for `IntentEscrow` walks Funding → Active → Claimable → Closed inside a single function. Between each transition transaction, a script reads back the state and the test asserts on it.

```cadence
import Test
import "IntentEscrow"
import "TestToken"

access(all) let depositor = Test.serviceAccount()
access(all) let beneficiary = Test.createAccount()
access(all) let oracle = Test.createAccount()

access(all) var setupHeight: UInt64 = 0

access(all) fun setup() {
    var err = Test.deployContract(
        name: "TestToken",
        path: "../contracts/TestToken.cdc",
        arguments: []
    )
    Test.expect(err, Test.beNil())

    err = Test.deployContract(
        name: "IntentEscrow",
        path: "../contracts/IntentEscrow.cdc",
        arguments: []
    )
    Test.expect(err, Test.beNil())

    // Fund all participants and snapshot AFTER everything they need exists.
    fundAccount(beneficiary.address, amount: 10.0)
    setupHeight = readBlockHeight()
}

access(all) fun beforeEach() {
    Test.reset(to: setupHeight)
}

access(all) fun testEscrowFullHappyPath() {
    let id: UInt64 = 1
    let amount: UFix64 = 100.0

    // Tx 1: deposit — state becomes Funding (0)
    let depositResult = Test.executeTransaction(Test.Transaction(
        code: Test.readFile("../transactions/deposit.cdc"),
        authorizers: [depositor.address],
        signers: [depositor],
        arguments: [id, amount, beneficiary.address, oracle.address]
    ))
    Test.expect(depositResult, Test.beSucceeded())
    Test.assertEqual(0 as UInt8, readStatus(id: id))
    Test.assertEqual(amount, readBalance(id: id))
```

```cadence
    // Tx 2: arm — state becomes Active (1)
    let armResult = Test.executeTransaction(Test.Transaction(
        code: Test.readFile("../transactions/arm.cdc"),
        authorizers: [beneficiary.address],
        signers: [beneficiary],
        arguments: [id]
    ))
    Test.expect(armResult, Test.beSucceeded())
    Test.assertEqual(1 as UInt8, readStatus(id: id))

    // Tx 3: release (oracle signals condition satisfied) — Active -> Claimable (2)
    let releaseResult = Test.executeTransaction(Test.Transaction(
        code: Test.readFile("../transactions/release.cdc"),
        authorizers: [oracle.address],
        signers: [oracle],
        arguments: [id]
    ))
    Test.expect(releaseResult, Test.beSucceeded())
    Test.assertEqual(2 as UInt8, readStatus(id: id))

    // Tx 4: claim — Claimable -> Closed (3), balance drains
    let claimResult = Test.executeTransaction(Test.Transaction(
        code: Test.readFile("../transactions/claim.cdc"),
        authorizers: [beneficiary.address],
        signers: [beneficiary],
        arguments: [id]
    ))
    Test.expect(claimResult, Test.beSucceeded())
    Test.assertEqual(3 as UInt8, readStatus(id: id))
    Test.assertEqual(0.0 as UFix64, readBalance(id: id))

    // Final invariant: one Claimed event was emitted for this id.
    let claimed = Test.eventsOfType(Type<IntentEscrow.Claimed>())
    Test.expect(claimed, Test.haveElementCount(1))
    let evt = claimed[0] as! IntentEscrow.Claimed
    Test.assertEqual(id, evt.id)
    Test.assertEqual(amount, evt.amount)
}
```

Two observations on this shape. First, every transition transaction is paired with a script-based assertion on the new state, not a direct field read. Second, the final assertion is on the terminal event — the test would silently pass if `claim` mutated state without emitting `Claimed`, which is exactly the kind of regression an off-chain indexer would notice in production. Pairing the state assertion with the event assertion catches both halves of the contract's promise.

## Failure path: mid-flight revert leaves state unchanged

A state machine that survives an unauthorised attempt mid-walk must reject the bad call cleanly and leave both the state enum and the stored balance untouched. The test:

```cadence
access(all) fun testWrongRecipientClaimRevertsAndPreservesState() {
    let id: UInt64 = 2
    let amount: UFix64 = 50.0
    let stranger = Test.createAccount()
    fundAccount(stranger.address, amount: 1.0)

    // Tx 1: deposit
    Test.expect(
        Test.executeTransaction(makeDeposit(id: id, amount: amount)),
        Test.beSucceeded()
    )

    // Tx 2: arm
    Test.expect(
        Test.executeTransaction(makeArm(id: id, signer: beneficiary)),
        Test.beSucceeded()
    )
    Test.assertEqual(1 as UInt8, readStatus(id: id))
    Test.assertEqual(amount, readBalance(id: id))

    // Tx 3: stranger tries to claim — must revert on the recipient check.
    let badClaim = Test.executeTransaction(Test.Transaction(
        code: Test.readFile("../transactions/claim.cdc"),
        authorizers: [stranger.address],
        signers: [stranger],
        arguments: [id]
    ))
    Test.expect(badClaim, Test.beFailed())
    Test.assert(
        badClaim.error!.message.contains("only beneficiary may claim"),
        message: "unexpected error: ".concat(badClaim.error!.message)
    )

    // Tx 4: assert state and balance survived the revert untouched.
    Test.assertEqual(1 as UInt8, readStatus(id: id))     // still Active
    Test.assertEqual(amount, readBalance(id: id))         // unchanged

    // Tx 5: timeout + refund — depositor recovers funds after expiry.
    Test.moveTime(by: 86401.0 as Fix64)                   // > 1 day expiry
    let refundResult = Test.executeTransaction(Test.Transaction(
        code: Test.readFile("../transactions/timeout_refund.cdc"),
        authorizers: [depositor.address],
        signers: [depositor],
        arguments: [id]
    ))
    Test.expect(refundResult, Test.beSucceeded())
    Test.assertEqual(4 as UInt8, readStatus(id: id))     // Refunded
    Test.assertEqual(0.0 as UFix64, readBalance(id: id))
}
```

Three things make this test load-bearing. The substring on `claim.error!.message` is short enough to survive a re-wording of the error message — see [patterns.md § Testing access control](./patterns.md). The post-revert assertions (`status == Active`, `balance == amount`) prove that Cadence's transaction-atomic semantics actually held; a half-applied revert is the worst kind of bug because the next test starts from a corrupt fixture. The refund tail extends the same test through the alternate terminal state, which is cheap because the prior steps are already in place.

## Race condition: two actors against one resource

When two off-chain actors race to claim the same `IntentEscrow`, the contract's pre-condition (`pre { self.status == Status.Claimable }`) must let the first claim through and reject the second with a clear error. Both outcomes are interesting and the test asserts on both.

```cadence
access(all) fun testRaceBetweenTwoBeneficiariesFirstWins() {
    let id: UInt64 = 3
    let amount: UFix64 = 25.0

    // Walk to Claimable.
    Test.expect(Test.executeTransaction(makeDeposit(id: id, amount: amount)), Test.beSucceeded())
    Test.expect(Test.executeTransaction(makeArm(id: id, signer: beneficiary)), Test.beSucceeded())
    Test.expect(Test.executeTransaction(makeRelease(id: id, signer: oracle)), Test.beSucceeded())
    Test.assertEqual(2 as UInt8, readStatus(id: id))

    // Construct two claims signed by the SAME beneficiary back-to-back.
    // The first succeeds, the second fails on the state-machine pre-condition.
    let claim1 = Test.executeTransaction(Test.Transaction(
        code: Test.readFile("../transactions/claim.cdc"),
        authorizers: [beneficiary.address],
        signers: [beneficiary],
        arguments: [id]
    ))
    let claim2 = Test.executeTransaction(Test.Transaction(
        code: Test.readFile("../transactions/claim.cdc"),
        authorizers: [beneficiary.address],
        signers: [beneficiary],
        arguments: [id]
    ))

    // First wins.
    Test.expect(claim1, Test.beSucceeded())
    Test.assertEqual(3 as UInt8, readStatus(id: id))
    Test.assertEqual(0.0 as UFix64, readBalance(id: id))

    // Second sees the new state (Closed) and reverts on the pre-condition.
    Test.expect(claim2, Test.beFailed())
    Test.assert(
        claim2.error!.message.contains("status != Claimable"),
        message: "unexpected error: ".concat(claim2.error!.message)
    )

    // One Claimed event, not two — the second never made it past the pre.
    let claimed = Test.eventsOfType(Type<IntentEscrow.Claimed>())
    Test.expect(claimed, Test.haveElementCount(1))
}
```

For a race between two **different** actors (both holding the beneficiary capability, e.g. via a shared key), replace `beneficiary` with the second actor in `claim2` and the test reads identically. The point of the race test is not to prove there's only one winner under one actor — it's to prove the state-machine pre-condition is the gate that prevents double-claim, regardless of who tries.

`Test.executeTransactions([claim1Tx, claim2Tx])` is a useful variant when the test cares that both attempts land in the same block. The framework runs them in order, the first commits its state change, and the second's pre-condition sees the post-claim status — same outcome, with the extra invariant that the block boundary is shared.

## Snapshot-based isolation between phases

`Test.reset(to: setupHeight)` in `beforeEach` keeps each test starting from a clean post-deploy state. The pattern is documented in detail in [patterns.md § Test isolation](./patterns.md) and [blockchain-emulation.md § State reset](./blockchain-emulation.md#state-reset-snapshot-isolation); the multi-tx specialisation here is that **every account that any test signs a transaction with must be created before the snapshot height is recorded**.

```cadence
access(all) let depositor = Test.serviceAccount()         // exists at init
access(all) let beneficiary = Test.createAccount()        // file-load time
access(all) let oracle = Test.createAccount()             // file-load time

access(all) var setupHeight: UInt64 = 0

access(all) fun setup() {
    deployAll()                                            // contracts deploy
    fundAccount(beneficiary.address, amount: 10.0)         // beneficiary gets a vault
    setupHeight = readBlockHeight()                        // AFTER everything ready
}

access(all) fun beforeEach() {
    Test.reset(to: setupHeight)
}
```

Reading the snapshot height through a script is required — `getCurrentBlock().height` from inside `setup()` returns the same stale value as for `timestamp`. A short helper:

```cadence
access(all) fun readBlockHeight(): UInt64 {
    let result = Test.executeScript(
        "access(all) fun main(): UInt64 { return getCurrentBlock().height }",
        []
    )
    Test.expect(result, Test.beSucceeded())
    return result.returnValue! as! UInt64
}
```

Two failure modes to avoid. First, snapshotting *before* an account is created: `Test.reset(to: <pre-account-height>)` invalidates the account and the next signed transaction fails with `account public key not found`. Second, snapshotting before a fund-transfer the test depends on: the rewind drops the balance, and the next test's `arm`/`claim` transaction reverts on insufficient FLOW with a much less obvious error than "you forgot to fund this account."

For multi-tx state-machine tests specifically, the snapshot height is taken **once, after every persistent participant is in place** and **before any transition transaction runs**. Each test then walks the same baseline state graph from scratch, and the reset between tests means a partially-walked graph from one test can never leak into another.

## Common pitfalls

- **Asserting state directly via contract field access from the test function.** `IntentEscrow.statuses[id]!` from inside a `testXxx` function returns the pre-transaction snapshot, not the post-transaction view. Always go through `Test.executeScript`. Same rule as the `getCurrentBlock().timestamp` staleness in [blockchain-emulation.md § Time manipulation](./blockchain-emulation.md#time-manipulation), applied to every contract field a transaction touched.
- **Sharding a state machine across multiple tests.** A test that walks only Funding → Active and another that walks only Active → Claimable both pass while a regression at the Active → Claimable seam (wrong event field, off-by-one in expiry math) is invisible. Walk the full graph in one test, and split only when the graph branches.
- **Expecting block height to advance with `Test.moveTime`.** It does not — `moveTime` advances the wall clock, `commitBlock` advances the height, and the two are independent. A contract that reads both `getCurrentBlock().height` and `getCurrentBlock().timestamp` needs both calls, in that order, after each transition that should see the new clock.
- **Expecting time to pass without `moveTime`.** Scheduled callbacks, expiry checks, and rate-limit cooldowns all key on `getCurrentBlock().timestamp`, and that value only changes when `Test.moveTime(by:)` runs. A test that submits transactions in a loop will see the same wall clock across all of them unless `moveTime` is interleaved.
- **Asserting on events without resetting between tests.** `Test.eventsOfType(...)` returns *every* event the file has emitted since it started — across `setup()`, all prior tests, and the current one. Either call `Test.reset(to: setupHeight)` in `beforeEach` (which clears the event log) or snapshot `.length` before the act-and-assert pair and subtract. The naive `Test.expect(claimed, Test.haveElementCount(1))` after three tests have run is asserting "three claims happened across the whole file," not "one claim happened in this test."
- **Snapshotting before fund transfers.** `Test.reset(to: <height-before-funding>)` rewinds the FLOW balance back to zero, and the next test that signs a transaction reverts on missing fee balance with a non-obvious error. Always take the snapshot height *after* every fund transfer every test depends on.
- **Letting a multi-tx test silently skip the pre-condition revert.** If `claim2` in the race test were to succeed (because the pre was missing or worded wrong), the test would still pass the `haveElementCount(1)` event check if the second `Claimed` was de-duped at the indexer layer rather than at the contract. Assert on `claim2.error!.message` containing a substring from the pre-condition, not just on the success of `claim1`.
- **Forgetting that a contract field read from inside the test runs against the file-load snapshot.** Even `IntentEscrow.statuses.length` from the test body is stale after the first transaction. The fix is uniform: every state read goes through `Test.executeScript`, every helper returns a scalar projection, and the test body asserts on the helper's return value.
