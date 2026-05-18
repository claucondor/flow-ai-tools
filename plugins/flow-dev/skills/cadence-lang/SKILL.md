---
name: cadence-lang
description: |
  Comprehensive guide for writing correct, secure, and idiomatic Cadence smart contract code on the Flow blockchain. Covers language fundamentals (resources, contracts, transactions, interfaces, accounts, references, imports), access control and entitlements, capabilities, pre/post conditions, security best practices, anti-patterns to avoid, and proven design patterns.
  TRIGGER when: writing or debugging Cadence code, asking about Cadence syntax, access(self), access(all), entitlements, resources, move operator (<-), capabilities, references, pre/post conditions, storage paths, "how do I write cadence", "cadence error", "compile error in .cdc", "what does access(self) mean", "how do resources work", "capability-based security", randomness, revertibleRandom, RandomBeaconHistory, RandomConsumer, Xorshift128plus, commit-reveal, "random number in cadence", "how to generate random in flow", scheduled transactions, FlowTransactionScheduler, TransactionHandler interface, callback resource, "schedule a callback", "on-chain automation", "Forte upgrade", resource state machines, phase enum, multi-tx escrow, custody, "hold funds across transactions", "intent escrow", "async settlement", Fix128, UFix128, "128-bit fixed point", "fixed-point precision", "UFix64 vs UFix128", "high-precision math Cadence", "AMM math precision", event naming conventions, "Cadence event taxonomy", admin capability pattern, "admin via capability", "modern admin pattern", strategy registry, "strategy whitelist", "approved strategies", "intent executor pattern", "capability registry", "Cadence Crypto API", "signature verification Flow", "BLS aggregation Cadence", "KMAC128", "PublicKey.verify", "SignatureAlgorithm", "HashAlgorithm Cadence", "Crypto.KeyList", "verifyPoP", "proof of possession BLS", "secp256k1 Cadence", "P-256 Cadence", "WebAuthn Flow".
  DO NOT TRIGGER when: building NFT/FT token contracts (use cadence-tokens), setting up flow.json or FCL (use flow-project-setup), reviewing existing code for vulnerabilities (use cadence-audit), generating new contracts from scratch (use cadence-scaffold).
---

# Cadence Language Guide

Write secure, correct Cadence code by following these rules. Cadence uses resource-oriented programming, capability-based security, and explicit access control.

## Key Principles

1. **Secure by default** — start with `access(self)`, expand only when needed
2. **Resource safety** — resources exist in one place, must be explicitly moved (`<-`) or destroyed
3. **Capability-based security** — access delegated through unforgeable, revocable capabilities
4. **Explicit over implicit** — force developers to make security decisions
5. **Document generated code** — follow the Cadence documentation conventions in the relevant reference docs

## Navigation Map

Read the relevant reference file based on your task:

| Task | Reference |
|------|-----------|
| Import patterns, flow.json setup | [imports.md](references/imports.md) |
| Resource lifecycle, move operator | [resources.md](references/resources.md) |
| Contract structure, init, deployment | [contracts.md](references/contracts.md) |
| Transaction phases, entitlements | [transactions.md](references/transactions.md) |
| Documentation conventions | [documentation.md](references/documentation.md) |
| Interfaces, intersection types | [interfaces.md](references/interfaces.md) |
| Account storage, keys, capabilities | [accounts.md](references/accounts.md) |
| References, authorized refs | [references.md](references/references.md) |
| Access modifiers, visibility rules | [access-control.md](references/access-control.md) |
| Entitlements, mappings, Identity, sets | [entitlements.md](references/entitlements.md) |
| Capabilities, security model | [capabilities.md](references/capabilities.md) |
| Pre/post conditions, `before()` | [conditions.md](references/conditions.md) |
| Security best practices | [security-best-practices.md](references/security-best-practices.md) |
| Anti-patterns to avoid | [anti-patterns.md](references/anti-patterns.md) |
| Design patterns | [design-patterns.md](references/design-patterns.md) |
| CU cost by operation, optimization patterns, MAX_SAFE_N methodology | [cu-optimization.md](references/cu-optimization.md) |
| Randomness APIs (revertibleRandom, beacon, RandomConsumer, commit-reveal) | [randomness.md](references/randomness.md) |
| Scheduled transactions (`FlowTransactionScheduler`, callback resource, fees, failure handling) | [scheduled-transactions.md](references/scheduled-transactions.md) |
| Resource state machines (phase enum, entitled transitions, pre/post guards) | [resource-state-machines.md](references/resource-state-machines.md) |
| Multi-tx escrow / custody (fund-holding resources, recovery paths, two-phase commit) | [multi-tx-escrow.md](references/multi-tx-escrow.md) |
| 128-bit fixed-point types (`Fix128`/`UFix128`, precision, overflow, conversions) | [numeric-fixed-point.md](references/numeric-fixed-point.md) |
| Event taxonomy (naming conventions, required defaults, argument typing) | [event-taxonomy.md](references/event-taxonomy.md) |
| Admin-via-capability (modern admin role pattern, migration from legacy) | [admin-via-capability.md](references/admin-via-capability.md) |
| Strategy registry (capability whitelists, intent execution, lifecycle states) | [strategy-registry.md](references/strategy-registry.md) |
| Cadence Crypto API (SignatureAlgorithm, HashAlgorithm, PublicKey.verify, BLS aggregation, KMAC128, KeyList) | [crypto.md](references/crypto.md) |

For security-sensitive tasks, also read `security-best-practices.md` and `anti-patterns.md`.

## Companion Skills

This skill provides the language foundation. Other skills build on it:

- **`cadence-testing`** — Use alongside when writing tests for Cadence code. Tests are Cadence too and must follow every rule in this skill.
- **`cadence-tokens`** — Use alongside this skill when building NFT/FT contracts. Token contracts must follow all rules here plus token-specific standards.
- **`cadence-audit`** — Use to verify code follows the security rules and patterns documented here.
- **`cadence-scaffold`** — Use to generate contracts/transactions that follow these rules by default.
- **`flow-cli`** — Use to deploy and test the Cadence code written with this skill's guidance.
- **`flow-react-sdk`** — Use when the Cadence scripts/transactions will be called from React hooks.
