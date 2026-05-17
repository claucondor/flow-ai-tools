# COA Lifecycle for Escrow

A Cadence Owned Account (COA) is a Cadence resource (`@EVM.CadenceOwnedAccount`)
whose `uuid` deterministically derives an EVM address — one Flow account, two
VMs, no separate keypair. Because the COA is a resource and not a key, custody
follows Cadence's linear-ownership rules: whoever holds the resource (or the
right entitled capability to it) controls the EVM address. This is precisely
what makes COAs the right escrow primitive on Flow — funds and call authority
both live in a single account, atomic with surrounding Cadence logic, and there
is no off-chain signer to compromise. It is also what makes mishandling
dangerous: an `auth(EVM.Call) &EVM.CadenceOwnedAccount` capability is bearer
authority over an EVM address, capabilities are not view-only, and a misplaced
`publish` can drain the account.

This reference covers how to **get** a COA and keep it safe. For what to do
with it once you have one (`coa.call`, ABI encoding, `result.status` handling),
see [evm-call.md](evm-call.md). For the Cadence-only escrow analog, see
[../../cadence-lang/references/multi-tx-escrow.md](../../cadence-lang/references/multi-tx-escrow.md);
this reference is the cross-VM extension of that pattern.

## Entitlements on `EVM.CadenceOwnedAccount`

The EVM contract exposes five entitlements on the COA resource. Each `coa.*`
method requires a specific one — capabilities you issue should request the
minimum that the consumer actually needs.

| Entitlement       | Grants                                       | Use for                              |
|-------------------|----------------------------------------------|--------------------------------------|
| `EVM.Call`        | `coa.call`, `coa.callWithSigAndArgs`         | invoking EVM contracts               |
| `EVM.Withdraw`    | `coa.withdraw` (FLOW out of EVM to Cadence)  | bridging back to Cadence             |
| `EVM.Deploy`      | `coa.deploy`                                 | publishing new EVM contracts         |
| `EVM.Validate`    | `coa.protectedAddress`                       | proving ownership without doing more |
| `EVM.Owner`       | every privileged method on the COA           | full custody (avoid sharing)         |

`coa.deposit`, `coa.balance`, `coa.address`, and `coa.dryCall` are
**un-entitled** — anyone with even a bare `&EVM.CadenceOwnedAccount` reference
can deposit into the COA, read its balance, read its address, and dry-call
EVM contracts. That is intentional: deposit is always safe (you only add
value) and reads cannot mutate state.

## Canonical storage paths

By convention, a COA lives at:

- `/storage/evm` — the resource itself
- `/public/evm`  — an **un-entitled** public capability for reads

These paths are not enforced by the contract, but every Flow tool (FCL,
wallets, indexers) expects them. Deviating breaks integrations.

## Creating a COA

The creation transaction is idempotent — it must not panic if the user already
has a COA. It also issues two capabilities: one private auth capability for
internal use, one public un-entitled capability for reads.

```cadence
import "EVM"

transaction() {
    prepare(signer: auth(
        SaveValue,
        IssueStorageCapabilityController,
        PublishCapability,
        UnpublishCapability
    ) &Account) {

        // 1. Idempotency check — only create if one does not already exist.
        if signer.storage.type(at: /storage/evm) != nil {
            return
        }

        // 2. Create the COA. EVM.createCadenceOwnedAccount() returns a fresh
        //    @EVM.CadenceOwnedAccount whose EVM address is derived from its uuid.
        let coa <- EVM.createCadenceOwnedAccount()

        // 3. Save at the canonical path.
        signer.storage.save(<-coa, to: /storage/evm)

        // 4. Issue and publish the read-only public capability.
        //    Note: this is un-entitled — &EVM.CadenceOwnedAccount only.
        //    Anyone can call balance(), address(), deposit(), dryCall() through it.
        //    They CANNOT call call(), withdraw(), or deploy().
        signer.capabilities.unpublish(/public/evm)   // safe; no-op if absent
        let publicCap = signer.capabilities.storage
            .issue<&EVM.CadenceOwnedAccount>(/storage/evm)
        signer.capabilities.publish(publicCap, at: /public/evm)
    }
}
```

The public path must stay un-entitled. Any `auth(...) &EVM.CadenceOwnedAccount`
published at `/public/evm` becomes bearer-controllable — see anti-pattern 1.
A script can read the COA's EVM address via
`getAccount(addr).capabilities.borrow<&EVM.CadenceOwnedAccount>(/public/evm)!.address()`.

## Funding a COA from Cadence

Native FLOW lives as a `@FlowToken.Vault` on the Cadence side. To move it
into the COA's EVM balance, withdraw a chunk and call `coa.deposit`. The
`deposit` method is `access(all)` — no capability needed.

```cadence
import "EVM"
import "FungibleToken"
import "FlowToken"

transaction(amount: UFix64) {
    prepare(signer: auth(BorrowValue) &Account) {
        let vaultRef = signer.storage.borrow<auth(FungibleToken.Withdraw) &FlowToken.Vault>(
            from: /storage/flowTokenVault
        ) ?? panic("could not borrow FlowToken vault")
        let coa = signer.storage.borrow<&EVM.CadenceOwnedAccount>(from: /storage/evm)
            ?? panic("no COA at /storage/evm — run create-coa.cdc first")

        // coa.deposit requires the concrete @FlowToken.Vault type, not the interface.
        let chunk <- vaultRef.withdraw(amount: amount) as! @FlowToken.Vault
        coa.deposit(from: <-chunk)
    }
}
```

### attoFLOW vs UFix64 — the conversion

EVM uses 18 decimal places (`attoflow`, a `UInt`); `FlowToken.Vault` uses
`UFix64` (8 decimals). `EVM.Balance` carries both:

```cadence
// 1 FLOW = 10^18 attoflow
let oneFlow = EVM.Balance(attoflow: 1_000_000_000_000_000_000)

let b = EVM.Balance(attoflow: 0)
b.setFLOW(flow: 1.5)                  // converts from UFix64
let asFlow: UFix64 = b.inFLOW()       // 1.50000000
let asAtto: UInt   = b.inAttoFLOW()   // 1_500_000_000_000_000_000
```

You do not need `EVM.Balance` for `coa.deposit` — it takes a vault and reads
the amount from `vault.balance`. `EVM.Balance` matters for `coa.withdraw`
and for passing `value` to `coa.call`.

## Withdrawing FLOW from a COA back to Cadence

The reverse direction uses `coa.withdraw(balance: EVM.Balance)` and requires
the `EVM.Withdraw` entitlement.

```cadence
import "EVM"
import "FungibleToken"
import "FlowToken"

transaction(amountFlow: UFix64) {
    prepare(signer: auth(BorrowValue) &Account) {

        // Withdraw requires auth(EVM.Withdraw). EVM.Owner subsumes it.
        let coa = signer.storage.borrow<auth(EVM.Withdraw) &EVM.CadenceOwnedAccount>(
            from: /storage/evm
        ) ?? panic("no COA at /storage/evm")

        // Build an EVM.Balance from the UFix64 amount.
        let bal = EVM.Balance(attoflow: 0)
        bal.setFLOW(flow: amountFlow)

        // withdraw returns @FlowToken.Vault. Smallest withdrawable unit is
        // 1e10 attoflow because UFix64 has only 8 decimals (10^18 / 10^8 = 10^10).
        // Smaller amounts panic.
        let vault <- coa.withdraw(balance: bal)

        // Deposit back into the signer's FlowToken vault.
        let receiver = signer.capabilities.borrow<&{FungibleToken.Receiver}>(
            /public/flowTokenReceiver
        ) ?? panic("no FlowToken receiver")
        receiver.deposit(from: <-vault)
    }
}
```

### Why `coa.withdraw` is safer than an EVM-side transfer

A naive alternative — `coa.call` into a Solidity router that emits a
"release" event for an off-chain bridge — has three failure modes absent
from `coa.withdraw`: it relies on off-chain relayers, opens an
event-then-process gap, and requires inspecting `result.status` because a
silent EVM failure does not revert the Cadence transaction (see
[evm-call.md](evm-call.md)). `coa.withdraw` runs entirely inside the Flow
protocol, returns the `@FlowToken.Vault` atomically in the same Cadence
transaction, and panics on failure. For native-FLOW movement between VMs,
always prefer `coa.deposit` / `coa.withdraw`; save `coa.call` for ERC20s and
Solidity logic.

## Custody pattern: COA inside an escrow resource

For escrow semantics across multiple transactions, do not store the COA
inside the escrow resource — store a **capability** to it. The COA continues
to live in the depositor's account at `/storage/evm`; the escrow only borrows
authority during the phases where it needs it. The phase machine is the same
one documented in
[../../cadence-lang/references/multi-tx-escrow.md](../../cadence-lang/references/multi-tx-escrow.md);
only the leaf actions differ (`coa.call` and `coa.withdraw` instead of
`vault.withdraw`).

```cadence
import "EVM"
import "FungibleToken"

access(all) contract IntentEscrowCrossVM {

    access(all) entitlement Claim
    access(all) entitlement Refund

    access(all) enum Phase: UInt8 {
        access(all) case Funding
        access(all) case Claimable
        access(all) case Settled
        access(all) case Expired
        access(all) case Closed
    }

    access(all) event Armed(id: UInt64, balanceAttoFlow: UInt)
    access(all) event Claimed(id: UInt64, amountAttoFlow: UInt)
    access(all) event Refunded(id: UInt64, amountAttoFlow: UInt)
    access(all) event Closed(id: UInt64)

    access(all) resource Escrow {
        access(all) let depositor: Address
        access(all) let recipient: Address
        access(all) let deadline: UFix64
        access(self) var phase: Phase

        // Capability to the depositor's COA — NOT the resource itself.
        // EVM.Call + EVM.Withdraw is the minimum: pay an EVM contract on
        // claim, refund FLOW on expiry. Nothing else.
        access(self) let depositorCOA:
            Capability<auth(EVM.Call, EVM.Withdraw) &EVM.CadenceOwnedAccount>
        access(all) let depositorCOAControllerID: UInt64

        init(
            depositor: Address,
            recipient: Address,
            deadline: UFix64,
            depositorCOA: Capability<auth(EVM.Call, EVM.Withdraw) &EVM.CadenceOwnedAccount>,
            controllerID: UInt64
        ) {
            pre {
                depositorCOA.check(): "depositor COA capability invalid"
                deadline > getCurrentBlock().timestamp: "deadline in past"
            }
            self.depositor = depositor
            self.recipient = recipient
            self.deadline = deadline
            self.phase = Phase.Funding
            self.depositorCOA = depositorCOA
            self.depositorCOAControllerID = controllerID
        }

        access(all) view fun getPhase(): Phase { return self.phase }

        access(all) fun arm() {
            pre { self.phase == Phase.Funding: "wrong phase" }
            let coa = self.depositorCOA.borrow() ?? panic("COA cap revoked")
            self.phase = Phase.Claimable
            emit Armed(id: self.uuid, balanceAttoFlow: coa.balance().attoflow)
        }

        // The escrow holds the auth cap but only exercises it during Claimable.
        access(Claim) fun claim(
            to: EVM.EVMAddress, data: [UInt8], gasLimit: UInt64, value: EVM.Balance
        ) {
            pre {
                self.phase == Phase.Claimable: "wrong phase"
                getCurrentBlock().timestamp <= self.deadline: "deadline passed"
            }
            let coa = self.depositorCOA.borrow() ?? panic("COA cap revoked")
            let result = coa.call(to: to, data: data, gasLimit: gasLimit, value: value)
            // All-or-nothing: if EVM call failed, revert the whole Cadence tx.
            assert(result.status == EVM.Status.successful,
                message: "EVM claim call failed: ".concat(result.errorMessage))
            self.phase = Phase.Settled
            emit Claimed(id: self.uuid, amountAttoFlow: value.attoflow)
        }

        access(all) fun expireIfStale() {
            if self.phase == Phase.Claimable
                && getCurrentBlock().timestamp > self.deadline {
                self.phase = Phase.Expired
            }
        }

        access(Refund) fun refund(value: EVM.Balance): @{FungibleToken.Vault} {
            self.expireIfStale()
            assert(self.phase == Phase.Expired, message: "wrong phase")
            let coa = self.depositorCOA.borrow() ?? panic("COA cap revoked")
            let vault <- coa.withdraw(balance: value)
            emit Refunded(id: self.uuid, amountAttoFlow: value.attoflow)
            return <-vault
        }

        access(all) fun close() {
            pre {
                self.phase == Phase.Settled || self.phase == Phase.Expired:
                    "must drain before closing"
            }
            self.phase = Phase.Closed
            emit Closed(id: self.uuid)
        }
    }
}
```

### Issuing and revoking the COA capability

The depositor issues a **narrow** auth cap to the escrow at setup time, and
**revokes** it (deletes the capability controller) after the escrow closes.
Revocation is the cross-VM equivalent of draining a Cadence vault:
capabilities, not vaults, are what carry authority here.

```cadence
import "EVM"

// Setup: issue a narrow cap and hand it to the escrow.
transaction(recipient: Address, deadline: UFix64) {
    prepare(signer: auth(SaveValue, IssueStorageCapabilityController) &Account) {
        let cap = signer.capabilities.storage
            .issue<auth(EVM.Call, EVM.Withdraw) &EVM.CadenceOwnedAccount>(/storage/evm)
        // The controller ID is on the returned capability via cap.id.
        let escrow <- IntentEscrowCrossVM.createEscrow(
            depositor: signer.address, recipient: recipient, deadline: deadline,
            depositorCOA: cap, controllerID: cap.id
        )
        signer.storage.save(<-escrow, to: /storage/myCrossVmEscrow)
    }
}

// Close: delete the capability controller. Any subsequent .borrow() returns nil.
transaction(controllerID: UInt64) {
    prepare(signer: auth(BorrowValue, GetStorageCapabilityController) &Account) {
        let escrow = signer.storage.borrow<&IntentEscrowCrossVM.Escrow>(
            from: /storage/myCrossVmEscrow
        ) ?? panic("no escrow")
        assert(escrow.getPhase() == IntentEscrowCrossVM.Phase.Closed, message: "not closed")
        for c in signer.capabilities.storage.getControllers(forPath: /storage/evm) {
            if c.capabilityID == controllerID { c.delete(); break }
        }
    }
}
```

## Anti-patterns

### Anti-pattern 1: publishing an auth capability at `/public/evm`

```cadence
// CRITICAL — anyone can drain the COA.
let cap = signer.capabilities.storage
    .issue<auth(EVM.Call) &EVM.CadenceOwnedAccount>(/storage/evm)
signer.capabilities.publish(cap, at: /public/evm)   // wrong target
```

Why it is wrong: `/public/*` capabilities are resolvable by *any* Cadence
script or transaction via `getAccount(addr).capabilities.get<...>(...)`.
Publishing `auth(EVM.Call)` there means any transaction signed by any other
account can borrow your COA and call EVM contracts — including ERC20
`transfer` to drain every token the COA holds, or routing trades on a DEX
through your address.

The public capability should always be **un-entitled**:

```cadence
// Correct — only ungated reads.
let publicCap = signer.capabilities.storage
    .issue<&EVM.CadenceOwnedAccount>(/storage/evm)
signer.capabilities.publish(publicCap, at: /public/evm)
```

If a third party needs to call EVM contracts on your behalf, hand them a
**narrow** auth capability via `inbox.publish(cap, name:, recipient:)` to a
single, named recipient address — not via `/public/*`.

### Anti-pattern 2: sharing one `auth(EVM.Call)` cap with multiple consumers

Capabilities are **not view-only proofs of ownership**. Anyone holding an
`auth(EVM.Call) &EVM.CadenceOwnedAccount` can call any EVM contract through
the COA — including ERC20 approvals and transfers. Sharing one cap with
escrow A, escrow B, a router, and a frontend helper means a single bug in
any one of them drains the COA, and there is no way to revoke from just
one consumer.

```cadence
// Correct: one narrow cap per consumer, each with its own controller ID.
let escrowCap = signer.capabilities.storage
    .issue<auth(EVM.Call, EVM.Withdraw) &EVM.CadenceOwnedAccount>(/storage/evm)
let routerCap = signer.capabilities.storage
    .issue<auth(EVM.Call) &EVM.CadenceOwnedAccount>(/storage/evm)
// Deleting only routerCap's controller leaves escrowCap intact.
```

### Anti-pattern 3: storing the COA resource inside another resource

```cadence
// Bad — the COA leaves the depositor's account.
access(all) resource BadEscrow {
    access(self) var coa: @EVM.CadenceOwnedAccount
}
```

The COA is no longer at `/storage/evm` on the depositor's account, so every
wallet UI and indexer breaks. You also lose the natural revocation point:
there is no capability to delete, only a resource to move back (which
requires the holder's cooperation). Keep the COA at `/storage/evm`; pass a
narrow auth capability into the escrow.

## Common pitfalls

- **`coa.withdraw` smallest unit is 1e10 attoflow.** `FlowToken.Vault` uses
  `UFix64` (8 decimals). Withdrawing less than `0.00000001` FLOW panics with
  `withdraw failed! smallest unit allowed to transfer is 1e10 attoFlow`.
  Round on the Cadence side before calling.
- **`coa.deposit` is un-entitled.** Anyone with even a bare reference can
  deposit. This is fine (it only adds value), but do not confuse it with a
  sign of capability ownership. The public cap can deposit; only the auth
  cap can withdraw or call.
- **Idempotency on create.** Always check `signer.storage.type(at: /storage/evm)`
  before creating. Calling `EVM.createCadenceOwnedAccount()` and `save()` a
  second time panics because the path is occupied — but the freshly created
  resource is *destroyed* on revert, leaving the original COA intact. A user
  re-running the setup transaction by accident still loses the gas. Make it
  idempotent.
- **`/public/evm` must be un-entitled.** Re-publishing an auth capability to
  `/public/evm` by mistake (e.g. copying from a setup script that issues
  `auth(EVM.Call)`) silently makes the COA bearer-controllable. Always
  `unpublish` then re-publish, and always type-check the capability shape in
  code review.
- **EVM.Call atomicity gotcha — see [evm-call.md](evm-call.md).** A failed
  EVM call inside `claim()` does NOT automatically revert the surrounding
  Cadence transaction. The escrow's `claim` method here uses
  `assert(result.status == EVM.Status.successful, ...)` to enforce all-or-
  nothing semantics — drop that assert and a "successful" claim with a
  silently-failed EVM payment becomes possible.
- **Controller IDs are not stable across recreations.** If a depositor
  deletes their COA and re-creates it, every previously issued capability is
  permanently invalid (the storage path now holds a new resource with a new
  uuid → new EVM address). Recovery flows must account for this.
- **CU ceiling.** Funding a COA, calling an EVM contract, and withdrawing in
  the same Cadence transaction shares the 9999 CU per-tx budget across both
  VMs. Multi-step escrow flows that include a `coa.deposit` plus several
  `coa.call`s should split across transactions; see
  [cu-ceiling.md](cu-ceiling.md). [UNVERIFIED: exact CU cost of
  `coa.deposit` and `coa.withdraw` — measure on the target network before
  budgeting tight transactions.]
- **Account storage minimum applies to the COA holder.** The
  `@EVM.CadenceOwnedAccount` resource itself occupies Cadence storage on the
  depositor's account, in addition to its EVM-side balance. Long-lived
  escrows where many depositors create COAs they later abandon will
  accumulate storage fees on those accounts; budget accordingly.
