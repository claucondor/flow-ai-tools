# Testing FT and NFT Contracts

Contracts that conform to `FungibleToken` or `NonFungibleToken` need the standard interfaces and any utility contracts (`Burner`, `FungibleTokenMetadataViews`, `MetadataViews`) deployed before they will compile in the test environment. This reference covers the patterns specific to token-standard tests: dependency deployment order, the `createEmptyVault` lifecycle, the difference between a contract interface and a concrete implementation, and how to write a test-only token mock.

## FungibleToken Is a Contract Interface, Not a Contract

`FungibleToken` declares an interface for vault types, but most of its methods have **no body** — only post-conditions. The contract-level method:

```cadence
// In FungibleToken (the interface), declared but not implemented:
access(all) fun createEmptyVault(vaultType: Type): @{FungibleToken.Vault} {
    post {
        result.balance == 0.0: "..."
        result.getType() == vaultType: "..."
    }
}
```

Critically, the interface body has no concrete implementation. **You cannot call `FungibleToken.createEmptyVault(...)` from another contract.** Doing so produces:

```
error: cannot find variable in this scope: FungibleToken
```

The reason: `FungibleToken` is not deployed as an executable contract whose static functions you can invoke. It's a contract-interface declaration. To create an empty vault, call the **concrete contract** that conforms to it:

```cadence
// correct — concrete contract has the body
let vault <- TestToken.createEmptyVault(vaultType: Type<@TestToken.Vault>())

// wrong — interface, no body to invoke
let vault <- FungibleToken.createEmptyVault(vaultType: Type<@{FungibleToken.Vault}>())
```

This is why production protocols that accept "any FT" usually take a vault parameter in their setup transaction (the caller supplies an empty vault of the right concrete type) rather than constructing one in `init`. A contract that hard-codes `TestToken.createEmptyVault(...)` in its init is fine for tests but not deployable to mainnet against arbitrary tokens.

## A Test-Only TestToken Mock

A common pattern: write a `TestToken.cdc` that conforms to `FungibleToken` and lives only in the test fixture. The minimum viable shape:

```cadence
import "FungibleToken"
import "Burner"

access(all) contract TestToken: FungibleToken {
    access(all) var totalSupply: UFix64

    access(all) resource Vault: FungibleToken.Vault {
        access(all) var balance: UFix64
        init(balance: UFix64) { self.balance = balance }

        access(all) view fun getBalance(): UFix64 { return self.balance }
        access(all) view fun getSupportedVaultTypes(): {Type: Bool} {
            return { self.getType(): true }
        }
        access(all) view fun isSupportedVaultType(type: Type): Bool {
            return self.getSupportedVaultTypes()[type] ?? false
        }
        access(all) view fun isAvailableToWithdraw(amount: UFix64): Bool {
            return self.balance >= amount
        }

        access(FungibleToken.Withdraw) fun withdraw(amount: UFix64): @{FungibleToken.Vault} {
            self.balance = self.balance - amount
            return <- create Vault(balance: amount)
        }
        access(all) fun deposit(from: @{FungibleToken.Vault}) {
            let v <- from as! @TestToken.Vault
            self.balance = self.balance + v.balance
            destroy v
        }

        // The PER-VAULT createEmptyVault — required by FungibleToken.Vault.
        // Returns @{FungibleToken.Vault}, takes NO arguments.
        access(all) fun createEmptyVault(): @{FungibleToken.Vault} {
            return <- create Vault(balance: 0.0)
        }

        access(all) view fun getViews(): [Type] { return [] }
        access(all) fun resolveView(_ view: Type): AnyStruct? { return nil }
        access(contract) fun burnCallback() {
            if self.balance > 0.0 {
                TestToken.totalSupply = TestToken.totalSupply - self.balance
            }
            self.balance = 0.0
        }
    }

    // The CONTRACT-LEVEL createEmptyVault — required by FungibleToken.
    // Returns @{FungibleToken.Vault}, takes a `vaultType: Type` argument.
    access(all) fun createEmptyVault(vaultType: Type): @{FungibleToken.Vault} {
        return <- create Vault(balance: 0.0)
    }

    access(all) view fun getContractViews(resourceType: Type?): [Type] { return [] }
    access(all) fun resolveContractView(resourceType: Type?, viewType: Type): AnyStruct? {
        return nil
    }

    init() {
        self.totalSupply = 0.0
    }
}
```

Two `createEmptyVault` declarations, two different signatures:
- `Vault.createEmptyVault(): @{FungibleToken.Vault}` — instance method on a vault. No arguments.
- `TestToken.createEmptyVault(vaultType: Type): @{FungibleToken.Vault}` — contract-level. Takes a `vaultType: Type` parameter.

If you write the contract-level one without the `vaultType` parameter, conformance fails at compile time with:

```
error: cannot use incompatible type when conforming to FungibleToken
```

## Deployment Order in setup()

Two categories of contracts are involved:

**Dependency contracts** (`Burner`, `FungibleToken`, `FungibleTokenMetadataViews`, `NonFungibleToken`, `MetadataViews`, `ViewResolver`):
- Declared in `flow.json` `dependencies` (typically pulled via `flow dependencies install`).
- Add a `testing` alias to each one's entry in `flow.json`.
- These **AUTO-LOAD** when imported by your contracts. **Do not** call `Test.deployContract` on them — the framework will fail with `account with address 0x... not found` because that address is already provisioned for the dependency.

**Project contracts** (your `TestToken.cdc`, `TipJar.cdc`, etc.):
- Declared in `flow.json` `contracts` with `source` and `testing` alias.
- These do NOT auto-deploy. You MUST call `Test.deployContract(name, path, arguments)` in `setup()` for each, in dependency order.

Example `setup()` for a TipJar consumer (only project contracts deployed):

```cadence
import Test
import "FungibleToken"
import "TestToken"
import "TipJar"

access(all) fun setup() {
    // Burner, FungibleToken, ViewResolver auto-load via testing alias — DO NOT deploy them.
    
    // Project contracts deploy explicitly:
    var err = Test.deployContract(
        name: "TestToken",
        path: "../contracts/TestToken.cdc",
        arguments: []
    )
    Test.expect(err, Test.beNil())
    
    err = Test.deployContract(
        name: "TipJar",
        path: "../contracts/TipJar.cdc",
        arguments: []
    )
    Test.expect(err, Test.beNil())
}
```

Pull standard sources to disk with `flow dependencies install` so they live under `imports/<addr>/<Name>.cdc` and can be referenced from `flow.json` by source path.

## Common Pitfalls

| Error | Root cause | Fix |
|---|---|---|
| `cannot find variable in this scope: FungibleToken` | Tried to call `FungibleToken.X(...)` as if it were a concrete contract. | Call the concrete implementation instead, e.g. `TestToken.createEmptyVault(...)`. |
| `cannot find declaration FungibleToken in <path>` | `flow.json` has no `FungibleToken` contract entry, or the source path doesn't exist on disk. | Add `FungibleToken` to `flow.json` `contracts` with `source` + `testing` alias; run `flow dependencies install` to materialize the source. |
| `account with address 0000000000000XXX not found` | You called `Test.deployContract` on a dependency contract that already has a `testing` alias — the framework auto-loaded it into that address and the redeploy attempt collides. | Remove the `Test.deployContract` call for that dependency. Only project contracts need explicit deployment; dependencies AUTO-LOAD via `testing` alias. |
| `cannot use incompatible type when conforming to FungibleToken` | `createEmptyVault` signature mismatch — usually missing the `vaultType: Type` parameter on the contract-level method. | Match the interface exactly: `access(all) fun createEmptyVault(vaultType: Type): @{FungibleToken.Vault}`. |
| `could not resolve address of contract X: contract X not found` | Contract X has a `testing` alias in `flow.json` but was never `Test.deployContract`-ed. | Add the missing `Test.deployContract("X", "<path>", [])` to `setup()`. |
