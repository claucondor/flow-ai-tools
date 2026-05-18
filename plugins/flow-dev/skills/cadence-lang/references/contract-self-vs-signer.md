# Contract `self.account` vs Transaction Signer

Inside a Cadence contract function, `self.account` refers to the account the contract is
deployed on — its owner — not the account that signed the transaction calling the function.
Contracts have no ambient knowledge of who is calling them. The caller identity is a
transaction-level concept; it must be passed explicitly as a function parameter if the
contract needs it. Getting this wrong produces authorization checks that always pass for the
wrong party, events that log the wrong address, and storage operations that target the wrong
account.

---

## The Distinction

| Expression | What it refers to | Where it's valid |
|---|---|---|
| `self.account` (inside a contract) | The account that deployed the contract — the contract's owner | Inside any contract function or `init` |
| `signer.address` (in `prepare`) | The account that signed the current transaction | Only inside a transaction's `prepare` block |
| A parameter `callerAddress: Address` | Whatever the caller explicitly passed | Wherever the function is called |

There is no way for a contract function to read the calling transaction's signer address
without the signer explicitly passing it as an argument or capability reference. This is
intentional: it is part of Cadence's capability-based security model, which requires all
authority to flow through explicit, unforgeable handles.

---

## The Mistake

```cadence
// ❌ WRONG — self.account.address is the DEPLOYER, not the caller

access(all) contract TipJar {

    access(all) event TipReceived(from: Address, amount: UFix64)

    access(all) fun receiveTip(amount: UFix64) {
        // self.account.address is the address TipJar is deployed to.
        // It has NOTHING to do with who called this function.
        emit TipReceived(from: self.account.address, amount: amount)
        //                     ^^^^^^^^^^^^^^^^^^^^^ always the contract owner
    }
}
```

This compiles and runs without error. The event is emitted with `from` always set to the
deployer's address, regardless of who sent the tip. The bug is invisible in testing unless
tests explicitly check the `from` field and use more than one signer.

---

## The Fix — Explicit Caller Parameter

```cadence
// ✅ CORRECT — caller address flows in through an explicit parameter

access(all) contract TipJar {

    access(all) event TipReceived(from: Address, amount: UFix64)

    access(all) fun receiveTip(from callerAddress: Address, amount: UFix64) {
        emit TipReceived(from: callerAddress, amount: amount)
    }
}
```

The corresponding transaction passes the signer's address from the `prepare` block:

```cadence
import "TipJar"
import "FungibleToken"

transaction(amount: UFix64) {
    let vault: @{FungibleToken.Vault}
    let callerAddress: Address

    prepare(signer: auth(BorrowValue) &Account) {
        self.callerAddress = signer.address
        self.vault <- signer.storage
            .borrow<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>(
                from: /storage/flowTokenVault
            )!
            .withdraw(amount: amount)
    }

    execute {
        TipJar.receiveTip(from: self.callerAddress, amount: amount)
        // deposit vault...
    }
}
```

The `prepare` block is the **only place** where the transaction signer is available. Capture
what you need into transaction-level `let` bindings in `prepare`, then use those in `execute`.

---

## Variant — Passing an Authorized Reference Instead of an Address

When the contract needs to write to the caller's storage (not just read their address), pass
an authorized reference rather than a plain `Address`. The entitlement requested should be
the minimum required for the operation.

```cadence
access(all) contract Vault {

    // Takes an authorized reference so it can deposit into the caller's receiver.
    access(all) fun distributeReward(
        recipient: &{FungibleToken.Receiver},
        amount: UFix64
    ) {
        // `recipient` was passed by the transaction's prepare block from the signer's account.
        // It has nothing to do with self.account.
        let reward <- self.account.storage
            .borrow<auth(FungibleToken.Withdraw) &{FungibleToken.Vault}>(
                from: /storage/rewardPool
            )!
            .withdraw(amount: amount)
        recipient.deposit(from: <-reward)
    }
}
```

Transaction side:

```cadence
import "Vault"
import "FungibleToken"

transaction(amount: UFix64) {
    let receiver: &{FungibleToken.Receiver}

    prepare(signer: auth(BorrowValue) &Account) {
        self.receiver = signer.capabilities
            .borrow<&{FungibleToken.Receiver}>(/public/flowTokenReceiver)!
    }

    execute {
        Vault.distributeReward(recipient: self.receiver, amount: amount)
    }
}
```

---

## When `self.account` IS Correct

`self.account` is the right choice for operations on the contract's own storage — resources
the contract itself owns and manages regardless of who is calling.

```cadence
// ✅ Correct use of self.account — reading the contract's own state

access(all) contract Counter {

    access(all) var count: Int

    init() {
        self.count = 0
        // self.account.storage.save(...) is appropriate here —
        // storing data in the deployer account's storage.
        self.count = 0
    }

    // Contract emits its own address as the source — intentional.
    access(all) event Incremented(by: Address, amount: Int)

    access(all) fun increment(by callerAddress: Address, amount: Int) {
        self.count = self.count + amount
        // self.account.address = deployer (correct for "which contract did this")
        // callerAddress = who called (correct for "who triggered this")
        emit Incremented(by: callerAddress, amount: amount)
    }
}
```

The rule of thumb: use `self.account` for the contract's own resources and storage; use an
explicit parameter for anything about the caller.

---

## The Broader Principle

Contracts are modules, not objects with ambient context. When a function on `MyContract` is
called from a transaction, the contract code runs in the transaction's execution context but
does NOT receive a reference to the transaction's signers automatically. Everything flows
through function parameters, events are the only way to record caller identity after the fact
(and that identity comes from parameters too), and `self.account` is the deployer account —
period.

This is architecturally similar to how Solidity's `msg.sender` works, but with one critical
difference: in Cadence there is no implicit `msg.sender`. You must thread the caller's address
(or an authorized reference) through every function that needs it. Forgetting to do so means
the function simply cannot access caller identity — it silently falls back to `self.account`,
which is the deployer.

---

## Common Pitfalls

- **Access control checks against `self.account.address`** — A function that checks
  `pre { self.account.address == adminAddress }` passes only when the deployer IS the admin
  and will pass for every caller because `self.account.address` is constant. Use the
  capability-based admin pattern from
  [`admin-via-capability.md`](admin-via-capability.md) instead.
- **Minting to `self.account` instead of the caller** — A mint function that deposits tokens
  into `self.account.storage` deposits them into the deployer's storage, not the caller's.
  Always take a `&{FungibleToken.Receiver}` or `&{NonFungibleToken.CollectionPublic}` parameter
  for the deposit target.
- **Logging the wrong address in events** — Events that record `self.account.address` as
  "who performed this action" will always show the deployer. Capture the intended address as
  a parameter.
- **In test files, confusing the admin account with self.account** — When a test calls a
  contract function, `self.account.address` inside that function is the contract's testing
  alias address (e.g., `0x0000000000000007`), not the test's `admin` variable. This mismatch
  can make access-control assertions pass in tests but fail in production if the addresses differ.

---

## See Also

- [`transactions.md`](transactions.md) — `prepare` phase, signer access, and the four
  transaction phases
- [`admin-via-capability.md`](admin-via-capability.md) — The correct pattern for restricting
  contract functions to an admin, using unforgeable capability handles rather than address comparisons
- [`access-control.md`](access-control.md) — Access modifiers and entitlements
- [`accounts.md`](accounts.md) — Account storage, the `Account` type, and capabilities
