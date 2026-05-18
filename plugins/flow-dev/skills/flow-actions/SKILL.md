---
name: flow-actions
description: |
  Guide for building composable, protocol-agnostic DeFi workflows on the Flow blockchain using the DeFiActions framework. Covers the four core struct interfaces — Source, Sink, Swapper, and PriceOracle — their method signatures (minimumAvailable, minimumCapacity, quoteIn/quoteOut, swap, swapBack, price), UniqueIdentifier traceability, atomic Source → Swapper → Sink composition patterns, the Weak Guarantees philosophy, and how to combine DeFi Actions strategies with FlowTransactionScheduler for automated on-chain execution.
  TRIGGER when: "Source interface", "Sink interface", "Swapper interface", "PriceOracle interface", "DeFiActions", "DeFiActions composition", "DeFi Actions", "Source Sink Swapper", "atomic DeFi composition", "modular DeFi Flow", "intent executor pattern", "DeFi Actions standalone", "Source -> Swapper -> Sink", "scheduled DeFi tick", "FlowTransactionScheduler with strategy", "minimumAvailable", "minimumCapacity", "depositCapacity", "withdrawAvailable", "quoteIn", "quoteOut", "swapBack", "IdentifiableStruct", "UniqueIdentifier DeFi", "DeFiActions.createUniqueIdentifier", "ComponentInfo", "DeFiActions contract", "money LEGOs Flow", "connector composition", "SwapSource", "SwapSink", "VaultSource", "VaultSink", "AutoBalancer DeFi".
  DO NOT TRIGGER when: writing Cadence syntax in isolation (use `cadence-lang`), designing DeFi architecture without composing actions (use `flow-defi`), scaffolding an IncrementFi-coupled DeFi transaction from scratch (use `cadence-scaffold`), writing the FlowTransactionScheduler callback resource itself (use `cadence-lang`'s scheduled-transactions reference), building a React frontend that consumes a composition (use `flow-react-sdk`).
---

# Flow Actions

The DeFiActions framework is a set of composable, protocol-agnostic struct interfaces that act as glue between standard DeFi primitives — DEXes, lending pools, farms — on the Flow blockchain. Rather than embedding IncrementFi or any other protocol into transaction logic directly, developers implement four lightweight interfaces (Source, Sink, Swapper, PriceOracle) and compose them into atomic workflows within a single Cadence transaction. This skill covers those interfaces and their composition at the abstract level; for IncrementFi-coupled scaffolding of a specific restake transaction, use `cadence-scaffold` instead. The two skills are complementary: `flow-actions` teaches the interface contract and composition rules; `cadence-scaffold` generates concrete, protocol-specific transaction code that follows those rules.

> **Beta notice:** `DeFiActions` is in beta on Testnet and Mainnet. Interfaces may change before final release. Monitor [`onflow/FlowActions`](https://github.com/onflow/FlowActions) for breaking changes.

## Key Principles

1. **Single `UniqueIdentifier` across all connectors in a composition.** Create one `uniqueID` via `DeFiActions.createUniqueIdentifier()` and pass it to every Source, Swapper, and Sink in the chain. This ID surfaces in every emitted event (`Withdrawn`, `Swapped`, `Deposited`), enabling full end-to-end traceability of a single operation.
2. **Validate vault empty (`balance == 0.0`) before destruction.** After `depositCapacity` is called on a Sink, assert that the residual vault has `balance == 0.0` before `destroy`ing it. Residual tokens indicate a logic error in the composition or a capacity mismatch.
3. **Nil-check every borrowed capability.** All connector constructors accept Capabilities; borrow them in `prepare` with `?? panic(...)` rather than force-unwrapping. A missing capability aborts the transaction before any vault movement occurs.
4. **Atomicity is at the Cadence transaction boundary.** All connector calls happen within one Cadence transaction — either everything executes or the entire transaction reverts. There is no partial-success state. Compose accordingly: if the Sink cannot accept the full output of the Swapper, the transaction should revert rather than leave funds stranded.

## Navigation Map

| Task | Reference |
|------|-----------|
| Source interface: `getSourceType`, `minimumAvailable`, `withdrawAvailable` signature, weak-guarantee semantics, type validation requirement | [source-interface.md](references/source-interface.md) |
| Sink interface: `getSinkType`, `minimumCapacity`, `depositCapacity` signature, liveness philosophy, residual-vault pattern | [sink-interface.md](references/sink-interface.md) |
| Swapper interface: `inType`/`outType`, `quoteIn`/`quoteOut`, `swap`/`swapBack` signatures, Quote struct, reverse-flag semantics | [swapper-interface.md](references/swapper-interface.md) |
| PriceOracle interface: `unitOfAccount`, `price(ofToken:)` signature, optional-return semantics, adapter pattern for Band/ERC4626 | [price-oracle-interface.md](references/price-oracle-interface.md) |
| Composing Source → Swapper → Sink atomically: SwapSource/SwapSink wrappers, token-order reversal, `minimumAvailable`/`minimumCapacity` sizing | [composition-patterns.md](references/composition-patterns.md) |
| Integrating a DeFi Actions strategy with FlowTransactionScheduler: AutoBalancer pattern, scheduled-tick callback, per-tick CU budget | [scheduled-integration.md](references/scheduled-integration.md) |

## When to Use This Skill

Use the following decision tree to choose the right skill:

- **You need to understand or implement a Source, Sink, Swapper, or PriceOracle interface from scratch** → `flow-actions` (this skill).
- **You need to compose two or more connectors into an atomic workflow** → `flow-actions` + optionally `cadence-lang` for resource safety rules.
- **You need to build an intent executor that fires on a schedule** → `flow-actions` (`scheduled-integration.md`) + `cadence-lang` (`scheduled-transactions.md`).
- **You need a one-off IncrementFi restake transaction generated for you** → `cadence-scaffold` (uses `scaffold-defi.md`; IncrementFi-coupled).
- **You need to design the overall DeFi protocol architecture** → start with `flow-defi`, then return here to compose primitives via `flow-actions`.
- **You need to audit a composed workflow for security pitfalls** → `cadence-audit` + `flow-actions` for context on what the interfaces guarantee.

## Companion Skills

- **`cadence-lang`** — Every connector is a Cadence struct; resource safety, entitlements (`access(FungibleToken.Withdraw)`), and pre/post condition rules all apply. Consult `cadence-lang` for language fundamentals and `scheduled-transactions.md` for the FlowTransactionScheduler callback resource interface.
- **`flow-defi`** — Use for protocol-level architectural decisions (AMM type selection, liquidity strategy, MEV-free EVM advantages) before wiring up connectors with `flow-actions`.
- **`cadence-scaffold`** — Use to generate IncrementFi-coupled DeFi transactions from scratch. Those transactions follow the composition rules documented here; `cadence-scaffold` handles the concrete protocol wiring.
- **`cadence-audit`** — Use to review composed workflows for common pitfalls: unchecked residual vaults, missing nil-checks, UniqueIdentifier misalignment, and CU-ceiling overruns in multi-connector chains.
- **`cadence-testing`** — Use to write unit tests for connector implementations and composition workflows, including time-mocked scheduled rebalance tests.

---

