---
name: flow-crossvm
description: |
  Guide for building CrossVM applications on Flow — Cadence transactions and scripts that read from or call into Flow EVM atomically. Covers the Cadence Owned Account (COA) pattern, `EVM.call` / `coa.call` mechanics, ABI encoding and decoding from Cadence, reading ERC20 storage from Cadence scripts, native FLOW bridging between Cadence and EVM, the 9999 CU per-transaction ceiling shared across both VMs, and Solidity test fixtures for integration testing.
  TRIGGER when: writing CrossVM transactions or scripts, "EVM.call", "coa.call", "coa.borrow", "COA", "Cadence Owned Account", "EVM on Flow", "EVM contract from Cadence", "ERC20 from Cadence", "ERC20 balance Cadence", "call EVM contract", "FLOW bridge Cadence to EVM", "bridge FLOW from Cadence", "EVM gas Flow", "CrossVM", "cross-VM", "ABI encode Cadence", "decode EVM response", "EVM.encodeABI", "EVM.decodeABI", "EVM.dryCall", "9999 CU CrossVM", "CrossVM CU ceiling", "atomic EVM call from Cadence", "escrow via COA", "COA custody", "result.status EVM".
  DO NOT TRIGGER when: writing Cadence syntax in isolation (use `cadence-lang`), designing DeFi architecture without an EVM bridge component (use `flow-defi`), building React frontend EVM hooks like `useCrossVmBatchTransaction` (use `flow-react-sdk`), auditing existing CrossVM code for vulnerabilities (use `cadence-audit`), generating brand-new CrossVM transactions from scratch (use `cadence-scaffold`).
---

# Flow CrossVM

Build atomic Cadence + EVM transactions on Flow. CrossVM lets a single Cadence transaction call EVM contracts, read EVM storage, transfer native FLOW between the two execution environments, and revert both sides together — without bridges, relayers, or cross-chain messaging.

Flow CrossVM is structurally different from cross-chain bridges on other networks: both VMs share the same block, the same finality, and the same fee budget. A Cadence transaction calling an EVM contract is one atomic unit — there is no asynchronous message, no challenge period, and no MEV opportunity between the two halves. Reach for this skill when a transaction needs to touch EVM state from Cadence (call an EVM contract, read ERC20 balances, move FLOW across the VM boundary, or hold EVM assets in a Cadence-controlled account).

## When to Use This Skill

Use `flow-crossvm` when the task involves the **boundary** between Cadence and EVM. Typical signals:

- A Cadence transaction borrows an `auth(EVM.Call) &EVM.CadenceOwnedAccount` reference.
- A Cadence script reads ERC20 state (`balanceOf`, `allowance`, `decimals`, `totalSupply`).
- A transaction moves native FLOW from a Cadence vault into an EVM address (or vice versa).
- The protocol uses a COA as an escrow vault, a per-user EVM identity, or a programmatic EVM operator.
- The transaction makes one or more `coa.call` / `EVM.dryCall` invocations and must reason about their results.

If the work is purely Cadence (no EVM interaction) or purely Solidity (no Cadence side), use `cadence-lang` or a Solidity-specific tool instead.

## Key Principles

1. **Atomicity is at the Cadence layer, not the EVM layer.** If Cadence panics, both Cadence and EVM state revert. If the EVM call reverts but Cadence does not panic, only the EVM side reverts and the Cadence transaction still commits its other state changes.
2. **Always check `result.status`.** `coa.call` returns an `EVM.Result` — a failed EVM call does NOT automatically abort the surrounding Cadence transaction. Code must inspect `result.status` and panic explicitly when the EVM side must succeed. Treat unchecked `EVM.Result` values as a Cadence anti-pattern equivalent to swallowing a thrown exception.
3. **The 9999 CU per-transaction ceiling is shared across both VMs.** Cadence opcodes, EVM gas, ABI encoding/decoding, and `coa.call` overhead all draw from the same budget. CrossVM transactions hit the ceiling sooner than pure-Cadence transactions and require deliberate batching.
4. **`gasLimit` is per-EVM-call, not per-transaction.** Each `coa.call` declares its own EVM gas limit. Setting it too low causes a silent EVM failure that only surfaces if `result.status` is checked. Pick `gasLimit` based on the EVM function's worst-case, not its average.
5. **COAs are user-owned EVM addresses.** A Cadence Owned Account lives in Cadence account storage as an `EVM.CadenceOwnedAccount` resource and has a deterministic EVM address. Custody, escrow, and per-user EVM identity flow through this single primitive — there is no separate keypair or signature scheme on the EVM side.
6. **Use `EVM.dryCall` for read paths.** Scripts and view-only logic should never use a state-mutating `coa.call`. `EVM.dryCall` runs the EVM call without committing state, returning the same `EVM.Result` shape for decoding.

## Navigation Map

Read the relevant reference based on the task:

| Task | Reference |
|------|-----------|
| Canonical `coa.call` and `EVM.dryCall` pattern, calldata layout, `result.status` handling, revert decoding | [evm-call.md](references/evm-call.md) |
| Reading ERC20 storage (`balanceOf`, `allowance`, `decimals`, `totalSupply`) from Cadence scripts | [erc20-read.md](references/erc20-read.md) |
| COA lifecycle (`createCadenceOwnedAccount`, storage path, custody, escrow-via-COA, deposit/withdraw) | [coa-lifecycle.md](references/coa-lifecycle.md) |
| 9999 CU ceiling specific to CrossVM, measurement methodology, batching strategies, when to split a tx | [cu-ceiling.md](references/cu-ceiling.md) |
| Native FLOW bridge between Cadence and EVM (`coa.deposit` / `coa.withdraw`), atomicity guarantees, fees | [flow-bridge.md](references/flow-bridge.md) |
| Minimal Solidity test fixtures (ERC20, AMM, oracle stubs) for integration tests against `coa.call` | [solidity-fixtures.md](references/solidity-fixtures.md) |
| CrossVM empirical explorations: 10 measured CU formulas, 10 named patterns, 7 decision trees, async-intent insights | [explorations.md](references/explorations.md) |

For security-sensitive work — especially anything that touches user funds across the VM boundary — pair this skill with `cadence-audit`.

## Companion Skills

- **`cadence-lang`** — Use alongside for the underlying Cadence rules. Every CrossVM transaction is a Cadence transaction first; access control, entitlements (`auth(EVM.Call) &EVM.CadenceOwnedAccount`), and resource semantics still apply.
- **`flow-defi`** — Use when the CrossVM transaction is part of a larger DeFi protocol architecture. `flow-defi` covers the *why* (MEV-free EVM, composability), this skill covers the *how* (concrete `coa.call` patterns).
- **`flow-react-sdk`** — Use for the frontend side: `useCrossVmBatchTransaction`, `useCrossVmTokenBalance`, and `useCrossVmTransactionStatus` hooks consume the Cadence transactions documented here.
- **`cadence-audit`** — Use to review CrossVM transactions for the common pitfalls: unchecked `result.status`, gas-exhaustion silent failures, and CU-ceiling overruns.
- **`cadence-scaffold`** — Use to generate new CrossVM transactions from scratch following the patterns in this skill.

---

