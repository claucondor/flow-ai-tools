# Flow DeFi Protocol Architecture

## Three Structural Advantages

### 1. MEV-Free EVM
Flow's consensus architecture separates transaction ordering from execution. Validators cannot reorder transactions to extract MEV (no front-running, no sandwich attacks) on the EVM side.

**DeFi implication:** LP positions retain significantly more fee yield vs Ethereum/Base where MEV bots extract 5–15% of AMM fees. This is a structural LP yield premium for Flow DEXes.

### 2. Cross-VM Atomic Transactions
A single Cadence transaction can call both Cadence contracts and EVM contracts atomically. Either everything executes or nothing does — with no bridge required.

```cadence
// Single transaction: Cadence logic + EVM call — atomic
transaction() {
    prepare(signer: auth(BorrowValue) &Account) {
        let coa = signer.storage.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(
            from: /storage/evm
        ) ?? panic("No COA found")

        // Call an EVM contract atomically within this Cadence transaction
        // evmContractAddress is an EVM.EVMAddress (e.g., obtained from a stored address or COA)
        let result = coa.call(
            to: evmContractAddress,
            data: calldata,
            gasLimit: 200000,
            value: EVM.Balance(attoflow: UInt(0))
        )
    }
}
```

**Supported since:** Crescendo upgrade (September 2024)

### 3. SPoCKs (Specialized Proofs of Confidential Knowledge)
SPoCKs are a consensus-layer cryptographic mechanism used by Flow execution nodes to prove they correctly executed a chunk of transactions without revealing the full computation. This is an internal node-protocol primitive — **not accessible to DeFi application developers** and not relevant to Cadence contract or transaction logic.

> **Note:** Privacy-preserving DeFi features (hidden order books, sealed auctions) are not natively enabled by SPoCKs. They require application-layer design patterns.

---

## COA Pattern (Cadence Owned Accounts)

COAs bridge Cadence and EVM within a single Flow address. One Flow account controls both a Cadence storage space and an EVM address.

### Creating a COA
```cadence
transaction() {
    prepare(signer: auth(SaveValue, IssueStorageCapabilityController, PublishCapability) &Account) {
        // Create COA (one-time per account)
        if signer.storage.borrow<&EVM.CadenceOwnedAccount>(from: /storage/evm) == nil {
            let coa <- EVM.createCadenceOwnedAccount()
            signer.storage.save(<-coa, to: /storage/evm)
            let cap = signer.capabilities.storage.issue<&EVM.CadenceOwnedAccount>(/storage/evm)
            signer.capabilities.publish(cap, at: /public/evm)
        }
    }
}
```

### Using COA for EVM Calls
```cadence
// Borrow COA with call permission
let coa = signer.storage.borrow<auth(EVM.Call) &EVM.CadenceOwnedAccount>(
    from: /storage/evm
) ?? panic("COA not found")

// Encode EVM calldata (ABI encoding)
let calldata: [UInt8] = /* ABI-encoded function call */

// Call EVM contract
let result = coa.call(
    to: evmContractAddress,
    data: calldata,
    gasLimit: 100000,
    value: EVM.Balance(attoflow: UInt(0))
)
```

### COA Architecture Use Cases
| Pattern | How |
|---------|-----|
| Cadence NFT + EVM royalties | COA holds EVM revenue, Cadence logic distributes |
| Hybrid DEX | Cadence order book + EVM AMM liquidity pools |
| Cross-chain bridge endpoint | Cadence validates, COA executes EVM mint/burn |
| Protocol-owned EVM liquidity | Cadence governance controls EVM LP positions |

---

## On-Chain Automation (FlowTransactionScheduler)

Native protocol scheduling via the `FlowTransactionScheduler` contract removes
the need for off-chain keepers. Shipped with the Forte network upgrade
(October 22, 2025).

### DeFi use cases where scheduled txs pay for themselves

- **Automated rebalancing** — AutoBalancer pattern reacts to time, not state changes.
- **Interest accrual** — lending protocols can amortise the cost of compounding
  across many positions without keeper networks.
- **Epoch transitions** — staking / vesting contracts that gate by time instead
  of by a triggering tx.
- **Reward distribution** — no cron bots; the protocol pays itself to pay users.

### When to choose scheduled tx vs alternatives

- **Trigger is time** → scheduled tx.
- **Trigger is on-chain state change** → event hook in the state-changing tx,
  not a scheduled tx.
- **Trigger is external (oracle, off-chain API)** → off-chain keeper or oracle
  push.

For the full API (scheduling, callback resource interface, priority/fees,
cancellation, per-tick CU ceiling, failure handling), see
[`cadence-lang/references/scheduled-transactions.md`](../../cadence-lang/references/scheduled-transactions.md).
For testing scheduled tx in the Cadence Test framework, see
[`cadence-testing/references/scheduled-tx-time-mocking.md`](../../cadence-testing/references/scheduled-tx-time-mocking.md).
For the `flow schedule` CLI, see
[`flow-cli/references/scheduled-transactions.md`](../../flow-cli/references/scheduled-transactions.md).
For audit patterns specific to scheduled tx, see
[`cadence-audit/references/forte-anti-patterns.md`](../../cadence-audit/references/forte-anti-patterns.md).

---

## VRF Randomness

Flow provides on-chain verifiable random numbers via the `RandomBeaconHistory` contract — no oracle required.

```cadence
import "RandomBeaconHistory"

access(all) fun getRandomSeed(blockHeight: UInt64): [UInt8] {
    return RandomBeaconHistory.sourceOfRandomness(atBlockHeight: blockHeight).value
}
```

**DeFi applications:** Fair lottery/raffle contracts, randomized NFT drops, prediction market resolution.

> **See also:** `defi-primitives.md` for building blocks (lending models, AMM selection).
