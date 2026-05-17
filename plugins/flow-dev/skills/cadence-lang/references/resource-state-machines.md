# Resource State Machines

A resource state machine is a Cadence resource that holds a `phase` field and
moves through a fixed set of phases via dedicated transition methods. Cadence
encourages this pattern because resources are linear (they live in exactly one
place) and entitlements gate who may call which transition. The right place to
enforce a transition is in a `pre` condition on an entitled method: pre-checks
run *before* any side effects, so a wrong-phase call reverts cleanly with a
clear message and zero partial state.

Use this pattern whenever a resource has a lifecycle: an order
(`Pending → Active → Settled`), an escrow, a vesting schedule, an auction,
a multi-step swap, or anything that holds funds across transactions. For the
cross-transaction-with-funds case in particular, see
[multi-tx-escrow.md](multi-tx-escrow.md), which builds on the rules here.

## The four ingredients

1. **An enum** for the phase (not a `String`, not a `UInt8`, not a `Bool`).
2. **An `access(self)` field** holding the current phase — never `access(all) var`.
3. **One entitled method per legal transition**, each guarded by `pre` checks on
   the current phase.
4. **A terminal phase** (`Settled`, `Cancelled`, `Closed`) that no transition
   can leave.

## Canonical example: order with four phases

```cadence
import "FungibleToken"

access(all) contract OrderBook {

    access(all) entitlement Operator   // can advance Pending -> Active
    access(all) entitlement Settler    // can advance Active  -> Settled
    access(all) entitlement Canceller  // can cancel from Pending or Active

    access(all) enum Phase: UInt8 {
        access(all) case Pending     // created, not yet activated
        access(all) case Active      // live, accepting fills
        access(all) case Settled     // terminal: paid out
        access(all) case Cancelled   // terminal: refunded
    }

    access(all) event OrderCreated(id: UInt64)
    access(all) event OrderActivated(id: UInt64)
    access(all) event OrderSettled(id: UInt64, amount: UFix64)
    access(all) event OrderCancelled(id: UInt64)

    access(all) resource Order {
        access(all) let id: UInt64
        access(self) var phase: Phase             // private — only entitled methods mutate
        access(self) var escrow: @{FungibleToken.Vault}

        init(id: UInt64, deposit: @{FungibleToken.Vault}) {
            self.id = id
            self.phase = Phase.Pending
            self.escrow <- deposit
            emit OrderCreated(id: id)
        }

        access(all) view fun getPhase(): Phase { return self.phase }

        // Pending -> Active
        access(Operator) fun activate() {
            pre {
                self.phase == Phase.Pending:
                    "Order \(self.id) cannot activate from phase \(self.phase.rawValue)"
            }
            post {
                self.phase == Phase.Active: "activate() must leave phase Active"
            }
            self.phase = Phase.Active
            emit OrderActivated(id: self.id)
        }

        // Active -> Settled (pays out the escrow)
        access(Settler) fun settle(to: &{FungibleToken.Receiver}): UFix64 {
            pre {
                self.phase == Phase.Active:
                    "Order \(self.id) cannot settle from phase \(self.phase.rawValue)"
            }
            post {
                self.phase == Phase.Settled: "settle() must leave phase Settled"
            }
            let amount = self.escrow.balance
            let payout <- self.escrow.withdraw(amount: amount)
            to.deposit(from: <-payout)
            self.phase = Phase.Settled
            emit OrderSettled(id: self.id, amount: amount)
            return amount
        }

        // Pending or Active -> Cancelled (refunds the escrow)
        access(Canceller) fun cancel(to: &{FungibleToken.Receiver}) {
            pre {
                self.phase == Phase.Pending || self.phase == Phase.Active:
                    "Order \(self.id) cannot cancel from phase \(self.phase.rawValue)"
            }
            post {
                self.phase == Phase.Cancelled: "cancel() must leave phase Cancelled"
            }
            let amount = self.escrow.balance
            let refund <- self.escrow.withdraw(amount: amount)
            to.deposit(from: <-refund)
            self.phase = Phase.Cancelled
            emit OrderCancelled(id: self.id)
        }
    }

    access(all) fun createOrder(id: UInt64, deposit: @{FungibleToken.Vault}): @Order {
        return <- create Order(id: id, deposit: <-deposit)
    }
}
```

### Why each ingredient matters

- **Enum** — the compiler tracks all cases, and `phase.rawValue` gives a stable
  number for events and logs. You cannot accidentally compare against
  `"Penidng"` because no string is involved.
- **`access(self) var phase`** — only methods inside the `Order` resource body
  can write to `phase`. No external caller, no reference, no capability can
  flip the phase by direct assignment.
- **Entitled transition methods** — `activate`, `settle`, and `cancel` each
  carry a different entitlement. Whoever issues the capability decides which
  role a borrower has; the protocol cannot be tricked into letting a "filler"
  cancel an order.
- **`pre` on the *from* phase** — checked before any field is written or any
  vault is moved. A wrong-phase call reverts with no side effects.
- **`post` on the *to* phase** — defends the method body against bugs:
  if a future edit forgets `self.phase = Phase.Settled`, the post fails.

## Why `pre` is the right place — not `if` in the body

```cadence
// ❌ WRONG: phase check inside the body, after side effects already happened
access(Settler) fun settleBad(to: &{FungibleToken.Receiver}) {
    let amount = self.escrow.balance
    let payout <- self.escrow.withdraw(amount: amount)   // SIDE EFFECT
    to.deposit(from: <-payout)                            // SIDE EFFECT
    if self.phase != Phase.Active {                       // too late
        panic("wrong phase")
    }
    self.phase = Phase.Settled
}
```

The vault has already been withdrawn and deposited by the time the check fires.
A pre-condition would have rejected the call before a single token moved:

```cadence
// ✅ RIGHT: pre runs before the body
access(Settler) fun settle(to: &{FungibleToken.Receiver}) {
    pre { self.phase == Phase.Active: "wrong phase \(self.phase.rawValue)" }
    // ... side effects only run if pre passed
}
```

Pre-conditions also surface in events and logs as the canonical "why did this
revert" message, which is much friendlier to integrators than a mid-body panic
that depends on execution order.

## Anti-pattern: free `String` as the state field

```cadence
// ❌ DANGEROUS
access(all) resource OrderBad {
    access(all) var status: String   // also public — see next anti-pattern

    access(all) fun activate() {
        if self.status == "pendnig" {  // typo, never matches
            self.status = "active"
        }
    }
    access(all) fun settle() {
        if self.status == "Active" {   // case mismatch with "active"
            self.status = "settled"
        }
    }
}
```

Problems, in order of severity:

1. **No exhaustiveness check.** The compiler cannot tell you that `"pendnig"`,
   `"Active"`, and `"active"` are different. Every transition silently no-ops
   if you typo the comparison string.
2. **No enumeration of legal phases.** A reader has to grep every transition
   to find what strings are even valid.
3. **Comparison inside the body, not in a `pre`.** Even if the strings were
   correct, an inner-`if` would only revert *after* the function did partial
   work (events emitted, balances mutated). A `pre` rejects before any side
   effect; an inner-`if` is "discover failure halfway through."
4. **No way to switch-exhaustively** over `String`. With an enum, a `switch`
   statement over all four phases forces you to add a branch when you add
   a phase.

Use an enum. Always.

## Anti-pattern: `access(all) var phase` — anyone can flip the state

```cadence
// ❌ CRITICAL: external mutation
access(all) resource OrderUnsafe {
    access(all) var phase: Phase   // public AND var

    access(Settler) fun settle() {
        pre { self.phase == Phase.Active: "wrong phase" }
        // ...
    }
}

// In a transaction, anyone holding ANY reference to OrderUnsafe can do:
let ref = &order as &OrderUnsafe
ref.phase = Phase.Active   // bypasses settle()'s pre entirely
```

`access(all) var` exposes the setter — the entitlement on `settle()` is useless
because the attacker doesn't need to call `settle()`; they just write `phase`
directly through any reference. The fix is two-fold:

```cadence
// ✅ RIGHT: phase is privately stored, only entitled methods mutate it
access(all) resource Order {
    access(self) var phase: Phase                 // private storage
    access(all) view fun getPhase(): Phase {      // public read-only view
        return self.phase
    }
    // every transition is entitled AND guarded by pre
}
```

The same applies to `access(contract)` and `access(account)` on `var phase` —
both still expose the implicit setter to a wider audience than you usually
want. Default to `access(self) var` for state fields, and read them through
`access(all) view fun get…()`.

## Anti-pattern: branching inside a closure or callback

```cadence
// ❌ Phase check happens during a callback, after escrow already moved
access(Settler) fun settleViaCallback(_ cb: fun(): &{FungibleToken.Receiver}) {
    let to = cb()                              // arbitrary code runs
    let amount = self.escrow.balance
    let payout <- self.escrow.withdraw(amount: amount)
    to.deposit(from: <-payout)
    if self.phase != Phase.Active {            // check after the fact
        panic("wrong phase")
    }
    self.phase = Phase.Settled
}
```

Two failures:

1. The phase check runs after the callback and after the withdrawal. A
   re-entrant or misordered callback can observe a half-settled order.
2. The error surfaces mid-execution. A pre-condition gives you the failure
   *before* the callback fires.

Keep `pre` at the very top of every transition method, before any external
call, withdrawal, or capability borrow.

## Reading the phase from off-chain

Always expose a `view` reader so scripts can pre-flight:

```cadence
access(all) view fun getPhase(): Phase { return self.phase }
```

A frontend can call `getPhase()` before sending an `activate()` transaction
and tell the user "this order is already Active" without burning a fee on the
revert.

## Common pitfalls

- **Returning to a previous phase.** Once you reach a terminal phase
  (`Settled`, `Cancelled`), there must be no transition that leaves it.
  Audit by writing one `pre` per method on the *from* phase and checking
  that no method has the terminal phase in its allowed-from set.
- **Forgetting `post` on the new phase.** If only `pre` exists, a future edit
  can silently skip the `self.phase = …` assignment and the transition becomes
  a no-op that still emits an event. `post { self.phase == Phase.Active }`
  catches that on the next test run.
- **Using a `Bool` instead of an enum.** `var isActive: Bool` cannot represent
  `Cancelled` vs `Settled` — both look like `false`. Always use an enum, even
  for two phases.
- **Comparing `phase` with `phase.rawValue`.** Compare enum-to-enum
  (`self.phase == Phase.Active`), not enum-to-number. The compiler enforces
  the enum form; the numeric form silently accepts any `UInt8`.
- **Putting the entitlement check inside the body.** `if !auth …` is wrong —
  Cadence already gates the call at the reference layer via
  `access(Settler)`. The body should assume the entitlement was verified.
- **Holding funds across phases without a state machine.** If a resource holds
  a vault and the lifecycle spans transactions, use the state machine pattern
  here together with [multi-tx-escrow.md](multi-tx-escrow.md). Skipping the
  phase field is how you get "double-settle" and "settle after cancel" bugs.
- **Skipping `Cancelled`.** A two-phase machine without a refund path leaves
  funds stuck if the protocol is paused or the counterparty disappears. Always
  add at least one terminal phase that returns escrowed value.
