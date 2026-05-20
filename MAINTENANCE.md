# Maintenance Notes

This file tracks (a) canonical assignments for known duplicate-claim clusters and (b) STALE-tagged entries that need version-aware re-verification.

## Canonical Assignments

When a pitfall has near-identical treatment across multiple files, ONE file owns the deep treatment ("canonical") and the others carry one-line pointers. Updates to behavior should land in the canonical file first.

| Cluster | Canonical file | Pointers in |
|---|---|---|
| `result.status` must be checked after `coa.call` | `plugins/flow-dev/skills/flow-crossvm/references/evm-call.md` (Pitfall 1) | `flow-bridge.md`, `cu-ceiling.md`, `audit-checklist.md`, `crossvm-anti-patterns.md` (C1 retained as audit-checklist version) |
| Publishing auth COA cap at `/public/evm` | `plugins/flow-dev/skills/flow-crossvm/references/coa-entitlements.md` (Anti-pattern C) | `coa-lifecycle.md` (AP1), `crossvm-anti-patterns.md` (C5) |
| Sharing one COA auth cap across consumers | `plugins/flow-dev/skills/flow-crossvm/references/coa-entitlements.md` (Anti-pattern B) | `coa-lifecycle.md` (AP2), `crossvm-anti-patterns.md` (C4) |
| `Test.reset(to: 0)` wipes accounts | `plugins/flow-dev/skills/cadence-testing/references/test-reset-caveats.md` | `blockchain-emulation.md`, `crossvm-integration-testing.md`, `setup-and-basics.md`, `multi-tx-state-testing.md` |
| Handler panic in scheduled tx = silent loss | `plugins/flow-dev/skills/cadence-audit/references/forte-anti-patterns.md` (A6) | `cadence-lang/references/scheduled-transactions.md`, `flow-actions/references/scheduled-integration.md` |
| Events as state (replaying events is lossy) | `plugins/flow-dev/skills/cadence-lang/references/event-taxonomy.md` (AP5) | `multi-tx-escrow.md` (AP2), `strategy-registry.md` |

## STALE Entries

The following entries are tagged with specific CLI/Cadence versions and require re-verification on the next Flow CLI major release. Each entry has an inline `<!-- stale: re-verify on next Flow CLI major -->` HTML comment marker added next to it.

| Entry | File | Version-tag | Re-verify trigger |
|---|---|---|---|
| BN254 verifyProof encoding pitfall | `cadence-testing/references/crossvm-integration-testing.md` Pitfall 11 | "(v2.17.1)" inline; "verified empirically on 2026-05-19" | Re-verify on Flow CLI v2.18.0+ |
| EVM events logs field empty | `cadence-testing/references/crossvm-integration-testing.md` Pitfall 9 | "empirically (v2.17.1)" inline | Re-verify on Flow CLI v2.18.0+ |
| TopShot Dec-27-2025 contract-initializer | `cadence-lang/references/anti-patterns.md` AP5 | "patched in Cadence v1.8.9" (reframed as historical) | Verify it remains historical after v2.x; if any regression, re-elevate |
| Flow CLI schedule setup bug | `flow-cli/references/known-bugs.md` (extracted in this iteration) | "v2.17.1" | Remove once upstream PR lands |

## Last Audit

Date: 2026-05-19
Source report: `/home/oydual3/flow-ai-tools-drafts/pitfalls-audit/REPORT.md`

Next recommended audit: when total pitfall entries across the repo exceeds 200, OR after a Flow CLI major version bump.
