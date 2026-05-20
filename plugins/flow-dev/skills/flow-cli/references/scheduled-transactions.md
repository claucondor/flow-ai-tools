# Flow CLI — Scheduled Transactions

The Flow CLI's `flow schedule` command group is the operator-facing layer over the
`FlowTransactionScheduler` / `FlowTransactionSchedulerUtils` contracts that shipped
with the Forte upgrade (Oct 2025). Reach for it when you need to install the
`Manager` resource on an account, list/inspect/cancel transactions the Manager
owns, and — paired with `flow emulator -b 1s` — iterate on handlers locally.
For the Cadence-side API, types, fee math, and failure semantics, see the
companion reference [scheduled-transactions.md](../../cadence-lang/references/scheduled-transactions.md).

## Prerequisites: flow.json deps and account setup

Add the scheduler contracts (and their transitive deps) to your `flow.json`
`contracts` block. The `FlowTransactionSchedulerUtils.Manager` helper is
optional but required for `flow schedule list / get / cancel` to see your
transactions.

```json
{
  "contracts": {
    "FlowTransactionScheduler":      { "aliases": { "emulator": "f8d6e0586b0a20c7", "testnet": "8c5303eaa26202d6" } },
    "FlowTransactionSchedulerUtils": { "aliases": { "emulator": "f8d6e0586b0a20c7", "testnet": "8c5303eaa26202d6" } },
    "FlowToken":     { "aliases": { "emulator": "0ae53cb6e3f42a79", "testnet": "7e60df042a9c0868", "mainnet": "1654653399040a61" } },
    "FungibleToken": { "aliases": { "emulator": "ee82856bf20e2aa6", "testnet": "9a0766d93b6608b7", "mainnet": "f233dcee88fe0abe" } },
    "FlowFees":      { "aliases": { "emulator": "e5a8b7f23e8b548f", "testnet": "912d5440f7e3769e", "mainnet": "f919ee77447b7497" } }
  }
}
```

Mainnet scheduler addresses: both deployed on the service account — resolve via
`flow dependencies install` or the official registry.

### Minimum handler-account setup

The handler-owning account must:

1. Save a `TransactionHandler`-conforming resource to storage.
2. Issue **two** capability controllers off that storage path: an entitled
   `auth(FlowTransactionScheduler.Execute) &{...TransactionHandler}` for
   `schedule()`, and an un-entitled `&{...TransactionHandler}` published at
   a public path for indexers / view resolution.
3. Optionally save a `FlowTransactionSchedulerUtils.Manager` and publish
   `&{Manager}` at `FlowTransactionSchedulerUtils.managerPublicPath` so the
   CLI can read transactions back.

Worked example — handler install transaction, signed by the handler owner:

```cadence
import "FlowTransactionScheduler"
import "Counter"

transaction {
    prepare(acct: auth(Storage, Capabilities) &Account) {
        if acct.storage.borrow<&Counter.Handler>(from: /storage/counterHandler) != nil { return }
        acct.storage.save(<- Counter.createHandler(), to: /storage/counterHandler)

        // Entitled cap for the scheduler.
        let _ = acct.capabilities.storage
            .issue<auth(FlowTransactionScheduler.Execute) &{FlowTransactionScheduler.TransactionHandler}>(/storage/counterHandler)

        // Public un-entitled cap for indexers / view resolution.
        let pubCap = acct.capabilities.storage
            .issue<&{FlowTransactionScheduler.TransactionHandler}>(/storage/counterHandler)
        acct.capabilities.publish(pubCap, at: /public/counterHandler)
    }
}
```

```bash
flow transactions send cadence/transactions/install_counter_handler.cdc --signer my-account
flow schedule setup --signer my-account   # see bug callout below
```

## `flow schedule` command reference

### `flow schedule setup`

Creates a `FlowTransactionSchedulerUtils.Manager` in the signer's storage at
`FlowTransactionSchedulerUtils.managerStoragePath`. The Manager is the CLI's
indexed handle on every transaction *it* schedules — direct
`FlowTransactionScheduler.schedule(...)` calls bypass it.

| Flag | Default | Purpose |
|---|---|---|
| `--signer <name>` | `emulator-account` | Account from `flow.json` to install the Manager on |
| `--network <name>` | `emulator` | Target network |

```bash
flow schedule setup --signer my-account --network testnet
```

Expected output (v2.17.1 — note the missing publish step, see Known Issues):

```
Setting up Transaction Scheduler Manager...
Transaction ID: f90e9f314b74f8c7ee4d900d9dbc24c795792a2000393779fcd0bdab816d0949
Transaction Scheduler Manager is setup
Note: If the manager already existed, no changes were made
```

**Common errors**

- *"Could not borrow Manager from account"* on the next `flow schedule list` —
  the v2.17.1 setup bug. Workaround below.
- Contract-not-deployed errors — confirm both scheduler contracts are aliased
  in `flow.json` for the target network.

### `flow schedule list <account>`

Lists every scheduled transaction registered under the given account's Manager.
Requires the Manager's **public** capability at `managerPublicPath` — the
underlying script calls `FlowTransactionSchedulerUtils.borrowManager(at:)`,
which reads that exact path.

| Argument | Purpose |
|---|---|
| `<account>` | Address (`0x…` or raw hex) or `flow.json` account name |

```bash
flow schedule list my-account
flow schedule list 0xf8d6e0586b0a20c7 --network testnet
```

Expected output on a fresh account (with the bug worked around):

```
Listing scheduled transactions...
Account: my-account (f8d6e0586b0a20c7)
ℹ  No scheduled transactions found
```

**Common errors**

- `panic: Could not borrow Manager from account` — public Manager capability
  missing. Either setup was never run, or it ran on v2.17.1 and hit the bug.
- Empty result for direct `FlowTransactionScheduler.schedule(...)` calls —
  by design. See "Manager-only visibility" below.

### `flow schedule get <transaction-id>`

Returns scheduler-side details for one transaction: status, priority,
timestamp, execution effort, fees, handler owner. Reads
`FlowTransactionScheduler.getTransactionData(id:)` directly — no Manager needed.

| Argument | Type | Purpose |
|---|---|---|
| `<transaction-id>` | `UInt64` | Numeric scheduler id returned by `schedule()` — **not** a Flow transaction hash |

```bash
flow schedule get 42
flow schedule get 17 --network testnet
```

The CLI's `--help` text shows `flow schedule get 0x1234567890abcdef` as an
example — **misleading**. Pass the integer id you got from a `Scheduled` event
or `flow schedule list`. A `0x…` Flow tx hash will fail to parse or return
"transaction not found."

**Common errors**

- *"Transaction not found"* — wrong id, or the tx finalized and was pruned
  (`Executed` entries are removed entirely from `self.transactions`).
- Numeric parse error if you pass a hex string with the `0x` prefix.

### `flow schedule cancel <transaction-id>`

Cancels a `Scheduled`-status transaction and refunds 50% (default
`refundMultiplier`) of fees to the signer.

| Flag / Arg | Default | Purpose |
|---|---|---|
| `<transaction-id>` | — | Numeric scheduler id |
| `--signer <name>` | `emulator-account` | Signer whose Manager owns the tx |
| `--network <name>` | `emulator` | Target network |

```bash
flow schedule cancel 42 --signer my-account --network testnet
```

`cancel` borrows the Manager from **signer storage** with `auth(Owner) &{Manager}`,
so unlike `list` it works even on accounts hit by the v2.17.1 public-cap bug.

**Common errors**

- `panic: Invalid ID: <id> transaction not found` — already finalized
  (`Executed`/`Canceled`) or never existed.
- *"Could not borrow Manager…"* — run `flow schedule setup` on the signer first.

## Running scheduled transactions on the emulator

The emulator processes scheduled transactions automatically once block time
advances past their target timestamp. Two flags govern the experience:

| Flag | Default | What it does |
|---|---|---|
| `--scheduled-transactions` | **`true`** | Enables the scheduler subsystem. Default-on; setting it explicitly is a no-op. |
| `-b, --block-time <duration>` | `0` (manual) | How often the emulator seals a new block. Required for callbacks to fire. |
| `--transaction-fees` | `false` | Enables FLOW fees on transactions. Recommended on — the scheduler charges real fees. |
| `--contracts` | `false` | Deploys the standard contracts at emulator start. Useful but **not** required for scheduler (its contracts are bundled in service-account state). |

### The block-time gotcha — there is no `flow blocks tick`

There is **no** `flow blocks tick` or `flow schedule force` subcommand. With
the default `--block-time 0`, the emulator only seals a block when it receives
a transaction — your handler will never fire on a wall clock alone.

The canonical "make callbacks fire" recipe (used by the T01 verifier):

```bash
flow emulator --scheduled-transactions --transaction-fees --contracts -b 1s
```

`-b 1s` seals a block every wall-clock second. A scheduled timestamp
`getCurrentBlock().timestamp + 5.0` then fires ~5 seconds later, and the
handler's `log()`s and events show up in the emulator log.

### Manually advancing the clock

If you cannot use `-b 1s` (e.g. deterministic replays):

✅ **Send a no-op tx every wall-clock second.** Each accepted tx seals a block
and advances `getCurrentBlock().timestamp`.

```bash
while true; do
  flow transactions send cadence/transactions/noop.cdc --signer emulator-account
  sleep 1
done
```

where `noop.cdc` is:

```cadence
transaction { prepare(s: &Account) {} }
```

❌ **Do not** rely on `flow blocks` to advance time — the subcommand has no
"tick" verb in v2.17.x.

❌ **Do not** assume `flow scripts execute` seals blocks. Scripts do not. Only
transactions do.

### Confirming a tick fired

```bash
grep -i "Executed\|PendingExecution" emulator.log           # scheduler events
flow scripts execute cadence/scripts/get_status.cdc <id>    # on-chain status
```

`Status`: `Scheduled (0) | PendingExecution (1) | Executed (2) | Canceled (3)`.

## Manager-only visibility — and how to work around it

`flow schedule list` and `flow schedule cancel` operate against the
`FlowTransactionSchedulerUtils.Manager`. A handler that calls
`FlowTransactionScheduler.schedule(...)` **directly** (returning a
`@ScheduledTransaction` resource it owns) is invisible to `list`.

To make a direct schedule visible to the CLI, route through the Manager:

```cadence
let mgr = acct.storage.borrow<auth(FlowTransactionSchedulerUtils.Owner)
                              &{FlowTransactionSchedulerUtils.Manager}>(
    from: FlowTransactionSchedulerUtils.managerStoragePath
) ?? panic("no manager")

let id = mgr.schedule(handlerCap: cap, data: nil,
                      timestamp: ts, priority: pri,
                      executionEffort: effort, fees: <- fees)
```

`flow schedule get <id>` works for any tx (Manager-routed or not) because it
reads `FlowTransactionScheduler.getTransactionData(id:)` directly.

## Known issues

See [known-bugs.md](known-bugs.md) for confirmed Flow CLI defects affecting scheduled-transaction workflows.

---

**Trigger phrases to add to `flow-cli/SKILL.md` frontmatter** (for the
integrator, not edited here): `flow schedule`, `flow schedule setup`,
`flow schedule list`, `flow schedule get`, `flow schedule cancel`,
`scheduled transaction CLI`, `Could not borrow Manager from account`,
`Transaction Scheduler Manager`, `manual tick emulator`, `flow emulator -b 1s`,
`how to fire scheduled transaction on emulator`, `publish manager public
capability`, `flow-cli schedule setup bug`.
