# Funding an EVM Address from Cadence (Emulator)

## Why this step is needed

The gateway's `--coinbase` address starts at zero FLOW balance. When you try to deploy a
Solidity contract, the deployer transaction will fail on gas. You must bridge FLOW from
Cadence to an EVM address before the first deploy.

## The funding transaction

Save as `fund_evm.cdc`. Key correctness notes inline:

- Signer entitlement is `auth(BorrowValue) &Account`. `BorrowValue` alone is sufficient
  for `.storage.borrow()`. The `Storage` entitlement is only needed for `.save()` / `.load()`.
- `EVM.Balance(attoflow:)` takes **`UInt`** (Cadence arbitrary-precision integer), NOT `UInt256`.
  Passing `UInt256` fails at runtime: `expected 'UInt', got 'UInt256'`.
- `vaultRef.withdraw()` returns `@{FungibleToken.Vault}` (intersection type); `coa.deposit`
  expects concrete `@FlowToken.Vault` — the downcast `as! @FlowToken.Vault` is mandatory.

```cadence
import FungibleToken from 0xee82856bf20e2aa6
import FlowToken from 0x0ae53cb6e3f42a79
import EVM from 0xf8d6e0586b0a20c7

transaction(amount: UFix64, targetEVMAddressHex: String) {
    prepare(signer: auth(BorrowValue) &Account) {
        let vaultRef = signer.storage.borrow<auth(FungibleToken.Withdraw) &FlowToken.Vault>(
            from: /storage/flowTokenVault
        ) ?? panic("Could not borrow FlowToken.Vault")

        let withdrawn: @{FungibleToken.Vault} <- vaultRef.withdraw(amount: amount)
        let vault <- withdrawn as! @FlowToken.Vault   // required: intersection -> concrete type

        let coa = signer.storage.borrow<auth(EVM.Call, EVM.Withdraw) &EVM.CadenceOwnedAccount>(
            from: /storage/evm
        ) ?? panic("Could not borrow COA")

        // attoFLOW conversion — UInt is arbitrary-precision; overflow risk is UFix64 mul (~1844 FLOW max)
        let attoFlowAmount: UInt = UInt(amount * 100_000_000.0) * 10_000_000_000
        let evmBalance = EVM.Balance(attoflow: attoFlowAmount)   // UInt, NOT UInt256

        coa.deposit(from: <-vault)   // funds COA's internal EVM balance only

        let addrBytes = targetEVMAddressHex.decodeHex()
        let evmAddress = EVM.EVMAddress(
            bytes: [addrBytes[0],  addrBytes[1],  addrBytes[2],  addrBytes[3],  addrBytes[4],
                    addrBytes[5],  addrBytes[6],  addrBytes[7],  addrBytes[8],  addrBytes[9],
                    addrBytes[10], addrBytes[11], addrBytes[12], addrBytes[13], addrBytes[14],
                    addrBytes[15], addrBytes[16], addrBytes[17], addrBytes[18], addrBytes[19]]
        )

        // coa.call transfers from COA's EVM balance to the external address
        coa.call(to: evmAddress, data: [], gasLimit: 21_000, value: evmBalance)
    }
}
```

## Send command

```bash
flow transactions send fund_evm.cdc 1.0 "000000000000000000000000000000000000dEaD" \
  --network emulator \
  --signer emulator-account
```

## Gotchas

**Entitlements**: `auth(BorrowValue)` is sufficient for this transaction. `Storage` is required
only for `.storage.save()` and `.storage.load()` — not for `.storage.borrow()`.

**Deposit vs call**: `coa.deposit` loads FLOW into the COA's own EVM-side balance. The external
target address is untouched until you follow with `coa.call`. Both steps are required.

**Vault cast**: `vaultRef.withdraw()` returns `@{FungibleToken.Vault}` (intersection type).
`coa.deposit` expects `@FlowToken.Vault`. Without the cast you get:
`expected 'FlowToken.Vault', got '{FungibleToken.Vault}'`.

**Arithmetic and overflow**: `UInt` in Cadence is an arbitrary-precision bignum — it never
overflows. The overflow risk is in the UFix64 multiplication step `amount * 100_000_000.0`,
which overflows when `amount > ~1844 FLOW` (UFix64 maximum ~184,467,440,737 / 1e8 ≈ 1844.67).
For large funding amounts above 1844 FLOW, send multiple transactions.

## Verify on the gateway

After sending, confirm the balance via JSON-RPC:

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_getBalance","params":["0x000000000000000000000000000000000000dEaD","latest"],"id":1}' \
  http://127.0.0.1:3000/
# -> {"result":"0xde0b6b3a7640000"}  = 1_000_000_000_000_000_000 attoFLOW = 1.0 FLOW
```
