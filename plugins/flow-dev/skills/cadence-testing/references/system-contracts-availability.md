# System Contracts Availability in `flow test`

On Flow CLI v2.17.1, the in-process test runner (`flow test`) bootstraps a full chain that
includes all system contracts. Every contract previously documented as "NOT available in
`flow test`" — including `RandomBeaconHistory`, `EVM`, consensus contracts, and service
infrastructure — is auto-bootstrapped and importable without any `flow.json` setup or manual
`Test.deployContract` call. The legacy two-layer strategy driven by contract availability
concerns is no longer necessary on v2.17.1.

The remaining concerns for testing are different from availability:
1. `RandomBeaconHistory` beacon entries recorded before a `Test.reset()` are removed for
   heights after the reset target and are NOT re-populated on the second forward pass. See
   [`test-reset-caveats.md`](test-reset-caveats.md) for the failure mode and workarounds.
2. Heartbeat semantics in the in-process framework may differ from production in subtle ways
   (tick timing, beacon seed derivation). Verify before depending on exact tick-driven
   behavior in your tests.

---

## What Is Available in `flow test` (v2.17.1)

All contracts below are bootstrapped automatically by the in-process test framework on the
`flow-emulator-monotonic` chain. They do not need `flow dependencies install` and must not
be deployed manually via `Test.deployContract` — the framework already provisioned them.

| Contract | Notes |
|---|---|
| `NonFungibleToken`, `MetadataViews`, `ViewResolver` | Service account bootstrap |
| `FungibleToken` | System bootstrap |
| `FlowToken` | System bootstrap |
| `FlowFees` | System bootstrap |
| `Burner` | Auto-loads on import in v2.17.1 — no `flow.json` alias required. Confirmed on CLI v2.17.1. |
| `FungibleTokenMetadataViews` | Auto-loads on import in v2.17.1 — no `flow.json` alias required. Confirmed on CLI v2.17.1. |
| `RandomBeaconHistory` | Bootstrapped with automatic heartbeat active — beacon entries are recorded per block without manual setup. Confirmed on CLI v2.17.1. |
| `EVM` | Cross-VM environment is available in-process on v2.17.1. |
| `FlowEpoch`, `FlowDKG`, `FlowClusterQC` | Consensus contracts auto-bootstrap on v2.17.1. |
| `FlowStorageFees`, `FlowServiceAccount` | Service infrastructure auto-bootstraps on v2.17.1. |

Do NOT call `Test.deployContract` on any of these contracts — they are already provisioned.
A duplicate deploy attempt fails with `account with address 0x000000000000000X not found`.

---

## Pre-v2.17.1 Behavior (Historical)

On older CLI versions, several contracts were absent from the in-process test bootstrap and
required manual setup. If you are on an older CLI, run `flow --version` and consider upgrading.

**Pre-v2.17.1: these contracts required manual setup; v2.17.1 auto-loads them all.**

| Contract | Reason absent in older versions |
|---|---|
| `RandomBeaconHistory` | Required `Heartbeat.heartbeat()` per block; no automatic system-chunk invocation in-process |
| `EVM` | Required a cross-VM execution environment |
| `FlowEpoch`, `FlowDKG`, `FlowClusterQC` | Consensus infrastructure |
| `FlowStorageFees`, `FlowServiceAccount` | Service infrastructure |

The manual deploy-and-populate workaround (installing source via `flow dependencies install`,
adding a `flow.json` testing alias, calling `Test.deployContract` in setup, and manually
calling `heartbeat()` per block) was the correct approach on older CLI versions. On v2.17.1
this entire workflow is obsolete for system contracts.

---

## Two-Layer Testing Pattern (Updated Rationale)

The two-layer strategy is still architecturally sound, but the motivation has changed in
v2.17.1.

**Layer 1 — `flow test` unit tests**
- Pure logic: arithmetic, state transitions, access control, event emission
- All system contracts available including `RandomBeaconHistory`, `EVM`, and consensus
- Fast, deterministic, CI-ready
- One limitation: beacon entries removed by `Test.reset()` are not re-recorded automatically
  on the second pass (see [`test-reset-caveats.md`](test-reset-caveats.md))

**Layer 2 — Shell-script integration tests against `flow emulator`**
- Full lifecycle tests that depend on exact production-equivalent beacon tick behavior
- Tests that verify Cadence ↔ EVM bridging under realistic infrastructure load
- Acceptable to be slower; run on a longer CI schedule or as a pre-release gate

```bash
# Layer 2 example: a shell script integration test
flow emulator &
EMULATOR_PID=$!
flow project deploy --network emulator
flow transactions send cadence/transactions/setup_consumer.cdc --network emulator
# ... run test transactions and assert via scripts ...
kill $EMULATOR_PID
```

---

## RandomBeaconHistory — Automatic Heartbeat Confirmed

On v2.17.1, the in-process test framework records beacon entries automatically per block.
`RandomBeaconHistory.sourceOfRandomness(atBlockHeight:)` succeeds without any manual
`heartbeat()` call or `Test.deployContract` setup.

```cadence
// ✅ This works on v2.17.1 without any manual beacon setup
import Test
import "RandomBeaconHistory"

access(all) fun testBeaconAvailable() {
    Test.commitBlock()
    Test.commitBlock()
    let height = getCurrentBlock().height
    let result = Test.executeScript(
        "import \"RandomBeaconHistory\"\naccess(all) fun main(h: UInt64): RandomBeaconHistory.RandomSource { return RandomBeaconHistory.sourceOfRandomness(atBlockHeight: h) }",
        [height]
    )
    Test.expect(result, Test.beSucceeded())
}
```

The remaining caveat is reset behavior, not availability: after `Test.reset(to: h)`, entries
for heights above `h` are removed and not re-populated automatically when the chain advances
past those heights again. See [`test-reset-caveats.md`](test-reset-caveats.md).

---

## Common Pitfalls

- **Calling `Test.deployContract` on bootstrapped system contracts** — All contracts in the
  table above are already provisioned. A duplicate deploy attempt fails with
  `account with address 0x000000000000000X not found`.
- **Expecting beacon entries to survive `Test.reset()`** — Entries for heights after the reset
  target are removed. `sourceOfRandomness()` panics with "Source of randomness not yet
  recorded" for those heights after reset. Use single-chain continuity for beacon-dependent
  tests. See [`test-reset-caveats.md`](test-reset-caveats.md).
- **Using standalone emulator addresses in testing aliases** — The standalone emulator uses
  linear-code addresses (e.g., `0xf8d6e0586b0a20c7` for the service account). The in-process
  test framework uses monotonic addresses (`0x0000000000000001`). Aliases must use the
  monotonic form in `flow.json`.
- **Applying pre-v2.17.1 workarounds unnecessarily** — If you find documentation or examples
  that deploy `RandomBeaconHistory` manually inside `flow test`, that pattern was required on
  older CLI versions. On v2.17.1 it is unnecessary and will fail with a duplicate-deploy error.

---

## See Also

- [`test-reset-caveats.md`](test-reset-caveats.md) — Why beacon values differ before and after
  `Test.reset(to:)` even when targeting the same block height; why entries are not re-recorded
  after reset
- [`cadence-lang/references/randomness.md`](../../cadence-lang/references/randomness.md) —
  `RandomBeaconHistory`, `RandomConsumer`, and the commit-reveal pattern
- [`blockchain-emulation.md`](blockchain-emulation.md) — Accounts, deploys, and the
  auto-load vs manual-deploy distinction
