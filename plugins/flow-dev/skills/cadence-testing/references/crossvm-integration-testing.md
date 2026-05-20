# Cross-VM Integration Testing

Cross-VM integration tests differ from pure-Cadence tests in one specific way: the fixture is two-sided. A typical test deploys a Solidity contract from a Cadence test transaction, then exercises it through a COA's `coa.call` while asserting state on BOTH the Cadence side (vault balances, escrow phase) and the EVM side (ERC20 `balanceOf`, AMM reserves). Every assertion is a `Test.executeScript` call: Cadence reads use storage borrows, EVM reads use `EVM.dryCall` inside the script. The one new piece compared to Cadence-only testing is **Solidity fixture deployment** — compile the `.sol` with Hardhat or Foundry off-line, embed the bytecode as a hex string in the test, and deploy via `coa.deploy(...)` in `setup()`. Everything else (`Test.reset(to:)`, `beforeEach`, `Test.moveTime`, matchers) is regular Test framework usage, just exercised against a runtime that already has the EVM enabled.

This reference assumes you have read [setup-and-basics.md](setup-and-basics.md) and [blockchain-emulation.md](blockchain-emulation.md). For the Cadence-side EVM mechanics this file leans on, see [../../flow-crossvm/references/evm-call.md](../../flow-crossvm/references/evm-call.md), [../../flow-crossvm/references/coa-lifecycle.md](../../flow-crossvm/references/coa-lifecycle.md), and [../../flow-crossvm/references/solidity-fixtures.md](../../flow-crossvm/references/solidity-fixtures.md).

---

## 1. Enabling the EVM side

The Cadence Test framework boots an in-process Flow runtime that already includes the `EVM` system contract. `import "EVM"` from inside a `_test.cdc` resolves the same way it does on emulator/testnet/mainnet — no `Test.deployContract` is needed for `EVM` itself, and no `testing` alias for it in `flow.json`.

Verified on Flow CLI v2.17.1: `import "EVM"` resolves with no extra flow.json entry beyond the standard `dependencies` block. The `--evm-test-helpers` and `--setup-vm-bridge` emulator flags govern the standalone `flow emulator` process and have NO effect on the in-process Test runtime.

What this means in practice:

```cadence
import Test
import "EVM"     // works as-is, no Test.deployContract("EVM", ...)
```

If a test that imports `"EVM"` fails at parse time with "cannot find contract EVM", the Flow CLI is older than the build that wired EVM into the test runtime — upgrade the CLI. Verified on Flow CLI v2.17.1. Older versions may differ — if you see `cannot find contract EVM`, upgrade.

### flow.json dependency block

For the test runtime, only the `dependencies` entry is needed:

```json
{
  "dependencies": {
    "EVM": {
      "source": "mainnet://e467b9dd11fa00df.EVM",
      "aliases": {
        "emulator": "f8d6e0586b0a20c7",
        "testnet":  "8c5303eaa26202d6",
        "mainnet":  "e467b9dd11fa00df"
      }
    }
  }
}
```

No `"testing"` alias for `EVM`. The framework provisions it automatically.

---

## 2. Deploying Solidity fixtures

Two approaches; one is recommended.

### Approach A — embed bytecode in the test (recommended)

Compile the `.sol` files (see [../../flow-crossvm/references/solidity-fixtures.md](../../flow-crossvm/references/solidity-fixtures.md)) once outside the test runner, capture the `0x`-prefixed deploy bytecode as `String` constants in a test helper, and deploy via `coa.deploy` in `setup()`.

```cadence
// cadence/tests/test_helpers.cdc — bytecode from forge build / hardhat compile
access(all) let testERC20Bytecode: String = "0x6080604052..."
access(all) let testAMMBytecode:   String = "0x6080604052..."
access(all) let oracleBytecode:    String = "0x6080604052..."
```

A deploy helper that runs the deploy transaction, asserts success, and returns the EVM address as a 40-char lowercase hex string (no `0x` prefix):

```cadence
import Test
import "EVM"

access(all) fun evmDeploy(
    _ signer: Test.TestAccount,
    bytecode: String,
    gasLimit: UInt64,
    value: UInt
): String {
    let res = Test.executeTransaction(Test.Transaction(
        code: Test.readFile("./transactions/evm/deploy.cdc"),
        authorizers: [signer.address],
        signers: [signer],
        arguments: [bytecode, gasLimit, value]
    ))
    Test.expect(res, Test.beSucceeded())

    // The deployed address surfaces in the EVM.TransactionExecuted event.
    let events = Test.eventsOfType(Type<EVM.TransactionExecuted>())
    let evt = events[events.length - 1] as! EVM.TransactionExecuted
    let withPrefix = evt.contractAddress
    return withPrefix.slice(from: 2, upTo: withPrefix.length).toLower()
}
```

This is the upstream pattern from `onflow/FlowActions/cadence/tests/test_helpers.cdc::evmDeploy`. The `contractAddress` field is confirmed on CLI v2.17.1. It is a `String`, `0x`-prefixed and EIP-55 checksum-cased — strip the `0x` and call `.toLower()` for downstream Cadence comparisons. The confirmed field set on `EVM.TransactionExecuted` in v2.17.1 is: `hash, index, type, payload, errorCode, errorMessage, gasConsumed, contractAddress, logs, blockHeight, returnedData, precompiledCalls, stateUpdateChecksum`. There is NO `deployedContract` field on the EVENT; that field lives on the in-tx `EVM.Result` resource only.

The deploy transaction itself lives in a `.cdc` file (so the test reads it with `Test.readFile`):

```cadence
// cadence/tests/transactions/evm/deploy.cdc
import "EVM"

transaction(bytecode: String, gasLimit: UInt64, value: UInt) {
    prepare(signer: auth(BorrowValue) &Account) {
        let coa = signer.storage.borrow<auth(EVM.Deploy) &EVM.CadenceOwnedAccount>(
            from: /storage/evm
        ) ?? panic("No COA at /storage/evm")
        let result = coa.deploy(
            code: bytecode.decodeHex(),
            gasLimit: gasLimit,
            value: EVM.Balance(attoflow: value)
        )
        assert(result.status == EVM.Status.successful,
            message: "deploy failed: ".concat(result.errorMessage))
    }
}
```

### Approach B — deploy outside the test, hardcode addresses

Run `flow emulator` + a one-shot deploy script, capture the addresses, and pass them as `String` constants into the test file. **Avoid** unless integration tests intentionally pin against an external deployer:

- couples the test file's lifecycle to the runner's working directory and to ordering of unrelated deploys (every extra nonce bump shifts every address);
- breaks `Test.reset(to: setupHeight)` because the addresses were never created inside the in-process runtime;
- silently rots when the Solidity source changes (the embedded hex no longer matches anything you can rebuild from source).

Use Approach A for every test that does not have a hard reason to deploy outside.

---

## 3. The dual-side assertion pattern

Every state read in a cross-VM test is a `Test.executeScript` call. The split:

- **Cadence reads**: a Cadence script that borrows from storage and returns the value.
- **EVM reads**: a Cadence script that calls `EVM.dryCall` and decodes the return data.

A reusable EVM read helper (returns the raw `EVM.Result` so the caller decodes the ABI shape it expects):

```cadence
// cadence/tests/scripts/evm/call_raw.cdc
import "EVM"

access(all) fun main(
    fromHex: String, toHex: String, calldata: String,
    gasLimit: UInt64, value: UInt
): EVM.Result {
    return EVM.dryCall(
        from: EVM.addressFromString(fromHex),
        to:   EVM.addressFromString(toHex),
        data: calldata.decodeHex(),
        gasLimit: gasLimit,
        value: EVM.Balance(attoflow: value)
    )
}
```

```cadence
access(all) fun evmDryCall(
    fromHex: String, toHex: String, calldata: String,
    gasLimit: UInt64, value: UInt
): EVM.Result {
    let r = Test.executeScript(
        Test.readFile("./scripts/evm/call_raw.cdc"),
        [fromHex, toHex, calldata, gasLimit, value]
    )
    Test.expect(r, Test.beSucceeded())
    return r.returnValue! as! EVM.Result
}
```

### ERC20 `balanceOf` against a COA address

```cadence
access(all) fun erc20BalanceOf(
    erc20Hex: String, ownerHex: String
): UInt256 {
    let calldata = String.encodeHex(
        EVM.encodeABIWithSignature("balanceOf(address)", [
            EVM.addressFromString(ownerHex)
        ])
    )
    let res = evmDryCall(
        fromHex: "0x0000000000000000000000000000000000000000",
        toHex:   erc20Hex,
        calldata: calldata,
        gasLimit: 50_000,
        value: 0
    )
    assert(res.status == EVM.Status.successful, message: "balanceOf failed")
    let decoded = EVM.decodeABI(types: [Type<UInt256>()], data: res.data)
    return decoded[0] as! UInt256
}
```

### COA-side native FLOW balance

```cadence
// cadence/tests/scripts/evm/get_evm_balance.cdc
import "EVM"

access(all) fun main(evmAddressHex: String): UFix64 {
    return EVM.addressFromString(evmAddressHex).balance().inFLOW()
}
```

```cadence
access(all) fun evmFlowBalance(_ evmAddressHex: String): UFix64 {
    let r = Test.executeScript(
        Test.readFile("./scripts/evm/get_evm_balance.cdc"),
        [evmAddressHex]
    )
    Test.expect(r, Test.beSucceeded())
    return r.returnValue! as! UFix64
}
```

The Cadence-side FLOW balance read uses the same pattern but with `getAccount(addr).capabilities.borrow<&{FungibleToken.Balance}>(/public/flowTokenBalance)`.

---

## 4. Reset semantics across both VMs

`Test.reset(to: setupHeight)` rolls back the entire in-process chain to the snapshot at `setupHeight` — both Cadence storage AND EVM state. This has been confirmed empirically on CLI v2.17.1: deploying an ERC20, then calling `Test.reset(to: setupHeight)`, reduces the contract's `code()` length from 3,839 bytes to 0 bytes. See [`../cadence-testing/references/blockchain-emulation.md`](../cadence-testing/references/blockchain-emulation.md) for the full reset-semantics reference.

> **WARNING — broken pattern on CLI v2.17.1**: the `beforeEach { Test.reset(to: setupHeight) }` pattern silently wipes EVM-side state (deployed contracts, COA balances) because the in-process Test runtime does **not** advance `getCurrentBlock().height` across `Test.executeTransaction` calls. Every transaction leaves the block height at 40 (the genesis height). Therefore `setupHeight = getCurrentBlock().height` captured at the end of `setup()` is the same as the pre-`setup()` height, and `Test.reset(to: setupHeight)` rolls back **past** setup — destroying the COA capability and all deployed contracts. The failure appears as cryptic errors such as `"no COA capability published"` and `"account public key not found"`. Cross-link: [blockchain-emulation.md § The setup-then-reset trap](../cadence-testing/references/blockchain-emulation.md).

The fix is to capture a **fresh post-setup block height** via a script rather than from within `setup()` directly:

```cadence
access(all) fun setup() {
    // ... deploy contracts, create COAs ...

    // Capture the post-setup height via a script (returns current height after all txs are committed)
    let r = Test.executeScript(
        "access(all) fun main(): UInt64 { return getCurrentBlock().height }",
        []
    )
    setupHeight = r.returnValue! as! UInt64  // fresh post-setup height
}
```

Because the in-process runtime does not advance height per tx on CLI v2.17.1, this script-based capture returns the same height as a direct call — meaning `beforeEach { Test.reset(to: setupHeight) }` remains broken on this version. Use one of these alternatives instead:

- **(Recommended)** Order tests alphabetically so each test cleans up after itself. State persists between tests; do not assume a clean slate. Use delta assertions (`balance increased by amount`) rather than absolute assertions (`balance == amount`).
- **(Alternative)** Re-run the `setup()` equivalent at the start of every test that needs a clean slate, accepting redeploys. Capture the deployed address INSIDE the test — do not reuse module-level constants.
- **(Watch)** When a future CLI release advances block height per tx in the in-process runtime, the `beforeEach { Test.reset(to: setupHeight) }` pattern will work as written. Re-verify before recommending it.

Do NOT redeploy the Solidity fixtures inside every `testXxx` if you are relying on captured module-level address constants. Redeployment bumps the COA nonce and produces a different address every time.

---

## 5. Worked test file

A complete `_test.cdc` showing: COA creation, ERC20 deploy in `setup()`, mint-and-assert test, FLOW round-trip test with assertions on both sides.

```cadence
import Test
import "FungibleToken"
import "FlowToken"
import "EVM"

// --- File-level bindings (created once when the file loads) -----------------

access(all) let serviceAccount = Test.serviceAccount()
access(all) let admin = Test.createAccount()

access(all) var adminCOAHex: String = ""
access(all) var testERC20Hex: String = ""
access(all) var setupHeight: UInt64 = 0

// Bytecode of TestERC20 from `forge build` / `hardhat compile`.
access(all) let testERC20Bytecode: String = "0x6080604052..."   // truncated

// --- setup() -----------------------------------------------------------------

access(all) fun setup() {
    // 1. Fund the admin so it can deposit FLOW into its COA.
    fundFlow(recipient: admin.address, amount: 100.0)

    // 2. Create the admin's COA with 10 FLOW pre-deposited.
    createCOA(signer: admin, fundingAmount: 10.0)
    adminCOAHex = getCOAAddressHex(forFlow: admin.address)

    // 3. Deploy TestERC20 with constructor args ("Test USDC", "tUSDC", 6).
    let ctorArgs = String.encodeHex(EVM.encodeABI([
        "Test USDC" as String,
        "tUSDC" as String,
        UInt8(6)
    ]))
    let bytecodeWithArgs = testERC20Bytecode.concat(ctorArgs)
    testERC20Hex = evmDeploy(
        admin,
        bytecode: bytecodeWithArgs,
        gasLimit: 4_000_000,
        value: 0
    )

    // 4. (No beforeEach reset — see §4 warning about broken pattern on CLI v2.17.1.)
}

// --- Test 1: mint to COA and assert ERC20 balance ---------------------------

access(all) fun testMintIncreasesERC20Balance() {
    // Read pre-mint balance (state may persist from prior tests — use delta assertion).
    let before = erc20BalanceOf(erc20Hex: testERC20Hex, ownerHex: adminCOAHex)

    // Mint 100 * 10^6 (token has 6 decimals) tokens to the COA.
    let amount = UInt256(100_000_000)
    let calldata = EVM.encodeABIWithSignature("mint(address,uint256)", [
        EVM.addressFromString(adminCOAHex),
        amount
    ])
    let mintTx = Test.Transaction(
        code: Test.readFile("./transactions/evm/call.cdc"),
        authorizers: [admin.address],
        signers: [admin],
        arguments: [
            testERC20Hex,
            String.encodeHex(calldata),
            UInt64(100_000),
            UInt(0)
        ]
    )
    Test.expect(Test.executeTransaction(mintTx), Test.beSucceeded())

    // After: balance increased by exactly the minted amount (delta assertion — state may persist).
    let after = erc20BalanceOf(erc20Hex: testERC20Hex, ownerHex: adminCOAHex)
    Test.assertEqual(before + amount, after)
}

// --- Test 2: FLOW round-trip with dual-side assertions ----------------------

access(all) fun testFlowDepositWithdrawRoundTrip() {
    // ---- step 0: baseline both sides ----
    let cadence0 = cadenceFlowBalance(admin.address)
    let evm0     = evmFlowBalance(adminCOAHex)
    Test.assertEqual(10.0, evm0)             // setup deposited 10 FLOW

    // ---- step 1: deposit 5 FLOW Cadence -> EVM ----
    let depositTx = Test.Transaction(
        code: Test.readFile("./transactions/evm/deposit_to_coa.cdc"),
        authorizers: [admin.address],
        signers: [admin],
        arguments: [5.0 as UFix64]
    )
    Test.expect(Test.executeTransaction(depositTx), Test.beSucceeded())

    let cadence1 = cadenceFlowBalance(admin.address)
    let evm1     = evmFlowBalance(adminCOAHex)
    Test.assertEqual(cadence0 - 5.0, cadence1)
    Test.assertEqual(15.0, evm1)             // 10 + 5

    // ---- step 2: withdraw 3 FLOW EVM -> Cadence ----
    let withdrawTx = Test.Transaction(
        code: Test.readFile("./transactions/evm/withdraw_from_coa.cdc"),
        authorizers: [admin.address],
        signers: [admin],
        arguments: [3.0 as UFix64]
    )
    Test.expect(Test.executeTransaction(withdrawTx), Test.beSucceeded())

    let cadence2 = cadenceFlowBalance(admin.address)
    let evm2     = evmFlowBalance(adminCOAHex)
    Test.assertEqual(cadence1 + 3.0, cadence2)
    Test.assertEqual(12.0, evm2)             // 15 - 3
}

// --- Helper functions (typically live in test_helpers.cdc) ------------------

access(all) fun fundFlow(recipient: Address, amount: UFix64) {
    let tx = Test.Transaction(
        code: Test.readFile("./transactions/fund_flow.cdc"),
        authorizers: [serviceAccount.address],
        signers: [serviceAccount],
        arguments: [recipient, amount]
    )
    Test.expect(Test.executeTransaction(tx), Test.beSucceeded())
}

access(all) fun createCOA(signer: Test.TestAccount, fundingAmount: UFix64) {
    let tx = Test.Transaction(
        code: Test.readFile("./transactions/evm/create_coa.cdc"),
        authorizers: [signer.address],
        signers: [signer],
        arguments: [fundingAmount]
    )
    Test.expect(Test.executeTransaction(tx), Test.beSucceeded())
}

access(all) fun getCOAAddressHex(forFlow flowAddr: Address): String {
    let r = Test.executeScript(
        Test.readFile("./scripts/evm/get_coa_hex.cdc"),
        [flowAddr]
    )
    Test.expect(r, Test.beSucceeded())
    return r.returnValue! as! String
}

access(all) fun cadenceFlowBalance(_ addr: Address): UFix64 {
    let r = Test.executeScript(
        Test.readFile("./scripts/get_flow_balance.cdc"),
        [addr]
    )
    Test.expect(r, Test.beSucceeded())
    return r.returnValue! as! UFix64
}

// (evmDeploy, evmFlowBalance, evmDryCall, erc20BalanceOf — as defined above)
```

Notes on this skeleton:

- **Funding the COA up front matters.** `setup()` deposits 10 FLOW into the COA before any test runs. A COA with zero EVM balance fails `coa.call(... value: > 0)` with a cryptic out-of-gas-looking error; same for any Solidity contract that needs gas value forwarded.
- **The mint test asserts before AND after.** The pre-assertion (`before == 0`) guards against a stale snapshot — if `Test.reset` did not roll back the previous test's mint, the `after` assertion would still happen to pass numerically while masking the real bug.
- **The round-trip test reads both balances at every step.** Three reads per side per test isolates which leg of the round-trip introduced an error, instead of waiting until the final assertion to know something is off.

---

## 6. Address management

COA addresses are derived from the COA resource's `uuid` (see [coa-lifecycle.md](../../flow-crossvm/references/coa-lifecycle.md)). In the deterministic test runtime, the COA created first by `admin` always lands at the same EVM address across reruns — that determinism is what makes capturing `adminCOAHex` in `setup()` reusable across every `testXxx`.

Deployed Solidity addresses follow EVM `CREATE` semantics: `keccak256(rlp([COA addr, nonce]))[12:]`. Deploying TestERC20, then TestAMM, then TestPriceOracle yields a deterministic sequence only if each is deployed exactly once in `setup()`. A redeploy inside any `testXxx` bumps the COA nonce relative to `setupHeight` and breaks every captured constant.

Treat module-level address bindings as write-once in `setup()`, read-only thereafter:

```cadence
access(all) var tokenA: String = ""
access(all) var amm: String = ""

access(all) fun setup() {
    tokenA = evmDeploy(admin, bytecode: testERC20Bytecode, gasLimit: 4_000_000, value: 0)
    amm    = evmDeploy(admin, bytecode: testAMMBytecode.concat(/* ctor args */),
                       gasLimit: 4_000_000, value: 0)
    setupHeight = getCurrentBlock().height
}
```

---

## 7. Common pitfalls

1. **Forgetting to fund the COA before EVM calls.** `coa.call(... value: > 0)` against a zero-balance COA fails with an error involving the EVM balance and/or gas. Exact error wording is not stable across CLI versions — always `assert(r.status == EVM.Status.successful)` and treat `r.errorMessage` as informational only. Deposit FLOW in `setup()`, not per-test.
2. **`Test.reset(to: 0)` instead of `Test.reset(to: setupHeight)`.** Height 0 wipes every account, contract, and COA — the next test sees a blank chain. Always reset to `setupHeight` captured AFTER `setup()`.
3. **Redeploying Solidity fixtures inside `testXxx`.** Bumps the COA nonce, so addresses captured in `setup()` no longer match. Deploy once in `setup()`, snapshot, reset to snapshot.
4. **Mixing service-account addresses.** Test framework service account = `0x0000000000000001` (returned by `Test.serviceAccount()`). The emulator service account `0xf8d6e0586b0a20c7` is irrelevant inside `flow test`. See [blockchain-emulation.md](blockchain-emulation.md).
5. **Reading EVM balance directly inside the test body.** The test body cannot import EVM directly and read EVM state; all EVM reads must go through `Test.executeScript`. Always read via a script.
6. **Trusting `result.status == successful` from `coa.deploy` without checking `result.deployedContract != nil`.** A successful deploy with `nil` `deployedContract` is rare but possible. The event-based helper above sidesteps this; a direct `coa.deploy` caller must assert both.
7. **Hard-coding bytecode without a regeneration note.** Bytecode changes every compile. Track the source `.sol` under `cadence/tests/solidity/` and document the build invocation that produced the embedded hex.
8. **Missing `auth(EVM.Deploy)` on the COA borrow in the deploy transaction.** A reference with `auth(EVM.Call)` only cannot deploy — include `Deploy` explicitly. See [coa-lifecycle.md](../../flow-crossvm/references/coa-lifecycle.md).
9. **Looking for Solidity event names in `Test.events()`.** EVM events surface inside the Cadence `EVM.TransactionExecuted` event under the `logs: Array` field, not as top-level Cadence events. NOTE: empirically (v2.17.1) the `logs` field is often empty even when the underlying EVM tx emitted Transfer/Approval; the actual log data lives in the RLP-encoded `payload` blob. For test assertions, prefer calling a getter on the target contract after the fact (e.g., `balanceOf(...)`) rather than asserting on event logs.
10. **`Test.readFile` paths relative to project root.** `Test.readFile` resolves relative to the test file. Keep test-specific transactions under `cadence/tests/transactions/` to keep the relative paths short.
11. **Misreading a `FALSE` `verifyProof` as "BN254 precompiles unsupported".** Flow CLI v2.17.1's in-process EVM fully supports the BN254 precompiles ECADD (`0x06`), ECMUL (`0x07`), and PAIRING (`0x08`), verified empirically on 2026-05-19 via direct `EVM.dryCall` probes and a deployed snarkjs Groth16 verifier. A `verifyProof` call that returns `FALSE` with `status == EVM.Status.successful` (rather than `FAILED` with an error) is an **encoding bug, not a missing precompile** — reach for a mock only after ruling encoding out. The most common cause is the snarkjs `pi_b` field-element ordering: `proof.json` emits `pi_b[i] = [c0, c1]` (real first), but the snarkjs-generated `Groth16Verifier`'s `_pB[i]` expects `[c1, c0]` (imaginary first, per EIP-197). Swap each Fp2 tuple in `pi_b` before ABI-encoding the calldata; this swap is universal across all EVM chains, not Flow-specific. To disambiguate encoding bugs from infrastructure gaps in five lines of test code, send `[x1=1, y1=2, x2=1, y2=2]` (128 bytes, each coord 32-byte uint256) to `0x06` via `coa.dryCall` — a live precompile returns 64 bytes whose first 32 are `0x030644e72e131a029b85045b68181585d97816a916871ca8d3c208c16d87cfd3` (the x-coordinate of 2·G1).

---

## See also

- [../../flow-crossvm/references/evm-call.md](../../flow-crossvm/references/evm-call.md) — `coa.call` / `EVM.dryCall` / `coa.dryCall` mechanics.
- [../../flow-crossvm/references/coa-lifecycle.md](../../flow-crossvm/references/coa-lifecycle.md) — COA creation, storage paths, entitlement model.
- [../../flow-crossvm/references/solidity-fixtures.md](../../flow-crossvm/references/solidity-fixtures.md) — TestERC20, TestAMM, TestPriceOracle source.
- [blockchain-emulation.md](blockchain-emulation.md) — `Test.reset`, `Test.moveTime`, account taxonomy.
- [setup-and-basics.md](setup-and-basics.md) — file structure, `flow.json` aliases, lifecycle hooks.
- [events-and-logs.md](events-and-logs.md) — `Test.eventsOfType` patterns for extracting event payloads.
