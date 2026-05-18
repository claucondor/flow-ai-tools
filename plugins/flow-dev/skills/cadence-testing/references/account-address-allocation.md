# Account Address Allocation in `flow test`

The test framework allocates addresses sequentially. System contracts occupy a fixed range at
the low end of the address space; every call to `Test.createAccount()` hands out the next
address after those. On Flow CLI v2.17.1, the system reserves addresses `0x1` through `0x19`
(25 addresses), so the first `Test.createAccount()` call returns `0x000000000000001a`
(decimal 26), not `0x5` or `0x6` as older documentation stated.

If a project's `flow.json` assigns a contract a specific `testing` alias address, the alias
must be `0x1a` or higher. Addresses below `0x1a` are system-reserved and cannot be reached
via `Test.createAccount()`. If the alias is set to a value between `0x1a` and the address the
counter has reached so far, the test must call `Test.createAccount()` enough times to push the
counter to that address — otherwise no account exists at that address and the deployment fails
with `account with address ... not found`.

Understanding the allocation sequence lets you plan `flow.json` testing aliases upfront to
minimize filler accounts.

---

## System Contract Address Range (v2.17.1)

The in-process test framework bootstraps the following system accounts on the
`flow-emulator-monotonic` chain. All 25 of these addresses are consumed before
`Test.createAccount()` is ever called:

| Range | Notes |
|---|---|
| `0x0000000000000001` | Service account; also `NonFungibleToken`, `MetadataViews`, `ViewResolver` |
| `0x0000000000000002` | `FungibleToken` |
| `0x0000000000000003` | `FlowToken` |
| `0x0000000000000004` | `FlowFees` |
| `0x0000000000000005` – `0x0000000000000019` | Additional system infrastructure accounts (consensus, EVM, storage fees, service account contracts, and other protocol-level accounts) |

Verified on CLI v2.17.1: `Test.createAccount()` returns `0x000000000000001a` (decimal 26).
The system reserves `0x1` through `0x19` (25 addresses). Prior documentation stating the user
range starts at `0x5` or `0x6` reflects a much older CLI version where far fewer system
accounts were provisioned.

---

## Sequential Allocation Example

Each `Test.createAccount()` call increments the counter by one. Accounts at file-level scope
are created before `setup()` runs; accounts created inside `setup()` or test functions are
created when that code executes.

```cadence
import Test

// These execute in declaration order, before setup().
access(all) let alice = Test.createAccount()  // 0x000000000000001a
access(all) let bob   = Test.createAccount()  // 0x000000000000001b
access(all) let carol = Test.createAccount()  // 0x000000000000001c
```

After the three file-level accounts are created, any contracts deployed via `Test.deployContract`
with `testing` aliases at `0x1a`, `0x1b`, or `0x1c` can match those accounts:

```json
{
  "contracts": {
    "MyContract": {
      "source": "cadence/contracts/MyContract.cdc",
      "aliases": {
        "testing": "000000000000001a"
      }
    }
  }
}
```

---

## Reaching a Higher Address — Filler Accounts

If a project's `flow.json` was written assuming a specific testing alias above `0x1a` (for
example `0x000000000000001e`), the test must create enough accounts to push the counter to
that address.

Starting from the first user address `0x1a` (decimal 26), reaching `0x1e` (decimal 30)
requires creating 5 accounts:

```cadence
// 0x1a → 0x1b → 0x1c → 0x1d → 0x1e = 5 accounts needed

access(all) let _filler1  = Test.createAccount()   // 0x000000000000001a
access(all) let _filler2  = Test.createAccount()   // 0x000000000000001b
access(all) let _filler3  = Test.createAccount()   // 0x000000000000001c
access(all) let _filler4  = Test.createAccount()   // 0x000000000000001d
access(all) let yourAccount = Test.createAccount() // 0x000000000000001e
```

The filler accounts have no purpose other than consuming addresses. They can be named with
a leading underscore to signal that they are positional placeholders.

Note: addresses below `0x1a` (such as `0x0e`, `0x07`, `0x06`) are system-reserved. There is
no way to reach those addresses via `Test.createAccount()`. If your `flow.json` testing alias
is set to a value below `0x1a`, the framework allocates that slot during the system bootstrap
before user account creation begins — not through `createAccount` calls.

---

## Preferred Pattern — Use the Lowest Available Aliases

Rather than creating filler accounts, update `flow.json` testing aliases to use the first
few addresses in the user range (`0x1a`, `0x1b`, etc.):

```json
{
  "contracts": {
    "TokenA": {
      "source": "cadence/contracts/TokenA.cdc",
      "aliases": {
        "emulator": "f8d6e0586b0a20c7",
        "testing":  "000000000000001a"
      }
    },
    "TokenB": {
      "source": "cadence/contracts/TokenB.cdc",
      "aliases": {
        "emulator": "f8d6e0586b0a20c7",
        "testing":  "000000000000001b"
      }
    }
  }
}
```

Then:

```cadence
import Test

access(all) let tokenAAdmin = Test.createAccount()  // 0x1a — matches TokenA testing alias
access(all) let tokenBAdmin = Test.createAccount()  // 0x1b — matches TokenB testing alias
```

No filler accounts needed. Multiple contracts can share a single account (for example,
pointing everything at `0x1a`) to reduce the number of accounts required further.

---

## Multiple Contracts on One Account

Multiple contracts can be deployed to the same testing address. The framework does not restrict
the number of contracts per account:

```json
{
  "contracts": {
    "Burner":                   { "aliases": { "testing": "000000000000001a" } },
    "FungibleToken":            { "aliases": { "testing": "000000000000001a" } },
    "FungibleTokenMetadataViews": { "aliases": { "testing": "000000000000001a" } },
    "ExampleToken":             { "aliases": { "testing": "000000000000001a" } }
  }
}
```

With this configuration, one `Test.createAccount()` call suffices for all four contracts.

---

## Address Counter Scope

The address counter is scoped to the lifetime of the blockchain instance — not the file. Two
test files that each call `Test.createAccount()` as their first action see the same first
address `0x1a` (because each file runs against a fresh blockchain instance started by the test
runner). Within a single file, the counter is global across all lifecycle functions:
file-level bindings, `setup()`, `beforeEach()`, and test functions all share the same
counter.

`Test.reset(to: height)` does NOT reset the address counter. Addresses created after the
reset target height are pruned from the blockchain's state but the counter does not rewind.
Subsequent `createAccount()` calls continue from the next unused index. This means accounts
created after a `Test.reset()` will have higher addresses than accounts created before the
reset, even though their storage has been wiped. See
[`blockchain-emulation.md`](blockchain-emulation.md) for the related "never reset to a height
before account creation" warning.

---

## Quick Reference

```
System accounts (pre-allocated, cannot be created or overridden):
  0x1  through 0x19 — 25 system addresses (service account, token standards,
                       consensus infrastructure, EVM, protocol-level accounts)
  Verified on CLI v2.17.1.

First user account from Test.createAccount():
  0x000000000000001a  (decimal 26)

User range for flow.json testing aliases:
  0x1a and above
```

---

## Common Pitfalls

- **`account with address ... not found` during deployment** — The testing alias address in
  `flow.json` does not correspond to any account that `Test.createAccount()` has allocated.
  Either add filler accounts or lower the alias to the first available user address (`0x1a`
  or higher).
- **Setting a testing alias below `0x1a`** — Addresses `0x1` through `0x19` are system
  addresses. You cannot reach them via `Test.createAccount()`. If your alias is in this range,
  you need to understand that the framework occupies those slots during system bootstrap — they
  are not reachable for your project deployments via the normal user account path.
- **Hard-coding addresses in test assertions** — Addresses are determined by the allocation
  order across the file's lifetime. If you add a new file-level account above an existing one,
  every subsequent account shifts by one. Read addresses from the `TestAccount.address` field
  rather than hard-coding hex literals in assertions.
- **Resetting past account creation** — If `Test.reset(to: height)` rewinds past the block
  where an account was created, signing transactions with that account after the reset
  produces `account public key not found`. Ensure the reset target is at or after all
  account creations. See [`blockchain-emulation.md`](blockchain-emulation.md).
- **Assuming the first `createAccount()` address is stable across CLI versions** — The
  starting address depends on how many system accounts the framework provisions, which has
  changed across CLI versions. On v2.17.1 it is `0x1a`. Pin your testing aliases to specific
  addresses and verify them when upgrading the Flow CLI.

---

## See Also

- [`setup-and-basics.md`](setup-and-basics.md) — The `testing` alias mechanism in `flow.json`,
  the two-category split between auto-loading dependency contracts and manually-deployed project
  contracts, and the address range for user contracts
- [`blockchain-emulation.md`](blockchain-emulation.md) — `Test.createAccount()` semantics,
  the service account, and `Test.reset()` interaction with account state
- [`system-contracts-availability.md`](system-contracts-availability.md) — Full list of
  pre-allocated system contract addresses in the test framework
