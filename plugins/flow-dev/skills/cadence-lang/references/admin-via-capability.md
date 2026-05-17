# Admin via Capability (Modern Admin Role Pattern)

The modern Cadence admin pattern keeps the admin resource in the contract's own
account and hands out **entitled capabilities** to administrator addresses.
Admins never receive an `&Account` reference, an `AuthAccount`, or a custodied
key — they only borrow a typed, scoped capability whose entitlements gate
exactly the privileged methods they are allowed to call. Because capabilities
are unforgeable, revocable, and individually scoped, this pattern replaces both
the legacy "save Admin resource in deployer storage" model and the worse
"address allowlist + `tx.signer.address` check" model. The result is admin
power that is auditable, rotatable without a contract upgrade, and minimally
privileged by construction.

## Why capabilities are the right primitive

| Property | `&Account` / `AuthAccount` admin | Address allowlist (`{Address: Bool}`) | Entitled capability |
|---|---|---|---|
| Scope | Whole account | One contract function | One method set (per entitlement) |
| Revocable | Only by losing keys | Requires contract upgrade or flag flip with state migration risk | Yes, via controller `.delete()` or `unpublish` |
| Forgeable | N/A (signer based) | A bug or compromised key grants full admin | No — capabilities are unforgeable references |
| Multi-admin | Custodial keys / multi-sig wallet | Multiple addresses in a map; all share the same powers | Distinct caps per role per address |
| Migrates if deployer key lost | Painful — admin is tied to that account | Painful — must upgrade contract | Trivial — issue a new cap from the contract account |

## Contract side: define entitlements and admin resource

```cadence
import "FungibleToken"

access(all) contract TokenMinter {

    // Entitlements partition admin power into orthogonal capabilities.
    access(all) entitlement Mint
    access(all) entitlement Burn
    access(all) entitlement Config

    access(all) event TokensMinted(amount: UFix64, to: Address?)
    access(all) event TokensBurned(amount: UFix64)
    access(all) event MintCapIssued(capabilityID: UInt64, holder: Address)

    // Canonical storage path lives in the CONTRACT account, not the admin's.
    access(all) let AdminStoragePath: StoragePath

    access(self) var totalSupply: UFix64

    access(all) resource Administrator {

        // Each method is gated by ONE entitlement. The capability holder
        // gets only the methods their entitlement covers.
        access(Mint) fun mint(amount: UFix64, recipient: Address?): @{FungibleToken.Vault} {
            pre { amount > 0.0: "Cannot mint zero" }
            TokenMinter.totalSupply = TokenMinter.totalSupply + amount
            emit TokensMinted(amount: amount, to: recipient)
            // ... build and return vault
            return <- create Vault(balance: amount)
        }

        access(Burn) fun burn(vault: @{FungibleToken.Vault}) {
            let amount = vault.balance
            TokenMinter.totalSupply = TokenMinter.totalSupply - amount
            destroy vault
            emit TokensBurned(amount: amount)
        }

        access(Config) fun pauseMinting() { /* ... */ }
    }

    init() {
        self.AdminStoragePath = /storage/TokenMinterAdmin
        self.totalSupply = 0.0
        // Singleton admin: created once, lives in the contract account forever.
        self.account.storage.save(<- create Administrator(), to: self.AdminStoragePath)
    }
}
```

Key invariants:

- The `Administrator` resource is never moved out of the contract account.
- Each privileged method uses `access(Entitlement)` — owned access is gated only when borrowed through a reference (see `entitlements.md`, "Owned values are fully entitled").
- The contract emits an event when a capability is handed out, so issuance is auditable on-chain.

## Lifecycle: mint, publish, and revoke an admin capability

### 1. Mint (at deploy or via a `setupAdmin` transaction)

The `Administrator` resource is created in `init()`. New capabilities are issued from the contract account by a function or transaction that itself requires authority:

```cadence
// Inside the contract — only callable with auth(StorageCapabilities) &Account
access(all) fun issueMintCapability(deployer: auth(StorageCapabilities) &Account)
    : Capability<auth(Mint) &Administrator> {
    let cap = deployer.capabilities.storage
        .issue<auth(Mint) &Administrator>(self.AdminStoragePath)
    emit MintCapIssued(capabilityID: cap.id, holder: deployer.address)
    return cap
}
```

### 2. Hand the capability to the new admin

Use Flow's account inbox (preferred — pull model, no shared storage):

```cadence
// Transaction signed by the contract account
transaction(newAdmin: Address) {
    prepare(signer: auth(StorageCapabilities, PublishInboxCapability) &Account) {
        let cap = signer.capabilities.storage
            .issue<auth(TokenMinter.Mint) &TokenMinter.Administrator>(
                TokenMinter.AdminStoragePath
            )
        signer.inbox.publish(cap, name: "TokenMinterMintAdmin", recipient: newAdmin)
    }
}

// Transaction signed by the new admin — claims and saves
transaction(provider: Address) {
    prepare(signer: auth(SaveValue, ClaimInboxCapability) &Account) {
        let cap = signer.inbox.claim<auth(TokenMinter.Mint) &TokenMinter.Administrator>(
            "TokenMinterMintAdmin",
            provider: provider
        ) ?? panic("No mint cap pending")
        signer.storage.save(cap, to: /storage/TokenMinterMintCap)
    }
}
```

Alternative: store the capability at a **private-by-convention** path in the contract account and hand the controller ID to the admin out-of-band. Inbox is generally preferred.

### 3. Use the capability

```cadence
transaction(amount: UFix64) {
    prepare(signer: auth(BorrowValue) &Account) {
        let cap = signer.storage.copy<Capability<auth(TokenMinter.Mint) &TokenMinter.Administrator>>(
            from: /storage/TokenMinterMintCap
        ) ?? panic("No mint cap stored")
        let adminRef = cap.borrow() ?? panic("Mint cap revoked or stale")
        let minted <- adminRef.mint(amount: amount, recipient: signer.address)
        // deposit minted into signer's vault ...
        destroy minted
    }
}
```

### 4. Revoke

Revocation happens on the **issuing account** (the contract account), not the holder's:

```cadence
transaction(capabilityID: UInt64) {
    prepare(signer: auth(StorageCapabilities) &Account) {
        let ctrl = signer.capabilities.storage
            .getController(byCapabilityID: capabilityID)
            ?? panic("Unknown capability ID")
        ctrl.delete()  // ALL copies of this capability become invalid immediately
    }
}
```

If the capability was published at a public/private path (legacy publish-style flow), `account.capabilities.unpublish(/public/...)` removes the published handle but does **not** invalidate copies — only `controller.delete()` does that. Always track and delete controllers for true revocation.

### 5. Multi-admin / multi-sig

Issue one distinct capability per admin address:

```cadence
// Mint admin lives at Alice's account
let aliceMintCap = contractAccount.capabilities.storage
    .issue<auth(Mint) &Administrator>(TokenMinter.AdminStoragePath)

// Burn admin lives at Bob's account — separate capability, separate controller
let bobBurnCap = contractAccount.capabilities.storage
    .issue<auth(Burn) &Administrator>(TokenMinter.AdminStoragePath)
```

Each cap has its own controller ID, so revoking Alice does not affect Bob. Multi-sig is achieved by handing the same entitled cap to a multi-sig account, or by gating the admin method on a conjunction (e.g. `access(Mint, Cosign) fun mint(...)`).

## Anti-pattern 1: passing `&Account` as the admin token

```cadence
// ❌ NEVER DO THIS
access(all) fun adminMint(adminAccount: auth(Storage) &Account, amount: UFix64) {
    // anyone who has THIS reference can save/load/borrow ANYTHING in the account,
    // not just call adminMint.
}
```

`auth(Storage) &Account` grants `save`, `load`, `borrow`, and `copy` on every storage path in the account. It is a master key. Capability holders should never need it for routine admin work. If you find yourself accepting an `&Account` parameter for an admin operation, replace it with a typed capability.

The same applies to legacy `AuthAccount` (Cadence < 1.0). Any code still typed as `AuthAccount` must be migrated to `auth(...) &Account` with the minimal entitlement set, then ideally replaced entirely by an entitled capability over a resource.

## Anti-pattern 2: address-flag admin lists

```cadence
// ❌ WRONG: contract stores an admin allowlist and checks tx.signer.address
access(all) contract BadToken {
    access(self) var admins: {Address: Bool}

    access(all) fun mint(signer: auth(BorrowValue) &Account, amount: UFix64) {
        pre { self.admins[signer.address] == true: "Not admin" }
        // ...
    }
}
```

Problems:

- Adding or removing an admin requires either a contract upgrade or a privileged
  function that itself needs gating — recursive bootstrapping problem.
- Anyone who compromises an admin address gets full admin powers; there is no
  way to scope to "mint only" vs "burn only" without inventing a parallel
  entitlement system in storage.
- `tx.signer.address` is set by the signer; a contract called from another
  contract sees the caller's address, not the original initiator. Relying on
  it conflates authorization with identity.
- No on-chain audit trail of which admin called what — all calls come from
  whichever address holds the flag.

Capabilities fix every one of these: scoped (entitlement), revocable (controller delete), unforgeable (type system), individually addressable (capability ID), and never require a contract upgrade to rotate.

## Migration from a legacy `Admin` resource

Legacy pattern:

```cadence
// ❌ Legacy — admin tied to deployer's keys
init() {
    let admin <- create Admin()
    self.account.storage.save(<-admin, to: /storage/admin)
}
// Admin methods accessed via:
let adminRef = signer.storage.borrow<&Admin>(from: /storage/admin) ?? panic(...)
```

If the deployer's key is rotated or lost, the admin resource is stranded.

### Step-by-step migration

1. **Add entitlements without removing existing access.** In a contract upgrade, declare `entitlement Mint`, `entitlement Burn`, etc., and re-mark admin methods as `access(Mint)` (or whichever entitlement applies). Owned-value access still works inside the deployer account, so existing scripts/transactions that borrow `&Admin` directly continue to function.
2. **Issue capabilities from the contract account.** Sign a transaction with the contract account that calls `account.capabilities.storage.issue<auth(Mint) &Administrator>(/storage/admin)` and publishes the result to the new admin via inbox.
3. **New admin claims and stores the capability.** The new admin's transactions now borrow through the capability instead of the deployer's storage.
4. **Audit all existing controllers.** Run `account.capabilities.storage.getControllers(forPath: /storage/admin)` and delete any stale controllers issued before the migration. Tag remaining controllers with `setTag` so each future audit can identify the holder.
5. **Remove the legacy access path.** If any transaction code path still borrows `&Admin` directly without going through a capability, update it. The underlying resource stays in the contract account; nothing is moved.

The migration never requires moving the `Administrator` resource — only re-issuing capabilities pointing at it. This is why the modern pattern is robust to key rotation: the resource lives where the *contract* lives, and admin identity is decoupled from account identity.

## Worked example: split Mint / Burn admin

```cadence
import "TokenMinter"

// Tx 1 (signed by contract account): hand Mint cap to Alice
transaction(alice: Address) {
    prepare(signer: auth(StorageCapabilities, PublishInboxCapability) &Account) {
        let cap = signer.capabilities.storage
            .issue<auth(TokenMinter.Mint) &TokenMinter.Administrator>(
                TokenMinter.AdminStoragePath
            )
        signer.inbox.publish(cap, name: "MintAdmin", recipient: alice)
    }
}

// Tx 2 (signed by contract account): hand Burn cap to Bob
transaction(bob: Address) {
    prepare(signer: auth(StorageCapabilities, PublishInboxCapability) &Account) {
        let cap = signer.capabilities.storage
            .issue<auth(TokenMinter.Burn) &TokenMinter.Administrator>(
                TokenMinter.AdminStoragePath
            )
        signer.inbox.publish(cap, name: "BurnAdmin", recipient: bob)
    }
}
```

Even though both capabilities point at the **same** `Administrator` resource:

- Alice's cap is typed `Capability<auth(Mint) &Administrator>`. Borrowing it yields an `auth(Mint) &Administrator` reference. Calling `.burn(...)` fails to type-check — the reference does not carry the `Burn` entitlement.
- Bob's cap is typed `Capability<auth(Burn) &Administrator>`. Calling `.mint(...)` through it fails to type-check.
- If Alice forwards her cap to Charlie, Charlie also gets only `Mint` access. The cap's type travels with the value.
- Revoking Alice (delete her controller) does not affect Bob (different controller ID, different cap), and vice versa.

This is the security property that makes capabilities strictly more expressive than role flags: **the authority is in the reference type, not in the caller's identity.**

## Common pitfalls

- **Publishing entitled capabilities publicly.** `account.capabilities.publish(mintCap, at: /public/admin)` is catastrophic — anyone can borrow it. Entitled caps are passed via inbox or stored privately by the holder; only un-entitled `&Vault`-style caps belong at `/public/`.
- **Storing capabilities in a public field.** `access(all) let adminCapability: Capability<...>` exposes the cap to anyone scripting against the contract. Capability-typed fields must be `access(self)` (see `access-control.md` Rule 4).
- **Calling `unpublish` and assuming revocation.** `unpublish` removes the published handle but existing copies of the cap remain valid. Use `controller.delete()` to truly revoke.
- **Forgetting to track controller IDs.** Without the ID you cannot revoke a specific cap. Save the ID at issuance time and tag the controller with a human-readable description.
- **Issuing one mega-entitled cap and reusing it.** `auth(Mint, Burn, Config) &Administrator` defeats the point. One cap per entitlement, one entitlement per role.
- **Putting the admin resource in the admin's storage.** The admin should hold the *capability*, not the resource. The resource lives in the contract account so it can be re-issued to new admins after a key rotation.
- **Trusting `tx.signer.address` as proof of admin.** Capability presence is the proof. Treat the signer as just "someone with a transaction" and let the borrow succeed or fail.

## See also

- [capabilities.md](capabilities.md) — capability lifecycle, controllers, retargeting
- [entitlements.md](entitlements.md) — entitlement sets, mappings, owned-value rule
- [access-control.md](access-control.md) — access modifier hierarchy and rules
- [design-patterns.md](design-patterns.md) — Pattern 4 "Singleton Admin Resource"
- [strategy-registry.md](strategy-registry.md) — strategy registries are built on this admin-via-capability pattern, with each registered strategy gated by an entitled capability issued from the registry contract account
