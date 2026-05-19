# EVM Tooling on Flow

Flow EVM lets you deploy and interact with Solidity smart contracts on Flow using standard Ethereum tooling. Flow runs a full EVM interpreter (Geth v1.13) accessible via standard JSON-RPC.

**Only needed for Solidity/EVM development on Flow.** If you're writing Cadence contracts, skip this.

## Flow EVM Networks

| Network | Chain ID | RPC Endpoint | Block Explorer |
|---------|----------|--------------|----------------|
| Mainnet | 747 | `https://mainnet.evm.nodes.onflow.org` | `https://evm.flowscan.io` |
| Testnet | 545 | `https://testnet.evm.nodes.onflow.org` | `https://evm-testnet.flowscan.io` |

- Currency symbol: FLOW
- Token denomination: Atto-FLOW (1 FLOW = 10^18 Atto-FLOW), same as wei in Ethereum

## Local Development with EVM Gateway

### Why two processes are required

`flow emulator` alone does NOT serve EVM JSON-RPC. The emulator exposes only Flow REST, gRPC,
and admin ports; sending `eth_chainId` to any of those returns a 404 or protocol error. EVM
JSON-RPC only becomes available after starting `flow evm gateway` as a separate process.

### Process 1: Flow Emulator

```bash
flow emulator \
  --rest-port=8880 \
  --evm-test-helpers \
  --log-format=text
```

`--evm-test-helpers` activates convenience EVM debugging endpoints. Moving the emulator REST
to 8880 (from the default 8888) avoids a port conflict if you later assign the gateway to 8888.

### Process 2: EVM Gateway

```bash
COA_ADDR="f8d6e0586b0a20c7"   # emulator service account address
COA_KEY="<service-account-private-key-no-0x-prefix>"

flow evm gateway \
  --access-node-host localhost:3569 \
  --flow-network-id emulator \
  --evm-network-id testnet \
  --coa-address $COA_ADDR \
  --coa-key $COA_KEY \
  --coa-resource-create \
  --coinbase 0xYOUR_COINBASE_ADDRESS \
  --database-dir .evm-gateway-db \
  --gas-price 1 \
  --rpc-port 3000
```

### Chain ID truth table

`--evm-network-id` controls the chain ID the gateway reports. The default is `testnet`.
`emulator` is NOT a valid value for this flag (it is valid for `--flow-network-id`).

| `--evm-network-id` | `eth_chainId` (hex) | Decimal | Network |
|--------------------|---------------------|---------|---------|
| `testnet` (default)| `0x221` | **545** | Flow EVM Testnet |
| `preview` | `0x286` | **646** | Flow Previewnet |
| `mainnet` | `0x2eb` | **747** | Flow EVM Mainnet |
| `emulator` | error | — | not supported |

For local development use `testnet` (chain ID 545) or `preview` (chain ID 646). Do not pass
`emulator` — it is rejected with "EVM network ID not supported".

### Port conflict note

The gateway's default RPC port is **3000**, not 8888. Running both processes with stock
defaults (emulator REST=8888, gateway RPC=3000) produces no conflict.

The conflict only arises when you explicitly pass `--rpc-port 8888` to the gateway (a common
choice for Ethereum familiarity). In that case, also pass `--rest-port=8880` to the emulator
so both can bind.

### Verify the gateway is running

```bash
curl -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}' \
  http://127.0.0.1:<GATEWAY_PORT>/
# Expected: {"jsonrpc":"2.0","id":1,"result":"0x221"}  (chain 545 for testnet)
```

### Hardhat config for local gateway

Point `flowLocal` at whatever port you chose (3000 default, or 8888 if overridden):

```typescript
flowLocal: {
  url: 'http://127.0.0.1:3000',   // change to 8888 if you ran --rpc-port 8888
  accounts: [process.env.DEPLOY_WALLET_1 as string],
  chainId: 545,
},
```

---

## Funding an EVM Address from Cadence

The gateway's `--coinbase` starts at zero FLOW, so your first deploy will fail on gas without
a funding step. The full transaction (Cadence code, gotchas, arithmetic notes, and verification
curl) is in [`funding-evm-from-cadence.md`](funding-evm-from-cadence.md).

Quick summary:

- Use `auth(BorrowValue) &Account` — `BorrowValue` alone is sufficient for `.borrow()`.
- `EVM.Balance(attoflow:)` takes **`UInt`**, NOT `UInt256`.
- Cast the withdrawn vault: `<-withdrawn as! @FlowToken.Vault` (intersection vs concrete type).
- `coa.deposit` funds the COA only; follow with `coa.call` to reach an external EVM address.

---

## Hardhat

### Prerequisites
- Node.js installed

### Setup
```bash
npx hardhat init
npm install --save-dev @nomicfoundation/hardhat-toolbox-viem
npm install --save-dev @nomicfoundation/hardhat-ethers ethers
npm install --save-dev @openzeppelin/contracts
npm install --save-dev @openzeppelin/hardhat-upgrades
npm install dotenv
```

### hardhat.config.ts
```typescript
import type { HardhatUserConfig } from 'hardhat/config';
import '@nomicfoundation/hardhat-toolbox-viem';

require('@openzeppelin/hardhat-upgrades');
require('dotenv').config();

const config: HardhatUserConfig = {
  solidity: '0.8.24',
  networks: {
    flow: {
      url: 'https://mainnet.evm.nodes.onflow.org',
      accounts: [process.env.DEPLOY_WALLET_1 as string],
    },
    flowTestnet: {
      url: 'https://testnet.evm.nodes.onflow.org',
      accounts: [process.env.DEPLOY_WALLET_1 as string],
    },
  },
  etherscan: {
    apiKey: {
      flow: 'abc',
      flowTestnet: 'abc',
    },
    customChains: [
      {
        network: 'flow',
        chainId: 747,
        urls: {
          apiURL: 'https://evm.flowscan.io/api',
          browserURL: 'https://evm.flowscan.io/',
        },
      },
      {
        network: 'flowTestnet',
        chainId: 545,
        urls: {
          apiURL: 'https://evm-testnet.flowscan.io/api',
          browserURL: 'https://evm-testnet.flowscan.io/',
        },
      },
    ],
  },
};

export default config;
```

### Environment
Create `.env`:
```
DEPLOY_WALLET_1=<YOUR_PRIVATE_KEY>
```

### Deploy and Verify
```bash
npx hardhat ignition deploy ./ignition/modules/MyContract.ts --network flowTestnet
hardhat ignition verify chain-545 --include-unrelated-contracts
```

## Foundry

### Install
```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

This installs `forge`, `cast`, `anvil`, and `chisel`.

### Create Project
```bash
mkdir myproject && cd myproject
forge init
forge install OpenZeppelin/openzeppelin-contracts
```

### Build and Test
```bash
forge compile
forge test
```

### Deploy

**Important:** Use `--legacy` on all Flow commands — Flow does not support EIP-1559 transactions.

```bash
forge create --broadcast src/MyContract.sol:MyContract \
  --rpc-url https://testnet.evm.nodes.onflow.org \
  --private-key $DEPLOYER_PRIVATE_KEY \
  --constructor-args <args> \
  --legacy
```

### Verify
```bash
forge verify-contract --rpc-url https://testnet.evm.nodes.onflow.org/ \
  --verifier blockscout \
  --verifier-url https://evm-testnet.flowscan.io/api \
  $CONTRACT_ADDRESS \
  src/MyContract.sol:MyContract
```

### Useful Cast Commands
```bash
# Generate a new wallet
cast wallet new

# Check balance
cast balance --ether --rpc-url https://testnet.evm.nodes.onflow.org $ADDRESS
```

## Remix IDE

1. Add Flow network to MetaMask (use the network details above)
2. Fund your account via the [Flow Faucet](https://faucet.flow.com/fund-account)
3. In Remix, select **Injected Provider - MetaMask** as the environment
4. Deploy and interact with contracts through MetaMask

## Flow-Specific Differences from Ethereum

- **Foundry requires `--legacy` flag** — Flow does not support EIP-1559 transactions via Foundry. Use `--legacy` on `forge create` and `cast send`.
- **Etherscan API key is a placeholder** — Flowscan uses Blockscout, which does not require a real API key. Use `"abc"` or any non-empty string in Hardhat's `etherscan.apiKey`.
- **Verification uses Blockscout** — not Etherscan. Foundry uses `--verifier blockscout`. Hardhat handles this via `customChains`.
- **No exportable private keys from Flow Wallet** — Use MetaMask or another standard EOA wallet for Hardhat/Foundry. Flow Wallet keys are not compatible.
- **Zero base fee** — Gas costs are extremely low since the EVM base fee is zero.
- **Flow Wallet gas sponsorship** — The Flow Wallet provides automatic gas sponsorship on both testnet and mainnet.

## Funding Testnet Accounts

Get testnet FLOW tokens from the faucet:
- Web: https://faucet.flow.com/fund-account
- CLI: `flow accounts fund --network testnet <account>`

## zk-SNARK / Groth16 Verifiers

Deploying snarkjs-generated Groth16 verifiers on Flow EVM is mechanically identical to deploying on Ethereum, with one EIP-197 calldata convention that catches every team the first time. See [groth16-verifier-deploy.md](groth16-verifier-deploy.md) for the proof-encoding swap, chainId alignment, and on-chain testing checklist.

## Documentation

- EVM on Flow: https://developers.flow.com/build/evm
- Network details: https://developers.flow.com/build/evm/networks
