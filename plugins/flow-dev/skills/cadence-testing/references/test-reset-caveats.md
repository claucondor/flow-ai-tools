# Test.reset() Caveats

`Test.reset(to: height)` is a powerful tool for giving each test a clean fixture without
redeploying contracts. It restores the blockchain to a previously-observed height — but
"restore" applies only to Cadence storage and block height. It does not reproduce entropy.
Any system state that depends on per-block randomness seeding will silently diverge between
the pre-reset and post-reset chains even when the block height appears identical. Tests that
record a random beacon value, reset, advance to the same height, and expect the same value
will fail non-deterministically or consistently, depending on the source.

This reference extends the [setup-then-reset trap](blockchain-emulation.md#the-setup-then-reset-trap)
in `blockchain-emulation.md` with the specific failure mode around beacon history and other
per-block entropy sources.

---

## What `Test.reset(to:)` Does and Does Not Restore

| What is restored | What is NOT restored |
|---|---|
| Cadence contract storage at the target height | `RandomBeaconHistory` beacon values for heights after the target |
| Account balances at the target height | Internal RNG seeds of contracts dependent on per-block entropy |
| Deployed contract code at the target height | EVM state (rolled back together with Cadence state, but re-execution entropy differs) |
| Block height counter | The exact sequence of future beacon entries |
| Event log (cleared) | |

The key rule: **block height is rewound; block entropy is not replayed identically.**

The reason is that the test framework's in-process emulator generates per-block randomness
from an internal source that advances monotonically through the test process's lifetime. After
a reset, the internal entropy source continues from where it was — it does not rewind to the
same seed it had at the target height during the first pass. The exact Go-level mechanism was
not inspected. Empirically confirmed on v2.17.1: `revertibleRandom<UInt64>()` returns
different values before and after `Test.reset()` at the same block height. Additionally,
`RandomBeaconHistory` entries for heights after the reset target are removed and not
re-populated automatically on the second pass — `sourceOfRandomness()` panics with
"Source of randomness not yet recorded" for those heights.

---

## The Failure Pattern

```cadence
import Test
import BlockchainHelpers
import "RandomBeaconHistory"

// ❌ This test FAILS — beacon values differ after reset

access(all) fun testBeaconDeterminism() {
    // 1. Record a beacon entry and capture its value.
    let source: [UInt8] = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16]
    recordBeacon(source: source)  // advances chain to height N

    let heightN = getCurrentBlockHeight()
    let beaconBefore = RandomBeaconHistory.sourceOfRandomness(atBlockHeight: heightN).value

    // 2. Reset to before the beacon was recorded.
    Test.reset(to: heightN - 1)

    // 3. Re-record a beacon at the same height.
    recordBeacon(source: source)  // chain is now back at height N

    let beaconAfter = RandomBeaconHistory.sourceOfRandomness(atBlockHeight: heightN).value

    // 4. FAILS — beaconBefore != beaconAfter
    Test.assertEqual(beaconBefore, beaconAfter)
}
```

The test fails because re-executing `heartbeat(randomSourceHistory: source)` at the same
nominal height after a reset produces a different internal result — the framework's entropy
source has not been wound back.

---

## Workaround 1 — Single-Chain Continuity

Probe and consume beacon values in the same forward-moving chain. Never reset between the
probe and the assertion that depends on it.

```cadence
// ✅ Probe, derive result, and assert all in one forward chain

access(all) fun testBeaconUsage() {
    let source: [UInt8] = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16]
    recordBeacon(source: source)

    let height = getCurrentBlockHeight()
    let beacon = RandomBeaconHistory.sourceOfRandomness(atBlockHeight: height).value

    // Derive the result using the beacon.
    let outcome = deriveOutcome(beacon: beacon)

    // Assert on the derived outcome — no reset between probe and assertion.
    Test.assert(outcome < 100, message: "expected outcome < 100")
}
```

If the test file uses `beforeEach()` / `Test.reset(to: setupHeight)` for isolation between
cases, structure RBH-dependent cases so they run from `setupHeight` forward without any
mid-test reset. One RBH-dependent test per file is simpler than managing multiple.

---

## Workaround 2 — Shell-Script Integration Tests Against the Standalone Emulator

For tests that genuinely need deterministic beacon values across independent runs, move them
to Layer 2: shell scripts that drive the standalone `flow emulator`.

The standalone emulator's system chunk transaction executes `heartbeat()` once per block,
producing beacon values that are:
- Recorded automatically without manual transaction overhead
- Deterministic per run when the emulator is started from a clean state with the same seed
- NOT subject to the reset/replay divergence because the emulator is restarted, not rewound

```bash
#!/usr/bin/env bash
set -e

flow emulator --log-level error &
EMULATOR_PID=$!
trap "kill $EMULATOR_PID" EXIT

flow project deploy --network emulator

# At block 5, beacon is available at block 6 (1-block lag rule)
flow transactions send cadence/transactions/advance_block.cdc --network emulator
flow transactions send cadence/transactions/advance_block.cdc --network emulator
# ...

RESULT=$(flow scripts execute cadence/scripts/read_beacon.cdc 5 --network emulator)
echo "Beacon at block 5: $RESULT"
# Assert RESULT matches expected value from a known run
```

---

## Interaction with the setup-then-reset Trap

The [setup-then-reset trap](blockchain-emulation.md#the-setup-then-reset-trap) (capturing
`setupHeight` with a stale `getCurrentBlock().height`) compounds the beacon issue. If
`setupHeight` is captured incorrectly and `Test.reset(to: setupHeight)` rewinds past contract
deployments, the next test not only loses its contracts but also loses any beacon entries that
were recorded during setup. Fix the height capture first (use `BlockchainHelpers.getCurrentBlockHeight()`),
then evaluate whether beacon-dependent tests need the additional continuity guarantees above.

---

## Other Entropy-Dependent State

The `RandomBeaconHistory` divergence is the most commonly encountered case, but the same
principle applies to any contract that derives state from per-block entropy:

- `revertibleRandom<T>()` results — different invocations on the re-run chain produce
  different values even if the block height matches
- Any contract that stores a seed derived from `getCurrentBlock().timestamp` combined with
  block-specific nonces

For `revertibleRandom`, this is typically a non-issue because tests using it should not rely
on specific return values (the values are by definition non-deterministic). If a test checks
the distribution properties of randomness, structure it to run without resets.

---

## Common Pitfalls

- **"My test passes in isolation but fails in the suite"** — A reset in a preceding test
  altered the internal entropy state. Structure RBH-dependent tests to run first in the file
  (before any `Test.reset` call) or ensure they are fully self-contained from the post-setup
  height forward.
- **"I reset to exactly the block where I recorded the beacon but the value changed"** —
  This is the canonical failure pattern described above. The internal entropy source has
  advanced past the point of original recording. Use single-chain continuity instead.
- **"Test.reset does not clear my RandomBeaconHistory entries"** — Correct: `Test.reset`
  reverts storage, so entries recorded before the reset target height remain. Entries recorded
  between the target and the original height are removed. This is expected behavior.

---

## See Also

- [`system-contracts-availability.md`](system-contracts-availability.md) — Why
  `RandomBeaconHistory` must be deployed manually in `flow test` and how the heartbeat
  mechanism differs from the standalone emulator
- [`blockchain-emulation.md`](blockchain-emulation.md) — `Test.reset(to:)` usage, the
  setup-then-reset trap, and snapshot isolation patterns
- [`cadence-lang/references/randomness.md`](../../cadence-lang/references/randomness.md) —
  `RandomBeaconHistory`, `RandomConsumer`, and when to use each randomness primitive
