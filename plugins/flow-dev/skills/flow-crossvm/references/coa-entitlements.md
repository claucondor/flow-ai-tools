# COA Entitlement Model Deep-Dive

A Cadence Owned Account (`@EVM.CadenceOwnedAccount`) is not a passive data container —
it is a bearer-authority primitive whose reference type determines what the holder can do
across the VM boundary. Entitlement scoping matters more for COAs than for ordinary Cadence
resources because every privileged method ultimately reaches real EVM state: calls can
drain ERC20 balances, deploys can publish malicious contracts, and withdrawals move native
FLOW out of the EVM side with no confirmation step. The blast radius of an over-entitled
capability is therefore the full EVM portfolio of the COA's address, not just a Cadence
vault. This reference catalogs every entitlement, the exact methods it gates, and the
minimum-privilege patterns an auditor should enforce.

For COA creation, canonical storage paths, and the lifecycle pattern, see
[coa-lifecycle.md](coa-lifecycle.md). For `coa.call` mechanics, calldata encoding, and
`EVM.Result` handling, see [evm-call.md](evm-call.md). For the native FLOW bridge, see
[flow-bridge.md](flow-bridge.md).

---

## Entitlement Catalog

The EVM contract (deployed to the Flow service account) declares six entitlements on
`EVM.CadenceOwnedAccount`. The source is
`fvm/evm/stdlib/contract.cdc` in the `flow-go` repository; all method annotations below
are verified against that source.

```
access(all) entitlement Validate   // prove ownership without exercising call or fund authority
access(all) entitlement Withdraw   // bridge FLOW from EVM back to Cadence
access(all) entitlement Call       // send state-mutating EVM transactions
access(all) entitlement Deploy     // deploy new EVM contracts
access(all) entitlement Owner      // top-level — accepted wherever any of the above is accepted
access(all) entitlement Bridge     // bridge wrapped NFTs and fungible tokens via the bridge router
```

`EVM.Bridge` was not present in earlier documentation of the COA surface; it covers the
cross-VM token bridge and is distinct from the native FLOW bridge (`Withdraw`).

---

## Privilege Ladder

`EVM.Owner` is **not** a Cadence entitlement mapping aggregate. It is a plain entitlement
that the contract uses in disjunctive access sets. Every method that is gated by a
non-`Owner` entitlement also accepts `Owner` via `access(Owner | X)`. This means:

- A reference typed `auth(EVM.Owner) &EVM.CadenceOwnedAccount` can call every method on the
  resource, including those gated by `Call`, `Deploy`, `Withdraw`, `Validate`, and `Bridge`.
- A reference typed `auth(EVM.Call) &EVM.CadenceOwnedAccount` can only call methods whose
  access annotation includes `Call`. It cannot call `withdraw`, `deploy`, `protectedAddress`,
  or `withdrawNFT` / `withdrawTokens`.

```
EVM.Owner  (accepted wherever any other entitlement is accepted — via disjunctions)
  |
  +-- accepted by: access(Owner | Call)
  |     coa.call(...)
  |     coa.callWithSigAndArgs(...)
  |
  +-- accepted by: access(Owner | Deploy)
  |     coa.deploy(...)
  |
  +-- accepted by: access(Owner | Withdraw)
  |     coa.withdraw(balance:)
  |
  +-- accepted by: access(Owner | Validate)
  |     coa.protectedAddress()
  |
  +-- accepted by: access(Owner | Bridge)
        coa.withdrawNFT(type:id:feeProvider:)
        coa.withdrawTokens(type:amount:feeProvider:)
```

`EVM.Call`, `EVM.Deploy`, `EVM.Withdraw`, `EVM.Validate`, and `EVM.Bridge` are mutually
independent. Holding `EVM.Call` does NOT imply `EVM.Withdraw`; holding `EVM.Bridge` does
NOT imply `EVM.Call`. The only entitlement that subsumes the others is `EVM.Owner`.

---

## Un-entitled Methods (no entitlement required)

These methods are `access(all)` and usable with any reference shape, including a bare
`&EVM.CadenceOwnedAccount`:

| Method | Signature | Effect |
|--------|-----------|--------|
| `address()` | `view fun address(): EVM.EVMAddress` | Read the COA's EVM address |
| `balance()` | `view fun balance(): EVM.Balance` | Read the EVM-side FLOW balance |
| `deposit(from:)` | `fun deposit(from: @FlowToken.Vault)` | Move FLOW from Cadence into EVM (irreversible without `Withdraw`) |
| `dryCall(to:data:gasLimit:value:)` | `fun dryCall(...): EVM.Result` | Simulate a call as the COA's EVM address; no state committed |
| `dryCallWithSigAndArgs(...)` | `fun dryCallWithSigAndArgs(...): EVM.ResultDecoded` | Same as `dryCall` but accepts a signature string and args |
| `depositNFT(nft:feeProvider:)` | `fun depositNFT(...)` | Bridge an NFT from Cadence to the COA's EVM address via the bridge router |
| `depositTokens(vault:feeProvider:)` | `fun depositTokens(...)` | Bridge a fungible token vault from Cadence to EVM |

`deposit`, `depositNFT`, and `depositTokens` are un-entitled by design: adding value to an
account is always safe. The inverse operations — withdrawing value out — are gated.

---

## Per-Entitlement Reference

### EVM.Call

**Access annotation on COA methods**: `access(Owner | Call)`

**Gated methods**:

| Method | Description |
|--------|-------------|
| `call(to:data:gasLimit:value:): EVM.Result` | Execute a state-mutating EVM transaction. Sends arbitrary calldata; the COA's EVM address is `msg.sender`. |
| `callWithSigAndArgs(to:signature:args:gasLimit:value:resultTypes:): EVM.ResultDecoded` | Convenience wrapper: derives the 4-byte selector from `signature`, ABI-encodes `args`, then performs the same EVM call. |

**State access**: Reads and writes EVM state. The call commits to EVM state unless it
reverts; the Cadence transaction commits regardless unless you explicitly `panic` on
`result.status != EVM.Status.successful`.

**Can it withdraw FLOW?** `EVM.Call` itself does not expose a FLOW withdrawal path to
Cadence. However, passing `value: EVM.Balance(attoflow: N)` in a `coa.call` transfers
native FLOW to an EVM contract's balance. That FLOW is then under the EVM contract's
control. The reverse path — bringing FLOW back to a Cadence vault — requires `EVM.Withdraw`.

**Implied by `EVM.Owner`?** Yes: any `auth(EVM.Owner)` reference can call these methods.

**If caller lacks this entitlement**: Cadence emits a type-check error at the capability
borrow site or at the direct downcast. The call does not proceed. This is a compile-time
or runtime type error, not an EVM error.

**Common errors**:
- Borrowing with `&EVM.CadenceOwnedAccount` (no `auth`) then calling `coa.call(...)` —
  compile-time error: `access denied: cannot access 'call' because function requires 'Owner | Call' authorization, but reference is unauthorized`.
- Forgetting to check `result.status` — silent EVM failure; the Cadence tx commits while
  EVM state is rolled back.

---

### EVM.Deploy

**Access annotation**: `access(Owner | Deploy)`

**Gated method**:

| Method | Description |
|--------|-------------|
| `deploy(code:gasLimit:value:): EVM.Result` | Deploy EVM bytecode. The COA's EVM address is `msg.sender` and `tx.origin`. Returns a `Result` whose `deployedContract` field contains the new contract's `EVM.EVMAddress` if `status == successful`. |

**Is it a subset of Call?** No — independent of `EVM.Call`. Issue both when needed:
`auth(EVM.Call, EVM.Deploy) &EVM.CadenceOwnedAccount`.

**State access**: Writes EVM state (publishes bytecode at a new address).

**Can it withdraw FLOW?** Passing `value` to `deploy` sends FLOW to the new contract's EVM
address; recovering it to Cadence still requires `EVM.Withdraw`.

**Implied by `EVM.Owner`?** Yes.

**Common errors**:
- `deploy` returns `EVM.Status.failed` if the constructor reverts or gas is too low.
- `result.deployedContract` is `nil` on failure; force-unwrapping panics.

---

### EVM.Withdraw

**Access annotation**: `access(Owner | Withdraw)`

**Gated method**:

| Method | Description |
|--------|-------------|
| `withdraw(balance: EVM.Balance): @FlowToken.Vault` | Decrements the COA's EVM balance by `balance.attoflow` and returns a `@FlowToken.Vault` containing the equivalent FLOW. The vault is a native Cadence resource — no EVM transaction is executed; this is a direct FVM balance move. |

**Does `EVM.Withdraw` imply `EVM.Call`?** No. A reference with only `EVM.Withdraw` can
withdraw FLOW to Cadence but cannot call EVM contracts. This distinction matters: a
"refund-only" escrow role needs `auth(EVM.Withdraw)`, not `auth(EVM.Call, EVM.Withdraw)`.

**State access**: Writes both EVM state (decrements EVM balance) and Cadence state
(returns a live vault). It errors if:
- The requested amount exceeds the COA's EVM balance — produces an EVM runtime error
  (Error Code 1300), not a Cadence panic.
- The requested amount is smaller than `1e10 attoflow` (the minimum representable in
  `UFix64`), producing a Cadence panic: `"withdraw failed! smallest unit allowed to transfer is 1e10 attoFlow"`.

**Implied by `EVM.Owner`?** Yes.

**Common errors**:
- Insufficient balance: EVM runtime error (not a Cadence panic): `"evm runtime error: insufficient funds for gas * price + value: address <EVM_ADDRESS> have <N> want <M>"` (Error Code 1300). The error originates in `InternalEVM.withdraw` and surfaces as a Cadence runtime error wrapping the EVM execution failure.
- Sub-`1e10` attoflow amount: Cadence panic with the exact string above (verified in
  `contract.cdc` doc comment).
- Missing entitlement: compile or runtime type error, same pattern as `EVM.Call`.

---

### EVM.Validate

**Access annotation**: `access(Owner | Validate)`

**Gated method**:

| Method | Description |
|--------|-------------|
| `protectedAddress(): EVM.EVMAddress` | Returns the COA's EVM address, identical to `address()`, but gated by `Validate` or `Owner`. |

**What is `EVM.Validate` for?** It provides a Cadence-native proof-of-ownership mechanism:
a verifier asks the caller to present `auth(EVM.Validate) &EVM.CadenceOwnedAccount` and
call `protectedAddress()`, demonstrating control of the COA without granting call or
withdrawal authority.

`EVM.validateCOAOwnershipProof(...)` is a separate top-level function that uses a public
path and cryptographic signatures; it does not require `Validate`. The `Validate`
entitlement is for on-chain identity flows only.

Confirmed: `EVM.Validate` is NOT used in EIP-712 or ERC-1271 flows. Those are handled at the EVM/Solidity layer by the COA smart contract wallet. `EVM.Validate` gates only `protectedAddress()` for Cadence-side identity proofs. `EVM.validateCOAOwnershipProof(...)` is a separate `access(all)` top-level function and does not require `EVM.Validate`.

**State access**: Read-only (`protectedAddress()` is `view`).

**Implied by `EVM.Owner`?** Yes.

---

### EVM.Bridge

**Access annotation**: `access(Owner | Bridge)`

**Gated methods**:

| Method | Description |
|--------|-------------|
| `withdrawNFT(type:id:feeProvider:): @{NonFungibleToken.NFT}` | Withdraw a wrapped NFT from the COA's EVM address back to Cadence via the `onflow/flow-evm-bridge` router. Internally calls the bridge accessor with `auth(EVM.Call) &CadenceOwnedAccount`. |
| `withdrawTokens(type:amount:feeProvider:): @{FungibleToken.Vault}` | Withdraw wrapped fungible tokens from the COA's EVM address back to Cadence via the bridge router. Also internally calls the bridge accessor with `EVM.Call`. |

**Relationship to `EVM.Call`**: The bridge methods use `EVM.Call` internally when calling
the bridge accessor (`&self as auth(Call) &CadenceOwnedAccount`). However, an
`auth(EVM.Bridge)` reference cannot call `coa.call()` directly; `EVM.Bridge` does not imply
`EVM.Call` from the caller's perspective.

**State access**: Reads and writes both Cadence and EVM state; fee provider vault is consumed.

**Implied by `EVM.Owner`?** Yes.

**Note**: `depositNFT` and `depositTokens` are `access(all)` — no entitlement required to
deposit. Only the withdrawal direction requires `EVM.Bridge`.

---

### EVM.Owner

**Access annotation**: every COA method that has a non-`all` access annotation accepts
`Owner` as the disjunct: `access(Owner | Call)`, `access(Owner | Deploy)`,
`access(Owner | Withdraw)`, `access(Owner | Validate)`, `access(Owner | Bridge)`.

**What does `EVM.Owner` gate uniquely?** Nothing — it is a "pass-everywhere" ticket
that saves you from listing all five entitlements separately. Its value is as a signal:
"full custody". Never issue `EVM.Owner` to a third-party consumer or escrow; use the
narrow entitlement set.

**Hierarchy summary**:

```
Entitlement    Gated by Owner?    Independent (not implied by others)?
-------------------------------------------------------------------
EVM.Call       Yes                Yes — does not imply Deploy/Withdraw/Validate/Bridge
EVM.Deploy     Yes                Yes
EVM.Withdraw   Yes                Yes
EVM.Validate   Yes                Yes
EVM.Bridge     Yes                Yes
EVM.Owner      N/A                Yes — accepted in all disjunctions
```

---

## Gated Methods Master Table

| Method | Entitlement | Notes |
|--------|-------------|-------|
| `coa.call(to:data:gasLimit:value:)` | `EVM.Call` or `EVM.Owner` | State-mutating EVM call |
| `coa.callWithSigAndArgs(to:signature:args:gasLimit:value:resultTypes:)` | `EVM.Call` or `EVM.Owner` | Same semantics; builds calldata from signature string |
| `coa.deploy(code:gasLimit:value:)` | `EVM.Deploy` or `EVM.Owner` | Deploys EVM contract |
| `coa.withdraw(balance:)` | `EVM.Withdraw` or `EVM.Owner` | Moves FLOW from EVM to Cadence vault |
| `coa.protectedAddress()` | `EVM.Validate` or `EVM.Owner` | Read-only identity proof |
| `coa.withdrawNFT(type:id:feeProvider:)` | `EVM.Bridge` or `EVM.Owner` | Bridge NFT from EVM to Cadence |
| `coa.withdrawTokens(type:amount:feeProvider:)` | `EVM.Bridge` or `EVM.Owner` | Bridge FT from EVM to Cadence |
| `coa.address()` | None (`access(all) view`) | Read EVM address |
| `coa.balance()` | None (`access(all) view`) | Read EVM FLOW balance |
| `coa.deposit(from:)` | None (`access(all)`) | Move FLOW from Cadence into EVM |
| `coa.dryCall(to:data:gasLimit:value:)` | None (`access(all)`) | Simulate call; no commit |
| `coa.dryCallWithSigAndArgs(...)` | None (`access(all)`) | Simulate call with signature; no commit |
| `coa.depositNFT(nft:feeProvider:)` | None (`access(all)`) | Bridge NFT into EVM |
| `coa.depositTokens(vault:feeProvider:)` | None (`access(all)`) | Bridge FT into EVM |

---

## Minimum-Privilege Patterns

### Read EVM state without a COA

```cadence
// ✅ No COA needed. EVM.dryCall is access(all) — no entitlement.
let r = EVM.dryCall(from: zero, to: token, data: cd,
                    gasLimit: 50_000, value: EVM.Balance(attoflow: 0))
assert(r.status == EVM.Status.successful, message: r.errorMessage)
```

Full script example in [evm-call.md](evm-call.md) § "View call".

### Call an EVM contract that mutates state

```cadence
// ✅ EVM.Call only — minimum required for coa.call.
let coa = signer.storage.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(
    from: /storage/evm
) ?? panic("no COA")
let result = coa.call(to: target, data: calldata, gasLimit: 100_000,
                      value: EVM.Balance(attoflow: 0))
assert(result.status == EVM.Status.successful, message: result.errorMessage)
```

```cadence
// ❌ Over-privileged — grants deploy, withdraw, and validate unnecessarily.
let coa = signer.storage.borrow<auth(EVM.Owner) &EVM.CadenceOwnedAccount>(
    from: /storage/evm
) ?? panic("no COA")
```

### Deploy a new EVM contract

```cadence
// ✅ EVM.Deploy only.
let coa = signer.storage.borrow<auth(EVM.Deploy) &EVM.CadenceOwnedAccount>(
    from: /storage/evm
) ?? panic("no COA")
let result = coa.deploy(code: bytecode, gasLimit: 2_000_000,
                        value: EVM.Balance(attoflow: 0))
assert(result.status == EVM.Status.successful, message: result.errorMessage)
let addr = result.deployedContract ?? panic("deploy returned nil address")
```

### Deposit FLOW then call an EVM contract (with recovery path)

```cadence
// ✅ Both EVM.Call (to call) and EVM.Withdraw (to recover if call fails).
//    Deposit itself needs no entitlement; the borrow is still useful to do once.
let coa = signer.storage.borrow<auth(EVM.Call, EVM.Withdraw) &EVM.CadenceOwnedAccount>(
    from: /storage/evm
) ?? panic("no COA")
coa.deposit(from: <-flowVault)
let result = coa.call(to: target, data: cd, gasLimit: 500_000,
                      value: EVM.Balance(attoflow: depositAttoflow))
if result.status != EVM.Status.successful {
    // Panic reverts the deposit too.
    panic("EVM call failed: ".concat(result.errorMessage))
}
```

### Prove COA identity on-chain without granting call authority

```cadence
// ✅ EVM.Validate — read-only ownership proof; cannot call or withdraw.
let coa = signer.storage.borrow<auth(EVM.Validate) &EVM.CadenceOwnedAccount>(
    from: /storage/evm
) ?? panic("no COA")
let evmAddr = coa.protectedAddress()
```

### Bridge a wrapped NFT back to Cadence

```cadence
// ✅ EVM.Bridge — does not grant EVM.Call to the caller.
let coa = signer.storage.borrow<auth(EVM.Bridge) &EVM.CadenceOwnedAccount>(
    from: /storage/evm
) ?? panic("no COA")
let nft <- coa.withdrawNFT(type: Type<@MyNFT.NFT>(), id: 42, feeProvider: feeVault)
```

### Full admin (single-account owner only)

```cadence
// ✅ EVM.Owner — appropriate only for the account owner; never share or delegate.
let coa = signer.storage.borrow<auth(EVM.Owner) &EVM.CadenceOwnedAccount>(
    from: /storage/evm
) ?? panic("no COA")
```

---

## Capability Issue and Borrow Patterns

### Issuing a narrow capability

```cadence
// Each .issue() creates an independent controller with its own ID.
// Call-only (router):
let routerCap = signer.capabilities.storage
    .issue<auth(EVM.Call) &EVM.CadenceOwnedAccount>(/storage/evm)

// Call + Withdraw (escrow with refund path):
let escrowCap = signer.capabilities.storage
    .issue<auth(EVM.Call, EVM.Withdraw) &EVM.CadenceOwnedAccount>(/storage/evm)

// Deploy-only (factory):
let factoryCap = signer.capabilities.storage
    .issue<auth(EVM.Deploy) &EVM.CadenceOwnedAccount>(/storage/evm)

// Store the controller ID for later revocation:
let escrowControllerID: UInt64 = escrowCap.id
```

### Borrowing a capability

```cadence
// From a held Capability<...>:
let coa = escrowCap.borrow() ?? panic("COA capability revoked")

// Direct storage borrow (signer owns the COA):
let coa = signer.storage.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(
    from: /storage/evm
) ?? panic("no COA at /storage/evm")
```

### Revoking a capability

```cadence
// Delete the controller; all subsequent .borrow() calls on the issued cap return nil.
for c in signer.capabilities.storage.getControllers(forPath: /storage/evm) {
    if c.capabilityID == controllerID { c.delete(); break }
}
```

Full revocation transaction pattern in [coa-lifecycle.md](coa-lifecycle.md) §
"Issuing and revoking the COA capability".

---

## Common Errors Per Entitlement

| Scenario | Error form | Exact message (or pattern) |
|----------|-----------|---------------------------|
| Borrow `&EVM.CadenceOwnedAccount` (no auth) then call `coa.call(...)` | Cadence compile-time error | `access denied: cannot access 'call' because function requires 'Owner \| Call' authorization, but reference is unauthorized` |
| Borrow `auth(EVM.Call)` then call `coa.withdraw(...)` | Cadence compile-time error | `access denied: cannot access 'withdraw' because function requires 'Owner \| Withdraw' authorization, but reference only has 'Call' authorization` |
| Borrow `auth(EVM.Deploy)` then call `coa.call(...)` | Cadence compile-time error | `access denied: cannot access 'call' because function requires 'Owner \| Call' authorization, but reference only has 'Deploy' authorization` |
| Call `coa.withdraw(balance:)` where balance exceeds COA's EVM balance | EVM runtime error (Error Code 1300) | `evm runtime error: insufficient funds for gas * price + value: address <EVM_ADDRESS> have <N> want <M>` — this is an EVM-layer error, not a Cadence panic |
| Call `coa.withdraw(balance:)` with amount < `1e10 attoflow` | Cadence panic | `"withdraw failed! smallest unit allowed to transfer is 1e10 attoFlow"` (doc-comment verified in `contract.cdc`) |
| Call `coa.deploy(...)` and the constructor reverts | `EVM.Result.status == .failed` | `result.errorMessage` contains the Solidity revert string if emitted |
| Call `coa.call(...)` and gas is exhausted | `EVM.Result.status == .failed` | `result.errorMessage` is `"out of gas"` or similar EVM-level message |
| Call any gated method while `EVM.isPaused() == true` | Cadence panic (pre-condition) | `"EVM operations are temporarily paused"` (verified in `contract.cdc`) |
| Force-unwrap `result.deployedContract` on a failed deploy | Cadence panic | `"unexpectedly found nil while forcing an Optional value"` |

---

## Anti-Patterns: Entitlement Bundling

Cross-link: [../../cadence-audit/references/crossvm-anti-patterns.md](../../cadence-audit/references/crossvm-anti-patterns.md)
(C4 — Sharing COA auth capabilities, C5 — Publishing auth cap at `/public/evm`).

### Anti-pattern A: granting `EVM.Owner` when only `EVM.Call` is needed

```cadence
// ❌ — grants deploy, withdraw, validate, and bridge for free
let cap = signer.capabilities.storage
    .issue<auth(EVM.Owner) &EVM.CadenceOwnedAccount>(/storage/evm)
signer.inbox.publish(cap, name: "routerCOA", recipient: routerAddr)
```

A compromised router can now call `coa.withdraw(...)` and drain every FLOW from the COA's
EVM balance in a single transaction. Replacing `EVM.Owner` with `EVM.Call` confines the
blast radius to ERC20 operations.

```cadence
// ✅ — narrow to what the router actually needs
let cap = signer.capabilities.storage
    .issue<auth(EVM.Call) &EVM.CadenceOwnedAccount>(/storage/evm)
signer.inbox.publish(cap, name: "routerCOA", recipient: routerAddr)
```

### Anti-pattern B: sharing one auth capability across multiple consumers

```cadence
// ❌ — one cap, three consumers; revoke one = revoke all
let cap = signer.capabilities.storage
    .issue<auth(EVM.Call) &EVM.CadenceOwnedAccount>(/storage/evm)
signer.inbox.publish(cap, name: "escrowCOA",  recipient: escrowAddr)
signer.inbox.publish(cap, name: "routerCOA",  recipient: routerAddr)
signer.inbox.publish(cap, name: "keeperCOA",  recipient: keeperAddr)
```

A bug in any one consumer compromises all three, and there is no per-consumer revocation.

```cadence
// ✅ — one cap per consumer; delete only keeperCap's controller to revoke just the keeper
let escrowCap = signer.capabilities.storage
    .issue<auth(EVM.Call, EVM.Withdraw) &EVM.CadenceOwnedAccount>(/storage/evm)
let routerCap = signer.capabilities.storage
    .issue<auth(EVM.Call) &EVM.CadenceOwnedAccount>(/storage/evm)
let keeperCap = signer.capabilities.storage
    .issue<auth(EVM.Call) &EVM.CadenceOwnedAccount>(/storage/evm)
```

### Anti-pattern C: publishing an auth capability at `/public/evm`

```cadence
// ❌ — CRITICAL: any account on the network can drain the COA
let cap = signer.capabilities.storage
    .issue<auth(EVM.Call) &EVM.CadenceOwnedAccount>(/storage/evm)
signer.capabilities.publish(cap, at: /public/evm)
```

`/public/*` capabilities are resolvable by any Cadence script or transaction via
`getAccount(addr).capabilities.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(...)`. The
first transaction to discover this can drain every ERC20 the COA holds. The public path
must be strictly un-entitled:

```cadence
// ✅ — read-only: anyone can read the address, balance, or deposit
let publicCap = signer.capabilities.storage
    .issue<&EVM.CadenceOwnedAccount>(/storage/evm)
signer.capabilities.publish(publicCap, at: /public/evm)
```

---

## Common Pitfalls

- **`EVM.Withdraw` does not undo EVM calls.** It only moves FLOW from the COA's EVM
  balance to a Cadence vault. ERC20 transfers, approvals, and other EVM state changes
  made by `coa.call` are not affected.

- **Omitting `EVM.Withdraw` when depositing + calling.** If the EVM call reverts and you
  borrowed without `EVM.Withdraw`, recovering the deposited FLOW requires a separate
  transaction. Always include `EVM.Withdraw` in the borrow type whenever the pattern is
  "deposit then call." See [flow-bridge.md](flow-bridge.md) § "Failure Modes".

- **Confusing `EVM.Bridge` with `EVM.Withdraw`.** `EVM.Bridge` is for the
  `onflow/flow-evm-bridge` wrapped-token bridge (NFTs, ERC20s). Native FLOW uses `deposit`
  (no entitlement) and `coa.withdraw` (`EVM.Withdraw`). They are completely separate code
  paths and entitlements.

- **Over-wide entitlement sets are not caught at compile time.** Issuing
  `auth(EVM.Call, EVM.Deploy)` when only `EVM.Call` was needed produces no warning — the
  type system accepts widened sets silently. Code review must enforce narrowing discipline.

- **Store the entitlement set alongside the controller ID.** When a controller is deleted,
  the Cadence runtime does not record what entitlements it had. Keep an audit log of
  `(controllerID, entitlementSet, recipient)` tuples so revocations are traceable.

- **`EVM.Validate` is still a bearer credential.** `protectedAddress()` returns the same
  value as `address()`. The entitlement's value is the signal, not a lock — issuing
  `auth(EVM.Validate)` to a third party still requires "one cap per consumer" hygiene.

- **`EVM.Owner` in an inbox-pending capability is a live bearer grant.** If issued during
  setup and not immediately consumed, it persists until claimed. Prefer the explicit set
  `auth(EVM.Call, EVM.Deploy, EVM.Withdraw, EVM.Bridge)` — it is auditable and revocable
  per entitlement boundary.

---

## See Also

- [coa-lifecycle.md](coa-lifecycle.md) — COA creation, storage paths, basic entitlement borrow patterns, escrow anti-patterns
- [evm-call.md](evm-call.md) — `coa.call` mechanics, `EVM.Result` handling, calldata encoding, gas limits
- [flow-bridge.md](flow-bridge.md) — `coa.deposit` / `coa.withdraw` (native FLOW), decimal reconciliation, atomicity
- [../../cadence-lang/references/entitlements.md](../../cadence-lang/references/entitlements.md) — Cadence entitlement primer: disjunctions, conjunctions, mappings, owned-vs-reference semantics
- [../../cadence-audit/references/crossvm-anti-patterns.md](../../cadence-audit/references/crossvm-anti-patterns.md) — C4 (shared caps), C5 (public auth cap), C3 (deposit+call atomicity) — the full audit checklist for CrossVM code
