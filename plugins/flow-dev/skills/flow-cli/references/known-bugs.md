# Known Bugs in Flow CLI

This file tracks confirmed upstream/CLI defects that the documentation must guard developers against until they are fixed upstream. Distinct from `## Common Pitfalls` sections (which document developer-error guards), Known Bugs are bugs in the tooling itself.

---

### v2.17.1 `flow schedule setup` does not publish the Manager public capability

<!-- stale: re-verify on next Flow CLI major -->
**Affected versions:** Flow CLI v2.16.0, v2.17.0, **v2.17.1**, and current
`master` at SHA `632cf020`. Full investigation:
`flow-ai-tools-drafts/scheduled-transactions/CLI-BUG-INVESTIGATION.md`.

**Symptom.** `flow schedule setup` reports success, but the next
`flow schedule list` panics:

```
$ flow schedule setup --signer my-account
Setting up Transaction Scheduler Manager...
Transaction Scheduler Manager is setup

$ flow schedule list my-account
panic: Could not borrow Manager from account
```

**Cause.** The inline setup tx in `flow-cli/internal/schedule/setup.go`
(introduced in PR #2130, Oct 15 2025) saves a `Manager` resource to storage
but never publishes the matching public capability at
`FlowTransactionSchedulerUtils.managerPublicPath`. The `list` script reads
from that public path.

✅ **Workaround A — publish the cap with a one-shot transaction (recommended).**
Save the following at `cadence/transactions/publish_manager_pub.cdc` and run
it once per affected account, after `flow schedule setup`:

```cadence
import "FlowTransactionSchedulerUtils"

transaction {
    prepare(acct: auth(Storage, Capabilities) &Account) {
        let existingCap = acct.capabilities.get<&{FlowTransactionSchedulerUtils.Manager}>(
            FlowTransactionSchedulerUtils.managerPublicPath)
        if existingCap.check() { return }

        let mgrCap = acct.capabilities.storage.issue<&{FlowTransactionSchedulerUtils.Manager}>(
            FlowTransactionSchedulerUtils.managerStoragePath)
        acct.capabilities.publish(mgrCap, at: FlowTransactionSchedulerUtils.managerPublicPath)
    }
}
```

```bash
flow schedule setup --signer my-account
flow transactions send cadence/transactions/publish_manager_pub.cdc --signer my-account
flow schedule list my-account   # now works
```

Idempotent: `capabilities.get(...).check()` short-circuits when the cap is
already published.

✅ **Workaround B — build a fixed Flow CLI locally.** The investigation report
includes a single-file patch to `internal/schedule/setup.go` and the exact
`go build` invocation. Use this if you operate many accounts and want
`flow schedule setup` itself to "just work." See `CLI-BUG-INVESTIGATION.md`
§5–6 for the diff and verification.

❌ **Do not** silently ignore the panic and re-run `flow schedule setup` —
re-running the buggy setup does not publish the cap either.

**Upstream status.** As of Flow CLI v2.17.1, no upstream issue or PR fixes
this. Track `github.com/onflow/flow-cli` for a patch release; remove the
workaround once the inline setup tx in `setup.go` is corrected.

### `flow schedule get` argument is `UInt64`, not a hash

The `--help` text shows `flow schedule get 0x1234567890abcdef` as an example,
which looks like a Flow transaction hash. It is **not** — `get` expects the
numeric scheduler id (`UInt64`) returned by `FlowTransactionScheduler.schedule(...)`
and emitted in the `Scheduled` event. Pass the integer, not a hex string.

### `flow schedule list` returns "no transactions" for direct schedules

By design — see "Manager-only visibility" in `scheduled-transactions.md`. Route through
`FlowTransactionSchedulerUtils.Manager` if you want the CLI to see them.

### No `flow blocks tick` or `flow schedule force`

The CLI has no command to manually fire a scheduled transaction or step the
emulator clock. Use `flow emulator -b 1s` (recommended) or send no-op
transactions in a loop to advance block time.
