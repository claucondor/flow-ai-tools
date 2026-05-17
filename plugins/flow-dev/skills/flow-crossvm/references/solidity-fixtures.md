# Solidity Test Fixtures for CrossVM Integration Testing

When a Cadence integration test needs to call into Flow EVM, the EVM side of the test must own some deployed contracts to call. This reference provides three minimal, single-file Solidity stubs — a permissionless ERC20, a constant-product AMM, and a settable Chainlink-style oracle — that you compile with Hardhat or Foundry, then deploy from a Cadence test transaction via `coa.deploy(...)`. They are deliberately stripped down: no access control, no reentrancy guards, no overflow-safety beyond the 0.8.x compiler default, no event emission for flows the tests don't observe, no gas optimization, and no audit. Their job is to let `coa.call` round-trip through real EVM bytecode inside `flow emulator` so the Cadence side has something deterministic to assert against. They MUST NEVER be deployed to mainnet — a public `mint(...)` with no guard is an infinite-supply faucet for anyone holding an EVM wallet, an AMM without slippage protection drains itself on the first arbitrage tx, and a `setAnswer(...)` with no auth on a price oracle is a self-service price manipulation knob. Treat the rest of this file as test scaffolding only.

See [evm-call.md](evm-call.md) for the Cadence-side mechanics of calling these stubs (selectors, ABI encoding, `result.status`). See [erc20-read.md](erc20-read.md) for read patterns against the ERC20 stub. The build-chain bootstrap (Hardhat config, Foundry install, `forge create` flags, `--legacy` rule) lives in `flow-dev-setup/references/evm-tooling.md` — this reference does not duplicate it.

---

## TestERC20.sol — minimal permissionless ERC20

Drop into `contracts/TestERC20.sol` (Hardhat) or `src/TestERC20.sol` (Foundry). Compiles standalone — no OpenZeppelin import.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// @title TestERC20
/// @notice Permissionless test-only ERC20. Anyone can mint or burn. DO NOT DEPLOY TO MAINNET.
contract TestERC20 {
    string public name;
    string public symbol;
    uint8 public immutable decimals;
    uint256 public totalSupply;

    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    constructor(string memory _name, string memory _symbol, uint8 _decimals) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        _transfer(msg.sender, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        uint256 allowed = allowance[from][msg.sender];
        require(allowed >= amount, "ERC20: allowance");
        if (allowed != type(uint256).max) {
            allowance[from][msg.sender] = allowed - amount;
        }
        _transfer(from, to, amount);
        return true;
    }

    /// @notice Permissionless mint — tests need to fund arbitrary addresses freely.
    function mint(address to, uint256 amount) external {
        totalSupply += amount;
        balanceOf[to] += amount;
        emit Transfer(address(0), to, amount);
    }

    /// @notice Permissionless burn — tests need to drain arbitrary addresses freely.
    function burn(address from, uint256 amount) external {
        require(balanceOf[from] >= amount, "ERC20: burn>balance");
        balanceOf[from] -= amount;
        totalSupply -= amount;
        emit Transfer(from, address(0), amount);
    }

    function _transfer(address from, address to, uint256 amount) internal {
        require(balanceOf[from] >= amount, "ERC20: balance");
        unchecked { balanceOf[from] -= amount; }
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
    }
}
```

What this intentionally omits compared to a production ERC20:
- No `Ownable` / `AccessControl` — anyone can mint. This is by design for tests.
- No permit / EIP-2612 — tests use direct `approve`.
- No fee-on-transfer / hooks — keeps invariants simple to assert.
- No `try`/`catch` on `_transfer` overflows — relies on 0.8.x checked arithmetic.

---

## TestAMM.sol — minimal constant-product AMM

Two-token pool, `x * y = k`. No fees, no slippage protection, no LP tokens. Useful for testing swap routing, price queries, and slippage handling on the Cadence side.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IERC20Minimal {
    function transfer(address to, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

/// @title TestAMM
/// @notice Constant-product (x*y=k) AMM stub for integration tests. No fee, no LP tokens,
///         no slippage guard. DO NOT DEPLOY TO MAINNET.
contract TestAMM {
    address public immutable tokenA;
    address public immutable tokenB;

    uint256 public reserveA;
    uint256 public reserveB;

    event LiquidityAdded(address indexed provider, uint256 amountA, uint256 amountB);
    event Swap(address indexed trader, address tokenIn, uint256 amountIn, uint256 amountOut);

    constructor(address _tokenA, address _tokenB) {
        require(_tokenA != address(0) && _tokenB != address(0), "AMM: zero token");
        require(_tokenA != _tokenB, "AMM: identical tokens");
        tokenA = _tokenA;
        tokenB = _tokenB;
    }

    /// @notice Pull `amountA` of tokenA and `amountB` of tokenB from msg.sender.
    /// @dev    Caller must `approve` this contract beforehand.
    function addLiquidity(uint256 amountA, uint256 amountB) external {
        require(amountA > 0 && amountB > 0, "AMM: zero amount");
        require(IERC20Minimal(tokenA).transferFrom(msg.sender, address(this), amountA), "AMM: pull A");
        require(IERC20Minimal(tokenB).transferFrom(msg.sender, address(this), amountB), "AMM: pull B");
        reserveA += amountA;
        reserveB += amountB;
        emit LiquidityAdded(msg.sender, amountA, amountB);
    }

    /// @notice Swap `amountIn` of `tokenIn` for the other token at the current x*y=k price.
    /// @return amountOut Tokens of the OTHER asset sent to msg.sender.
    function swap(uint256 amountIn, address tokenIn) external returns (uint256 amountOut) {
        require(amountIn > 0, "AMM: zero in");
        require(tokenIn == tokenA || tokenIn == tokenB, "AMM: unknown token");

        bool aIsIn = (tokenIn == tokenA);
        uint256 reserveIn  = aIsIn ? reserveA : reserveB;
        uint256 reserveOut = aIsIn ? reserveB : reserveA;
        require(reserveIn > 0 && reserveOut > 0, "AMM: empty pool");

        // x * y = k  =>  amountOut = reserveOut - k / (reserveIn + amountIn)
        // Equivalent to: amountOut = (amountIn * reserveOut) / (reserveIn + amountIn)
        amountOut = (amountIn * reserveOut) / (reserveIn + amountIn);
        require(amountOut > 0, "AMM: zero out");

        address tokenOut = aIsIn ? tokenB : tokenA;
        require(IERC20Minimal(tokenIn).transferFrom(msg.sender, address(this), amountIn), "AMM: pull in");
        require(IERC20Minimal(tokenOut).transfer(msg.sender, amountOut), "AMM: send out");

        if (aIsIn) { reserveA += amountIn; reserveB -= amountOut; }
        else       { reserveB += amountIn; reserveA -= amountOut; }

        emit Swap(msg.sender, tokenIn, amountIn, amountOut);
    }

    function getReserves() external view returns (uint256, uint256) {
        return (reserveA, reserveB);
    }

    /// @notice Instantaneous price of `token` denominated in the OTHER token, scaled by 1e18.
    /// @dev    Returns 0 if pool is empty. Spot price only — no TWAP.
    function getPrice(address token) external view returns (uint256) {
        require(token == tokenA || token == tokenB, "AMM: unknown token");
        if (reserveA == 0 || reserveB == 0) return 0;
        if (token == tokenA) {
            return (reserveB * 1e18) / reserveA;
        } else {
            return (reserveA * 1e18) / reserveB;
        }
    }
}
```

What this intentionally omits compared to a production AMM (Uniswap V2 etc.):
- No 0.3% fee on swaps — exact-arithmetic price for clean test assertions.
- No `mint` / `burn` LP tokens — liquidity is held positionally by the contract; cannot be withdrawn. Re-deploy between test scenarios if needed.
- No `minAmountOut` slippage guard on `swap` — caller is presumed cooperative.
- No reentrancy guard — relies on the trusted `TestERC20` not making external calls in `transfer`.
- No price oracle / TWAP — spot price only; vulnerable to manipulation by design.

---

## TestPriceOracle.sol — settable Chainlink-style oracle

Conforms to the minimum surface of Chainlink's `AggregatorV3Interface` consumers usually need (`latestAnswer`, `decimals`). The price is a public setter so tests can move it arbitrarily between steps.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// @title TestPriceOracle
/// @notice Chainlink-style price oracle stub with a permissionless setter.
///         Tests CAN and SHOULD manipulate the price via setAnswer between steps.
///         DO NOT DEPLOY TO MAINNET.
contract TestPriceOracle {
    int256 private _answer;
    uint8  public immutable decimals;
    uint256 public updatedAt;

    event AnswerUpdated(int256 indexed current, uint256 updatedAt);

    constructor(int256 initialAnswer, uint8 _decimals) {
        _answer = initialAnswer;
        decimals = _decimals;
        updatedAt = block.timestamp;
    }

    /// @notice Chainlink-compat: return the latest reported price.
    function latestAnswer() external view returns (int256) {
        return _answer;
    }

    /// @notice Permissionless setter — tests need to move the price freely.
    function setAnswer(int256 newPrice) external {
        _answer = newPrice;
        updatedAt = block.timestamp;
        emit AnswerUpdated(newPrice, block.timestamp);
    }

    /// @notice Subset of Chainlink's AggregatorV3Interface.latestRoundData for tests
    ///         that expect that shape. roundId / answeredInRound are stubbed to 1.
    function latestRoundData()
        external
        view
        returns (uint80 roundId, int256 answer, uint256 startedAt, uint256 updatedAt_, uint80 answeredInRound)
    {
        return (1, _answer, updatedAt, updatedAt, 1);
    }
}
```

What this intentionally omits compared to a real Chainlink feed:
- No multi-round history — `getRoundData(roundId)` is not implemented.
- No staleness check — `updatedAt` advances only when `setAnswer` is called.
- No access control on `setAnswer` — by design.
- No heartbeat / deviation thresholds.

---

## Build commands

Both toolchains drop a compiled artifact containing the deploy bytecode. The Cadence side reads the bytecode as a hex string and passes it to `coa.deploy(...)`. The build-chain setup (Hardhat config, `--legacy` flag, etc.) is documented once in `flow-dev-setup/references/evm-tooling.md` — only the commands relevant to producing the bytecode are repeated here.

### Hardhat

```bash
# From the Hardhat project root with the three .sol files under contracts/
npx hardhat compile

# Bytecode for each contract lives at:
#   artifacts/contracts/TestERC20.sol/TestERC20.json        -> .bytecode
#   artifacts/contracts/TestAMM.sol/TestAMM.json            -> .bytecode
#   artifacts/contracts/TestPriceOracle.sol/TestPriceOracle.json -> .bytecode
```

Extract the bytecode hex for a Cadence test fixture:

```bash
jq -r '.bytecode' artifacts/contracts/TestERC20.sol/TestERC20.json > TestERC20.bin
```

### Foundry

```bash
# From the Foundry project root with the .sol files under src/
forge build

# Bytecode lives at out/<Contract>.sol/<Contract>.json under .bytecode.object
jq -r '.bytecode.object' out/TestERC20.sol/TestERC20.json > TestERC20.bin
```

Both pipelines produce a `.bin` file containing a `0x`-prefixed hex string of the deploy bytecode plus any constructor-arg footprint (for constructor-less builds — pass args via the Cadence-side encoder; see below).

---

## Deploying from a Cadence test transaction

`coa.deploy(...)` is the COA's contract-deploy entry point. It mirrors EVM `CREATE`: the deployed address is deterministic from the COA's EVM address and its nonce, and the call returns an `EVM.Result` whose `deployedContract` field carries the new `EVMAddress`. Capture this address at deploy time and reuse it for the lifetime of the test file.

```cadence
import "EVM"

/// Deploys a single EVM contract from the test signer's COA.
/// @param bytecodeHex   Hex string of deploy bytecode (with or without "0x" prefix).
/// @param constructorArgs ABI-encoded constructor args, or empty if none.
/// @return The deployed EVM contract address.
transaction(bytecodeHex: String, constructorArgs: [UInt8]) {
    prepare(signer: auth(BorrowValue) &Account) {
        let coa = signer.storage.borrow<auth(EVM.Deploy) &EVM.CadenceOwnedAccount>(
            from: /storage/evm
        ) ?? panic("No COA at /storage/evm")

        // Strip "0x" if present; decode to bytes.
        let stripped = bytecodeHex.slice(from: 0, upTo: 2) == "0x"
            ? bytecodeHex.slice(from: 2, upTo: bytecodeHex.length)
            : bytecodeHex
        let bytecode = stripped.decodeHex()

        // Append constructor args (already ABI-encoded).
        let payload = bytecode.concat(constructorArgs)

        let result = coa.deploy(
            code: payload,
            gasLimit: 4_000_000,
            value: EVM.Balance(attoflow: 0)
        )
        assert(
            result.status == EVM.Status.successful,
            message: "Deploy failed: ".concat(result.errorMessage)
        )
        assert(result.deployedContract != nil, message: "No deployedContract on result")

        log("Deployed at: ".concat(result.deployedContract!.toString()))
    }
}
```

`auth(EVM.Deploy)` is the entitlement required to call `coa.deploy`. A plain `&EVM.CadenceOwnedAccount` is not sufficient.

### Encoding constructor args from Cadence

`TestERC20`'s constructor takes `(string, string, uint8)`. `TestAMM`'s constructor takes `(address, address)`. Encode them with `EVM.encodeABI`:

```cadence
// TestERC20("Test USDC", "tUSDC", 6)
let erc20Args = EVM.encodeABI(["Test USDC", "tUSDC", UInt8(6)])

// TestAMM(tokenA, tokenB)
let ammArgs = EVM.encodeABI([tokenAAddress, tokenBAddress])

// TestPriceOracle(1_50000000, 8)   // initial answer = $1.50 with 8 decimals
let oracleArgs = EVM.encodeABI([Int256(150_000_000), UInt8(8)])
```

Constructor args are appended to the bytecode payload BEFORE passing to `coa.deploy`. The example transaction above does this with `bytecode.concat(constructorArgs)`.

---

## Capturing addresses in a Cadence Test file

The address layout pattern for a `_test.cdc` file is: deploy every stub once in `setup()`, store the addresses on the test contract or as module-level `let` bindings inside the test, then reference them from individual `test*` functions.

```cadence
import Test

access(all) let admin = Test.createAccount()

// Captured at setup time and re-used by every test function.
access(all) var tokenA: String = ""
access(all) var tokenB: String = ""
access(all) var amm: String = ""
access(all) var oracle: String = ""

access(all) fun setup() {
    // 1. Create the COA on admin.
    let createCoaTx = Test.Transaction(
        code: createCoaTxCode,
        authorizers: [admin.address],
        signers: [admin],
        arguments: []
    )
    let r0 = Test.executeTransaction(createCoaTx)
    Test.expect(r0, Test.beSucceeded())

    // 2. Deploy each EVM contract; capture the returned address.
    tokenA = deployEvm(bytecode: tokenABytecode, ctorArgs: tokenACtor)
    tokenB = deployEvm(bytecode: tokenBBytecode, ctorArgs: tokenBCtor)
    amm    = deployEvm(bytecode: ammBytecode,    ctorArgs: ammCtor(tokenA, tokenB))
    oracle = deployEvm(bytecode: oracleBytecode, ctorArgs: oracleCtor)
}

access(all) fun testSwapHonorsConstantProduct() {
    // Use `tokenA`, `tokenB`, `amm`, `oracle` directly — addresses are stable.
    // ...
}
```

`deployEvm(...)` is a helper that runs the transaction above and returns the deployed address string by reading `result.deployedContract` from the transaction events. The Cadence Test framework's `Test.eventsOfType` is the typical extraction point; see `cadence-testing/references/events-and-logs.md`.

---

## Address determinism and CREATE2 (optional)

`coa.deploy` uses EVM `CREATE` semantics: the deployed address is `keccak256(rlp([senderAddress, senderNonce]))[12:]`. Inside the deterministic emulator with a fresh COA, the address sequence is fully reproducible — the first deploy from a freshly created COA always lands at the same address. This is what makes the capture-in-`setup()` pattern reliable.

Flow EVM also supports `CREATE2` for byte-identical addresses across COA-nonce changes, but `coa.deploy` does not take a salt — you have to deploy a small factory and invoke it via `coa.call`. Rarely worth it for emulator-only tests.

---

## Reset semantics inside a test file

The Cadence Test framework's `Test.reset(to: blockHeight)` rolls the entire chain — both Cadence state AND Flow EVM state — back to the snapshot at `blockHeight`. Concretely:

- If `setup()` deploys at block heights 1, 2, 3, 4 and you capture the heights in `let setupHeight = getCurrentBlock().height` after the deploys, then `Test.reset(to: setupHeight)` from inside a `test*` function returns the chain to the state where all four EVM contracts are still deployed.
- The captured `tokenA` / `tokenB` / `amm` / `oracle` address strings remain valid pointers — they refer to EVM accounts that exist at any block ≥ the deploy height.
- You do NOT need to redeploy in each test. Wire the snapshot to the post-`setup` state and reset back to it between tests:

```cadence
access(all) var setupHeight: UInt64 = 0

access(all) fun setup() {
    // ... deploy stubs ...
    setupHeight = getCurrentBlock().height
}

access(all) fun testFirstScenario() {
    Test.reset(to: setupHeight)
    // tokenA/tokenB/amm/oracle still resolve correctly.
}

access(all) fun testSecondScenario() {
    Test.reset(to: setupHeight)
    // Same EVM layout, fresh state for assertions.
}
```

Redeploying in every `test*` is a common mistake — it wastes test time and breaks address determinism (each deploy bumps the COA nonce, so the addresses captured in `setup()` no longer match the ones produced by a re-deploy). Capture once, reset between tests.

---

## Common pitfalls

1. **Believing the test stubs are safe enough for testnet.** They are not. `mint` is unauthenticated. A testnet deploy of `TestERC20` is a public infinite faucet that anyone can drain. Use the stubs ONLY inside the emulator (`flow emulator` or `Test.executeTransaction`) and never in `flow project deploy --network testnet`.

2. **Forgetting that `coa.deploy` requires `auth(EVM.Deploy)`.** A reference borrowed with `auth(EVM.Call)` only is not authorized to deploy. The borrow line must explicitly include the `Deploy` entitlement, e.g. `auth(EVM.Call, EVM.Deploy) &EVM.CadenceOwnedAccount`.

3. **Redeploying in every `test*` function.** Captures from `setup()` survive `Test.reset(to: setupHeight)`; redeploying invalidates them and slows the suite. Treat the post-setup snapshot as the test entry point.

4. **Copy-pasting `mint(address,uint256)` without access control into production code.** This is the highest-impact failure mode of using test fixtures as a starting point. If the production contract evolves from the stub, replace `mint` with `onlyOwner` (OpenZeppelin `Ownable`) or `AccessControl`-gated semantics BEFORE the first non-test deploy.

5. **Confusing fixture addresses across test files.** Each `_test.cdc` runs in its own chain instance — the addresses captured in `testFoo_test.cdc::setup()` are not visible to `testBar_test.cdc`. Do not export addresses between files; deploy fresh in each file's `setup()`.

6. **Assuming `getPrice` in the AMM stub is manipulation-resistant.** It is a spot read of the current reserves. Tests that want to assert "price moved by X" should `swap` into the pool first; tests that want a fixed oracle value should use the `TestPriceOracle` stub, not `TestAMM.getPrice`.

7. **Skipping `result.status` after `coa.deploy`.** Same rule as `coa.call`: a failed deploy does NOT abort the surrounding Cadence transaction. Always assert `result.status == EVM.Status.successful` and `result.deployedContract != nil` before reading the address.

8. **Hard-coding the bytecode hex inside the Cadence test source.** Bytecode is large and changes every Solidity compile. Read the `.bin` file at test load time (e.g. via the `flow.json` deployment script that prepares fixtures) and pass it as a transaction argument.

---

## See also

- [evm-call.md](evm-call.md) — `coa.call` / `EVM.dryCall` mechanics for invoking these stubs.
- [erc20-read.md](erc20-read.md) — read patterns (`balanceOf`, `allowance`, `decimals`) against the `TestERC20` stub.
- [coa-lifecycle.md](coa-lifecycle.md) — creating the COA that owns the deployed fixtures.
- `flow-dev-setup/references/evm-tooling.md` — Hardhat and Foundry setup for Flow EVM.
- `cadence-testing/references/setup-and-basics.md` — `setup()` / `beforeEach` / `Test.reset` discipline.
- `cadence-testing/references/events-and-logs.md` — extracting `deployedContract` from transaction events.
