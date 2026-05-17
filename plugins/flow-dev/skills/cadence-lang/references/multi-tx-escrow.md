# Multi-Transaction Escrow and Custody Pattern

Multi-transaction escrow is a resource that **holds funds across more than one
transaction**, releasing them only when an explicit on-chain condition is met
(a counter-party claim, a timeout, an admin override). It is the canonical
shape for async settlement, scheduled execution, cross-VM coordination, and
intent / order-style flows where the deposit and the consumption happen in
separate transactions submitted by separate signers. It is harder in Cadence
than in Solidity for one reason: Cadence resources have **linear ownership** —
a vault can sit in exactly one place, can only be moved with `<-`, and there is
no shared mutable global state to fall back on. The pattern below pins those
funds inside a resource governed by an explicit state machine so every actor
knows, at every moment, what they are allowed to do.

Cross-links:

- State machine fundamentals — [resource-state-machines.md](resource-state-machines.md)
- For EVM-side escrow via a Cadence-Owned-Account, see
  [coa-lifecycle.md](../../flow-crossvm/references/coa-lifecycle.md) — that
  reference does not yet exist on `origin/main`; it will be added by the
  `feature/flow-crossvm` theme branch.

## The phases

A multi-tx escrow lives through a fixed sequence of phases. The transitions
are one-way (except for `Funding -> Cancelled` as a two-phase-commit escape
hatch):

```
       deposit                arm                claim
  +---------+   ->   +------------+   ->   +-----------+   ->   +--------+
  | Funding |        |  Claimable |        |  Settled  |        | Closed |
  +---------+        +------------+        +-----------+        +--------+
       |                   |
       | cancel (pre-arm)  | deadline passes
       v                   v
  +-----------+      +-----------+
  | Cancelled |      |  Expired  |   -- depositor refund -->  Closed
  +-----------+      +-----------+
```

The phase is an **on-chain field** on the escrow resource. It is the single
source of truth. Events mirror transitions for indexers but are never read by
contract logic — see anti-pattern 2 below.

## Ownership invariants

Three actors, three capability shapes, one phase guard each:

| Actor      | Allowed action       | Allowed phase        | Capability shape                            |
|------------|----------------------|----------------------|---------------------------------------------|
| Depositor  | `deposit`, `cancel`  | `Funding`            | `auth(Deposit) &Escrow`                     |
| Depositor  | `refund`             | `Expired`            | `auth(Refund) &Escrow`                      |
| Recipient  | `claim`              | `Claimable`          | `auth(Claim) &Escrow`                       |
| Admin      | `recover`            | `Expired` only       | `auth(AdminRecover) &Escrow`                |
| Anyone     | `getInfo` (view)     | any                  | `&Escrow` (public)                          |
| Anyone     | mutate state         | `Closed`             | **forbidden** — every entry point panics    |

`pre` conditions on every entitled function enforce the phase. `Closed` is a
terminal sink: every state-mutating entry point starts with
`pre { self.phase != Phase.Closed: "escrow closed" }`.

## Worked example: `IntentEscrow`

A depositor locks `FlowToken` against an intent (e.g. "settle this order on
behalf of recipient before T+1h"). The recipient claims when they fulfil it.
If they do not, the depositor reclaims after the deadline. The admin holds an
emergency recovery key audit-logged on every use.

```cadence
import "FungibleToken"
import "FlowToken"

access(all) contract IntentEscrow {

    // ---- Entitlements: one per actor role -----------------------------------
    access(all) entitlement Deposit
    access(all) entitlement Claim
    access(all) entitlement Refund
    access(all) entitlement AdminRecover

    // ---- Phase: the on-chain state machine ----------------------------------
    access(all) enum Phase: UInt8 {
        access(all) case Funding     // accepting deposits, depositor can still cancel
        access(all) case Claimable   // armed; recipient may claim until deadline
        access(all) case Settled     // recipient claimed; funds gone
        access(all) case Expired     // deadline passed; depositor may refund
        access(all) case Cancelled   // depositor pulled out before arming
        access(all) case Closed      // terminal; no further mutation allowed
    }

    // ---- Events: observability only, never read back as state --------------
    access(all) event Created(id: UInt64, depositor: Address, recipient: Address, deadline: UFix64)
    access(all) event Funded(id: UInt64, amount: UFix64, total: UFix64)
    access(all) event Armed(id: UInt64, total: UFix64)
    access(all) event Claimed(id: UInt64, recipient: Address, amount: UFix64)
    access(all) event Refunded(id: UInt64, depositor: Address, amount: UFix64)
    access(all) event Cancelled(id: UInt64, depositor: Address, amount: UFix64)
    access(all) event AdminRecovered(id: UInt64, admin: Address, amount: UFix64, reason: String)
    access(all) event Closed(id: UInt64)

    // ---- Errors -------------------------------------------------------------
    access(all) view fun wrongPhaseError(want: Phase, got: Phase): String {
        return "wrong phase: expected ".concat(want.rawValue.toString())
            .concat(", got ").concat(got.rawValue.toString())
    }

    // ---- The escrow resource ------------------------------------------------
    access(all) resource Escrow {
        access(all) let depositor: Address
        access(all) let recipient: Address
        access(all) let deadline: UFix64        // unix seconds
        access(all) var phase: Phase
        // Funds live INSIDE the resource. Single location. Linear ownership.
        access(self) var vault: @{FungibleToken.Vault}

        init(depositor: Address, recipient: Address, deadline: UFix64) {
            pre { deadline > getCurrentBlock().timestamp: "deadline in past" }
            self.depositor = depositor
            self.recipient = recipient
            self.deadline = deadline
            self.phase = Phase.Funding
            self.vault <- FlowToken.createEmptyVault(vaultType: Type<@FlowToken.Vault>())
            emit Created(id: self.uuid, depositor: depositor, recipient: recipient, deadline: deadline)
        }

        access(all) view fun balance(): UFix64 { return self.vault.balance }

        // ---- Funding phase -------------------------------------------------
        access(Deposit) fun deposit(from: @{FungibleToken.Vault}) {
            pre { self.phase == Phase.Funding: IntentEscrow.wrongPhaseError(want: Phase.Funding, got: self.phase) }
            let amount = from.balance
            self.vault.deposit(from: <-from)
            emit Funded(id: self.uuid, amount: amount, total: self.vault.balance)
        }

        access(Deposit) fun arm() {
            pre {
                self.phase == Phase.Funding: IntentEscrow.wrongPhaseError(want: Phase.Funding, got: self.phase)
                self.vault.balance > 0.0: "cannot arm empty escrow"
            }
            self.phase = Phase.Claimable
            emit Armed(id: self.uuid, total: self.vault.balance)
        }

        // Two-phase commit escape: depositor can pull out before arming.
        access(Deposit) fun cancel(): @{FungibleToken.Vault} {
            pre { self.phase == Phase.Funding: "can only cancel during Funding" }
            let amount = self.vault.balance
            let out <- self.vault.withdraw(amount: amount)
            self.phase = Phase.Cancelled
            emit Cancelled(id: self.uuid, depositor: self.depositor, amount: amount)
            return <-out
        }

        // ---- Claim phase ---------------------------------------------------
        access(Claim) fun claim(): @{FungibleToken.Vault} {
            pre {
                self.phase == Phase.Claimable: IntentEscrow.wrongPhaseError(want: Phase.Claimable, got: self.phase)
                getCurrentBlock().timestamp <= self.deadline: "deadline passed; depositor must refund"
            }
            let amount = self.vault.balance
            let out <- self.vault.withdraw(amount: amount)
            self.phase = Phase.Settled
            emit Claimed(id: self.uuid, recipient: self.recipient, amount: amount)
            return <-out
        }

        // ---- Expiry transition: callable by anyone, idempotent -------------
        access(all) fun expireIfStale() {
            if self.phase == Phase.Claimable
                && getCurrentBlock().timestamp > self.deadline {
                self.phase = Phase.Expired
            }
        }

        // ---- Refund / recovery --------------------------------------------
        access(Refund) fun refund(): @{FungibleToken.Vault} {
            // Allow caller to lazy-transition Claimable -> Expired.
            self.expireIfStale()
            assert(self.phase == Phase.Expired,
                message: IntentEscrow.wrongPhaseError(want: Phase.Expired, got: self.phase))
            let amount = self.vault.balance
            let out <- self.vault.withdraw(amount: amount)
            emit Refunded(id: self.uuid, depositor: self.depositor, amount: amount)
            return <-out
        }

        access(AdminRecover) fun adminRecover(reason: String, admin: Address): @{FungibleToken.Vault} {
            pre {
                self.phase == Phase.Expired: "admin recovery only on expired escrows"
                reason.length > 0: "audit reason required"
            }
            let amount = self.vault.balance
            let out <- self.vault.withdraw(amount: amount)
            emit AdminRecovered(id: self.uuid, admin: admin, amount: amount, reason: reason)
            return <-out
        }

        // ---- Close: terminal sink ------------------------------------------
        access(all) fun close() {
            pre {
                self.vault.balance == 0.0: "cannot close with funds in escrow"
                self.phase == Phase.Settled
                    || self.phase == Phase.Cancelled
                    || self.phase == Phase.Expired: "must drain before closing"
            }
            self.phase = Phase.Closed
            emit Closed(id: self.uuid)
        }
    }

    access(all) fun createEscrow(
        depositor: Address, recipient: Address, deadline: UFix64
    ): @Escrow {
        return <- create Escrow(depositor: depositor, recipient: recipient, deadline: deadline)
    }
}
```

### Issuing the three capabilities

```cadence
let escrowRef = signer.storage.borrow<&IntentEscrow.Escrow>(from: /storage/myIntent)!

// Depositor: full Deposit + Refund authority on their own escrow.
let depositorCap = signer.capabilities.storage
    .issue<auth(IntentEscrow.Deposit, IntentEscrow.Refund) &IntentEscrow.Escrow>(/storage/myIntent)

// Recipient: only Claim. Publish on inbox or via signed link.
let claimCap = signer.capabilities.storage
    .issue<auth(IntentEscrow.Claim) &IntentEscrow.Escrow>(/storage/myIntent)
signer.inbox.publish(claimCap, name: "escrow-".concat(escrowRef.uuid.toString()), recipient: recipientAddr)

// Public read-only view (anyone can call expireIfStale or balance()).
let publicCap = signer.capabilities.storage
    .issue<&IntentEscrow.Escrow>(/storage/myIntent)
signer.capabilities.publish(publicCap, at: /public/myIntent)
```

## Recovery paths

1. **Timeout-based refund** — the deadline is part of the resource state at
   creation time and cannot be mutated. `expireIfStale` is `access(all)` so
   any party (recipient, depositor, a keeper, a scheduled tx) can drive the
   `Claimable -> Expired` transition; only the depositor's `Refund` cap can
   then withdraw. The funds are always reachable.
2. **Admin recovery** — `adminRecover` requires `Expired` (so it cannot front-
   run a legitimate claim) and a non-empty `reason` string that is emitted in
   the event. Pair with a `singleton Admin` resource gated by `account`
   entitlements so the admin identity is on-chain auditable.
3. **Two-phase commit / cancel-before-arm** — once the depositor calls `arm()`
   the escrow is irrevocably handed to the recipient until the deadline. Until
   then, `cancel()` returns the full balance. This is your only "I changed my
   mind" escape — design transactions so `deposit + arm` happen atomically only
   when you are sure.

A common composition: schedule a `FlowTransactionScheduler` transaction at
`deadline + epsilon` whose handler simply calls `expireIfStale` and then
`refund()` into the depositor's vault. The escrow does not even need to know
that scheduled transactions exist — it just exposes the right entitlements.

## Anti-patterns

### Anti-pattern 1: storing per-tx funds at a storage path indexed by txID

```cadence
// ❌ DO NOT DO THIS
let path = StoragePath(identifier: "escrow_".concat(txID.toString()))!
signer.storage.save(<-vault, to: path)
// Another tx with the same derived txID can overwrite or collide.
// `save` panics on collision, but `load` + re-save loses the resource on
// any interleaved failure. There is no atomic check-and-create primitive
// at the account-storage layer.
```

Why it is wrong: account storage is keyed by path, not by transaction
identity. Two flows that derive the same path race, and the resource is the
only copy. Even if you make the path globally unique, you now have an
unbounded number of dangling storage entries that no central index lists.

```cadence
// ✅ Correct shape: one resource per escrow, owned by a Manager dictionary.
access(all) resource Manager {
    access(self) var escrows: @{UInt64: IntentEscrow.Escrow}
    access(all) fun store(_ e: @IntentEscrow.Escrow): UInt64 {
        let id = e.uuid
        let old <- self.escrows[id] <- e
        destroy old   // always nil because uuid is unique
        return id
    }
    access(all) fun borrow(_ id: UInt64): &IntentEscrow.Escrow? { return &self.escrows[id] }
}
```

The manager dictionary gives atomic insert (`<-!` if you want to force
uniqueness), enumeration via `keys`, and a single storage path to back up.

### Anti-pattern 2: treating events as state

```cadence
// ❌ DO NOT trust event history for control flow.
// "If we ever emitted Claimed for this id, treat it as settled."
// Events are NOT readable from Cadence. They are an off-chain notification
// channel. Reorgs are not a concern on Flow, but consumer indexers can
// lag, miss, or replay them.
```

Always persist phase in the resource. If you need cheap external queries,
expose `view fun getInfo(): EscrowInfo` returning a struct — see Pattern 3
("Report Structs for Resources") in `design-patterns.md`. Events are for
indexers and UIs; they are not state, they are observability.

## Common pitfalls

- **Multi-actor races at the deadline boundary**. If a recipient submits
  `claim()` and a depositor submits `refund()` in the same block, Flow's
  collection ordering will pick a winner — both transactions execute
  sequentially, the second one's `pre` condition fails, the second one
  reverts. This is correct behaviour, not a bug; design UIs to surface the
  loser's revert clearly and resubmit.
- **Fee-of-time discounting on long-lived escrows**. Storage fees accrue to
  the account that holds the resource. For escrows open for weeks, factor in
  account storage minimums on the *holder* (usually the depositor) and
  consider a small handling fee skimmed into a contract-owned vault at
  `arm()` time. Without it, depositors are subsidising the protocol.
- **Dust balances stranding the resource in `Settled`**. If `claim()` is ever
  partial (it is not in this contract, but a variant might be), the closing
  invariant `vault.balance == 0.0` blocks `close()`. Either round to zero on
  the last claim, or expose a `sweepDust` that moves residual UFix64
  rounding (under `0.00000001`) to a protocol vault.
- **Forgetting `close()` leaves a `Settled` resource forever**. The state
  machine is correct but the resource still occupies storage. Either fold
  `close()` into the same tx as the final `claim`/`refund`/`cancel`, or have
  the manager auto-prune terminal escrows on a scheduled sweep.
- **Re-entrancy is not a Cadence concern** the way it is in Solidity — the
  move operator forbids two live references to the same resource — but
  **multi-actor interleaving across blocks is**. Always re-check `self.phase`
  inside `pre` conditions, never cache it across a function boundary.
- **Do not expose entitlements wider than necessary**. A single
  `auth(Deposit, Claim, Refund, AdminRecover) &Escrow` capability is a
  loaded gun: anyone holding it can settle the escrow in either direction.
  Issue one capability per actor with exactly one entitlement.

When you need EVM-side custody (e.g. holding ERC-20 collateral while the
Cadence side fulfils an intent), the same state machine lives on the COA;
see [coa-lifecycle.md](../../flow-crossvm/references/coa-lifecycle.md).
