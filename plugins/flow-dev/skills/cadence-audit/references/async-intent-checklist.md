# Audit Checklist: Async Intent Systems

Async intent systems on Flow combine four distinct protocol layers — the
`FlowTransactionScheduler` primitive, a strategy registry that controls which
execution paths are trusted, a multi-transaction escrow that holds user funds
across the execution window, and CrossVM calls that reach EVM contracts from
within scheduled handlers. Auditing any one layer in isolation misses the
failure modes that only appear at the seams: a handler that correctly narrows
its capabilities may still drain funds if the strategy it dispatches to lacks
usage caps; an escrow with a perfect phase machine may still leave funds stuck
if the CrossVM leg has no time-based reclaim; a registry with proper two-step
approval may still be compromised in a single block if its admin capability is
shared. Because every layer is asynchronous with respect to the user's original
signing transaction, the composite trust model is: user signs intent at T=0,
executor settles at T=N, and any of the four layers can fail silently or
maliciously in between. This checklist synthesises the per-domain audit
references into a unified instrument for systems that touch all four layers
simultaneously.

---

## Severity Rubric

### Critical — funds at risk

Failure leads to direct or near-direct loss of user principal: escrow drainable
without authorization, executor compromise granting arbitrary call authority,
replay allowing double-settlement.

Representative findings:

- ❌ Handler holds `auth(Admin)` capability to the registry: privilege escalation via scheduled execution (see forte-anti-patterns.md §A5)
- ❌ COA auth capability published at `/public/evm`: any account can drain the EVM-side escrow (see crossvm-anti-patterns.md §C5)
- ❌ Shared `auth(EVM.Call)` cap given to multiple consumers with no per-consumer revocation (see crossvm-anti-patterns.md §C4)
- ❌ Registry `addStrategy` takes immediate effect with no audit window and no multi-sig approval (see strategy-registry-anti-patterns.md §S1, §S8)
- ❌ `revertibleRandom` abort-on-bad-roll in settlement path: user selectively reverts unfavorable outcomes (see randomness-vulns.md §V1)

### High — execution-correctness failures

Settlement completes but delivers the wrong amount, the wrong recipient, or
leaves a leak that compounds over time.

Representative findings:

- ❌ `coa.call` result status not checked: Cadence side commits while EVM side reverted, inventing value (see crossvm-anti-patterns.md §C1)
- ❌ EVM return values used without range-checking: attacker-controlled oracle returns out-of-range price (see crossvm-anti-patterns.md §C7)
- ❌ Strategy upgrades silently: approved strategy replaced with drainer; registry has no kind-mismatch detection (see strategy-registry-anti-patterns.md §S7)
- ❌ Handler panic treated as skippable: `Executed` event absent, fees consumed, work lost (see forte-anti-patterns.md §A6, §A7)
- ❌ `data: AnyStruct?` used to control destination or amount inside handler (see forte-anti-patterns.md §A8)

### Medium — operational failures

System halts, funds are temporarily inaccessible, fees are wasted, or
cancellation windows are missed.

Representative findings:

- ❌ Unbounded self-rescheduling with no kill switch or fee-reserve floor (see forte-anti-patterns.md §A1)
- ❌ Registry has no `Paused` or `Deprecated` state; only binary active/absent (see strategy-registry-anti-patterns.md §S4)
- ❌ No time-based reclaim for stuck CrossVM operations (see crossvm-anti-patterns.md §C9)
- ❌ Race between scheduled tick and manual withdrawal against unguarded shared resource (see forte-anti-patterns.md §A9)
- ❌ Cancel-and-reschedule assumes 100% refund; half-funded reschedule panics (see forte-anti-patterns.md §A4)

### Low — observability gaps

System functions correctly but produces insufficient audit trail, making
incidents difficult to detect or reconstruct.

Representative findings:

- ❌ Registry mutations emit no events or emit generic `Updated` events with no admin address (see strategy-registry-anti-patterns.md §S9)
- ❌ EVM failure modes not emitted as Cadence events when protocol chooses to continue rather than panic (see crossvm-anti-patterns.md — Observability checklist)
- ❌ EVM addresses stored as `String` rather than `EVM.EVMAddress`; comparison bugs in authorisation logic (see crossvm-anti-patterns.md §C10)
- ❌ Scheduled tick execution confirmed by `getStatus()` rather than by presence of `Executed` event (see forte-anti-patterns.md §A7)

---

## Phase 1 — Intent Submission

The user signs a transaction that creates the intent struct or resource, locks
deposit terms, and hands escrow authority to the protocol. This is the only
moment at which the user's consent is on-chain. Errors here define the
boundaries the rest of the system must enforce.

**What an auditor reads first:** the intent struct or resource definition —
specifically its lifecycle states, its deadline field, its deposit type and
amount constraints, and whether its slippage terms can be modified after
signing.

### Checklist

1. **Are all intent terms (input type, input amount, output minimum, deadline, recipient, fee ceiling) encoded in on-chain resource state at creation time, not passed as calldata at settlement time?**
   Why it matters: terms supplied at settlement time are executor-controlled. A slippage floor passed as a parameter to the settlement transaction can be set to zero by a malicious or compromised executor.
   See: forte-anti-patterns.md §A8 — `data: AnyStruct?` is the analogous vulnerability when terms are baked into handler `data` instead of handler fields.

2. **Is the intent resource non-copyable and stored at a well-defined path indexed by a Manager resource (not derived from txID)?**
   Why it matters: a storage path derived from an external identifier races — two submissions with the same derived identifier can overwrite or collide, and `save()` panics on collision.
   See: multi-tx-escrow.md §Anti-pattern 1 — storing funds at paths indexed by txID.

3. **Does the intent creation transaction fail atomically if any part of it is invalid (broken capability, past deadline, zero amount)?**
   Why it matters: partial creation leaves orphaned state. A Manager with a reference to an uninitialized escrow can panic on borrow.
   See: multi-tx-escrow.md — `init` pre-conditions must reject past deadlines and empty vaults.

4. **Is the deposit token type validated at intake (not assumed from a type parameter)?**
   Why it matters: a capability typed `&{FungibleToken.Vault}` accepts any vault. Without a `getType()` check, a depositor can fund the intent with the wrong token, which the settlement strategy may then try to swap incorrectly.
   See: strategy-registry.md §Capability type validation at registration time — the same cap.check() principle applies at intent creation.

5. **Is the deadline a fixed field set at creation, not mutable by executor or admin without a separate governance step?**
   Why it matters: a mutable deadline allows an executor to extend the window indefinitely, deferring accountability while funds sit in escrow.
   See: multi-tx-escrow.md — deadline is an `access(all) let` field on the `Escrow` resource.

6. **Is the slippage floor (minimum output) part of the signed terms and enforced by the settlement transaction's post-condition, not by the strategy alone?**
   Why it matters: if only the strategy enforces slippage, a strategy upgrade (silent, see S7) or a malicious registry can bypass it. The settlement transaction must post-check independently.
   See: composition-patterns.md §Atomicity Guarantees — slippage guards belong in the caller's post-condition, not just inside the Swapper.

7. **Does the intent creation emit a fully-qualified event (intent ID, depositor, recipient, token type, amount, deadline, fee ceiling)?**
   Why it matters: the creation event is the indexer's source of truth for the intent's terms. Missing fields prevent off-chain monitoring from detecting deadline extensions, underpayments, or mismatched recipients.
   See: strategy-registry-anti-patterns.md §S9 — event completeness.

8. **Are fee caps (maximum executor fee) encoded in the intent and enforced at settlement, not left to executor discretion?**
   Why it matters: an uncapped executor fee is a theft surface. The fee a keeper extracts should be bounded by what the user agreed to at signing time.

---

## Phase 2 — Strategy Resolution

After the intent is created, the executor must select a strategy. The registry
is the trust boundary: only approved capabilities may execute. An audit of this
phase answers whether the registry's whitelist actually constrains execution or
is only nominal.

**What an auditor reads:** the registry contract and its `run()` or
`execute()` dispatch path; the interface the strategy must conform to; the
lifecycle state machine; the governance path for adding new strategies.

### Checklist

1. **Does `addStrategy` (or equivalent) call `cap.check()` in a pre-condition and reject when it returns false?**
   Why it matters: `Capability` values are runtime entities. The declared parameter type is a static promise; `cap.check()` exercises the actual resolution path.
   See: strategy-registry-anti-patterns.md §S3 — missing `cap.check()` at registration.

2. **Does registration require a two-step propose → delay → finalize flow (or equivalent audit window)?**
   Why it matters: a single-call approval means a compromised admin installs a draining strategy and routes funds in the same block.
   See: strategy-registry-anti-patterns.md §S1 — single-call approval; §S8 — admin key compromise.

3. **Does the registry store `Capability<...>` values rather than borrowed references?**
   Why it matters: references are transient. A stored `&{StrategyInterface}` becomes stale after a strategy upgrade.
   See: strategy-registry-anti-patterns.md §S5 — storing borrowed references instead of capabilities.

4. **Is the strategy's self-reported `kind` or version identifier recorded at registration and re-checked on every execute?**
   Why it matters: a strategy can be upgraded after approval. Without a kind-mismatch check, the upgraded code runs under the original approval.
   See: strategy-registry-anti-patterns.md §S7 — silent strategy upgrades.

5. **Does the registry enforce at least three lifecycle states (Active, Paused, Deprecated/Revoked) via an enum?**
   Why it matters: a binary active/absent state forces "leave a broken strategy live" vs "wipe the entry" — neither is acceptable during incident response.
   See: strategy-registry-anti-patterns.md §S4 — binary lifecycle.

6. **Is there a `revokeStrategy` callable by the same entitlement that added the strategy?**
   Why it matters: without revocation, a discovered vulnerability cannot be mitigated without a full contract upgrade.
   See: strategy-registry-anti-patterns.md §S2 — missing revocation path.

7. **Are per-tx, per-block, and per-user usage caps enforced by the registry on every execute?**
   Why it matters: a buggy strategy that leaks 0.1% per call becomes a 100% drain over enough calls; a malicious one can drain TVL in a single block without caps.
   See: strategy-registry-anti-patterns.md §S6 — no usage caps or rate limits.

8. **Is the admin capability (`auth(Admin) &Registry`) stored privately and never published at a public path?**
   Why it matters: a published admin capability is bearer authority over every strategy in the whitelist.
   See: strategy-registry.md — "Publishing the entitled registry cap. `auth(Admin) &Registry` must never be published."

9. **Does the registry's execute path borrow the capability at call time (not store a pre-borrowed reference) and route an observable failure when `borrow()` returns nil?**
   Why it matters: a nil borrow inside a settlement that continues silently can mark the intent as settled while the strategy never ran.
   See: strategy-registry-anti-patterns.md §S5; also crossvm-anti-patterns.md §C1.

---

## Phase 3 — Escrow Custody

Funds are held from intent creation through settlement or refund. The escrow
resource's phase machine determines who may move funds in each state and what
happens when the deadline passes.

**What an auditor reads:** the escrow resource definition, its phase enum, its
entry-point pre-conditions, and the capabilities issued to each actor (depositor,
recipient/executor, admin).

### Checklist

1. **Does every state-mutating entry point on the escrow check the phase via a `pre` condition before executing?**
   Why it matters: without phase guards, a manual transaction can claim funds during the `Funding` phase or refund during the `Claimable` phase.
   See: forte-anti-patterns.md §A2 — shared state without a state machine.

2. **Does the escrow use an explicit phase enum with at least five states (Funding, Claimable, Settled, Expired, Closed)?**
   Why it matters: a boolean `isClaimable` flag cannot express "expired but not yet refunded" — a state that requires a distinct refund path.
   See: multi-tx-escrow.md §The phases.

3. **Is `claim()` guarded by a deadline check so that a late executor cannot claim after expiry?**
   Why it matters: without a deadline check, an executor can claim after the depositor has reclaimed, if the order of transactions in a block goes wrong.
   See: multi-tx-escrow.md — `claim()` pre-condition checks `getCurrentBlock().timestamp <= self.deadline`.

4. **Is there a publicly callable `expireIfStale()` so any party can drive the `Claimable -> Expired` transition without privileged access?**
   Why it matters: if only the depositor can trigger expiry, a griefing executor who holds the claim capability can block the refund path indefinitely.
   See: multi-tx-escrow.md — `expireIfStale` is `access(all)`.

5. **Are the depositor's `Refund` capability and the executor's `Claim` capability issued separately and with the narrowest possible entitlement set?**
   Why it matters: a combined `auth(Claim, Refund)` capability handed to the executor is a full escrow takeover; the executor can both claim and refund at will.
   See: multi-tx-escrow.md §Ownership invariants — one entitlement per actor role.

6. **For CrossVM escrow: is the COA capability stored in the escrow as a `Capability<auth(EVM.Call, EVM.Withdraw) &EVM.CadenceOwnedAccount>` (not the COA resource itself)?**
   Why it matters: moving the COA resource out of the depositor's account breaks wallet tools, indexers, and every capability previously issued against `/storage/evm`.
   See: coa-lifecycle.md §Anti-pattern 3 — storing the COA resource inside another resource.

7. **Is the controller ID of every COA capability issued to the escrow tracked so it can be deleted on close?**
   Why it matters: an unclosed capability controller keeps bearer authority alive after the escrow is settled.
   See: coa-lifecycle.md §Issuing and revoking the COA capability.

8. **Does the escrow resource hold funds inside itself (not at a separate storage path referenced by the escrow)?**
   Why it matters: funds held at a separate path are not protected by the escrow's phase machine and can be moved by any transaction that knows the path.
   See: multi-tx-escrow.md — `access(self) var vault: @{FungibleToken.Vault}` inside the resource.

9. **Is the admin recovery path (`adminRecover`) gated to the `Expired` phase only, with a mandatory reason string emitted in the event?**
   Why it matters: an admin recovery available in `Claimable` phase can front-run a legitimate executor claim. A reason-less event destroys the forensic trail.
   See: multi-tx-escrow.md — `adminRecover` pre-condition requires `Phase.Expired` and a non-empty reason.

---

## Phase 4 — Scheduled Execution

The handler fires when the scheduler reaches the intent's execution timestamp.
It borrows the strategy, dispatches the DeFiActions composition, handles
CrossVM calls if applicable, and advances the escrow phase.

**What an auditor reads:** the `executeTransaction` body in full, every
capability it borrows, the CU budget assigned at schedule time, and the
conditions under which it reschedules itself.

### Checklist

1. **Does the handler hold only the narrowest capabilities needed for one tick — never `auth(Admin)`, never multi-purpose, never an `&Account` reference?**
   Why it matters: the handler is callable by anyone holding an `auth(Execute)` cap; wider entitlements escape to every capability-holder.
   See: forte-anti-patterns.md §A5 — privilege escalation via handler capability.

2. **Does `executeTransaction` use `return` (not `panic`) for all recoverable conditions such as empty source, full sink, or disabled flag?**
   Why it matters: a panic wastes the full tick fee, emits no `Executed` event, and terminates any self-rescheduling chain.
   See: forte-anti-patterns.md §A6 — handler panic as silent data loss; scheduled-integration.md §Failure Handling.

3. **Is `data: AnyStruct?` treated as untrusted and informational only — never used to select capabilities, destinations, or amounts?**
   Why it matters: `data` is opaque and attacker-controllable. Business-critical parameters must be baked into handler fields at construction, not passed per-tick.
   See: forte-anti-patterns.md §A8 — leaking COA/capability via publicly callable handler.

4. **If the handler dispatches a DeFiActions composition, does it generate a fresh `UniqueIdentifier` each tick (not reuse one stored at handler init)?**
   Why it matters: reusing an ID across ticks merges all `Withdrawn`, `Swapped`, and `Deposited` events into a single indistinguishable trace.
   See: scheduled-integration.md §UniqueIdentifier per Tick vs. per Handler Lifetime.

5. **Does the handler assert `vault.balance == 0.0` before `destroy vault` at the end of every composition pipeline?**
   Why it matters: a non-zero residual means the Sink absorbed less than expected — tokens were silently lost inside the tick.
   See: composition-patterns.md §Six Critical Safety Rules rule 5.

6. **For every `coa.call` inside the handler: is `result.status == EVM.Status.successful` checked, with a `panic` or observable failure path on non-success?**
   Why it matters: a failed EVM call does not automatically revert the Cadence side; Cadence commits whatever happened before and after it.
   See: crossvm-anti-patterns.md §C1 — ignoring `result.status`.

7. **Is the CU budget (`executionEffort`) sized from a measured sweep, not assumed, and is it below the 9,999 CU per-tx ceiling?**
   Why it matters: a handler that exceeds its budget mid-execution rolls back all work and consumes all fees with no retry.
   See: forte-anti-patterns.md (CU ceiling section); scheduled-integration.md §Per-Tick CU Budget; scheduled-transactions.md §Per-tick / per-slot CU ceilings.

8. **If the handler self-reschedules, does it (a) check a kill-switch before rescheduling, (b) enforce a hard iteration cap or absolute deadline, and (c) check the fee reserve balance before withdrawing for the next tick?**
   Why it matters: unbounded self-rescheduling drains the reserve vault until a panic inside the handler terminates the chain mid-execution.
   See: forte-anti-patterns.md §A1 — unbounded self-rescheduling.

9. **Does the handler call `FlowTransactionScheduler.estimate()` before each reschedule and handle a non-nil error by returning cleanly (not panicking)?**
   Why it matters: a full priority slot causes `schedule()` to panic, which rolls back the composition side effects that already ran in the same tick.
   See: scheduled-integration.md §Common Pitfalls — "Not calling estimate() before rescheduling."

10. **Is the handler's `auth(FlowTransactionScheduler.Execute)` capability stored privately and never published at a public path?**
    Why it matters: a published Execute capability lets any account trigger the handler outside the scheduler's slot model, bypassing timing and phase checks.
    See: forte-anti-patterns.md §A5; forte-anti-patterns.md §Audit checklist — Capability hygiene.

---

## Phase 5 — Settlement

The handler has executed. Settlement delivers output tokens to the recipient,
deducts fees, and transitions the escrow to `Settled`. This is the phase where
slippage, recipient identity, and fee correctness are observable on-chain.

**What an auditor reads:** the settlement transaction or handler tail, the
post-conditions asserted before `close()`, and the events emitted.

### Checklist

1. **Is the slippage floor (minimum output) checked by a `post` condition or `assert` in the settlement transaction, independent of the strategy's own checks?**
   Why it matters: defense in depth — a strategy upgrade or compromised strategy can bypass its own slippage guard; the settlement transaction's assertion cannot be upgraded by the strategy provider.
   See: composition-patterns.md §Pattern 2 — slippage guard in the `post` block.

2. **Is output delivered to the recipient address encoded in the intent, not to an address derived from handler `data` or strategy return values?**
   Why it matters: an executor-controlled output address is a theft path that bypasses the intent's signed terms.
   See: forte-anti-patterns.md §A8 — `data` used to select destination.

3. **When `coa.call` is used to deliver EVM-side output, is `result.status` checked and are the decoded return values range-validated before advancing the escrow phase?**
   Why it matters: a settled escrow whose EVM payment silently reverted has delivered nothing to the recipient while the Cadence phase says `Settled`.
   See: crossvm-anti-patterns.md §C1, §C7.

4. **Are fees deducted from the protocol's designated fee vault, not from the escrow principal, before settlement is confirmed?**
   Why it matters: a fee deduction that draws from the escrow principal can leave the recipient underpaid relative to the signed terms.

5. **Are all settlement events emitted with flat scalar fields (intent ID, executor address, input amount, output amount, fee charged, strategy ID, timestamp)?**
   Why it matters: settlement events are the indexer's record of execution quality. Missing fields — especially executor address and fee charged — prevent post-hoc audits.
   See: strategy-registry-anti-patterns.md §S9 — event completeness; strategy-registry.md `StrategyExecuted` event pattern.

6. **Does `close()` require `vault.balance == 0.0` before transitioning the escrow to `Closed`?**
   Why it matters: a non-zero balance at close means the resource persists with stranded funds — there is no path out.
   See: multi-tx-escrow.md — `close()` pre-condition.

7. **Is the `Settled` → `Closed` transition triggered in the same transaction as the final `claim()` (or at least by the same actor in the same block)?**
   Why it matters: leaving the escrow in `Settled` with no `close()` occupies storage indefinitely and prevents a clean audit of live vs. settled intents.
   See: multi-tx-escrow.md §Common pitfalls — "Forgetting `close()` leaves a `Settled` resource forever."

8. **Is the escrow COA capability controller deleted after `close()`, so EVM-side authority does not persist past the lifecycle?**
   Why it matters: an unclosed capability controller is bearer authority over the depositor's EVM address for an indefinite period.
   See: coa-lifecycle.md §Issuing and revoking the COA capability.

---

## Phase 6 — Refund Path

If execution fails, expires, or is cancelled by the depositor before arming,
the depositor must recover their principal. Auditing the refund path is not
optional — it is as important as auditing the happy path, because stuck escrows
are a common failure mode in async systems.

**What an auditor reads:** the `refund()`, `cancel()`, and admin recovery entry
points; the deadline check and who can trigger the `Claimable -> Expired`
transition; and the CrossVM reclaim path if EVM-side funds are involved.

### Checklist

1. **Is there a documented and tested deadline-based refund path (not just "admin can recover")?**
   Why it matters: a purely admin-controlled refund requires trusting the protocol operator; an on-chain deadline refund is trustless.
   See: multi-tx-escrow.md §Recovery paths 1 — timeout-based refund.

2. **Can the depositor call `refund()` unilaterally after the deadline without needing executor cooperation?**
   Why it matters: if the executor must call a function to unlock the refund path, they can block recovery indefinitely by doing nothing.
   See: multi-tx-escrow.md — `expireIfStale()` is `access(all)`; `refund()` requires only the depositor's `Refund` cap.

3. **For CrossVM escrow: does the reclaim path use `coa.withdraw(balance:)` with the `EVM.Withdraw` entitlement, not an EVM-side `transfer` call?**
   Why it matters: an EVM-side transfer depends on off-chain relayers and bridge handlers; `coa.withdraw` is atomic with the surrounding Cadence transaction and does not require a live relayer.
   See: coa-lifecycle.md §Withdrawing FLOW from a COA back to Cadence.

4. **Is there a time-based reclaim or admin sweep for EVM funds stuck in a COA after a CrossVM call failure?**
   Why it matters: if `coa.call` into a strategy reverts permanently (strategy paused, upgraded, or malicious), the FLOW in the COA has no on-chain escape path without an explicit reclaim.
   See: crossvm-anti-patterns.md §C9 — no timeout or fallback on stuck CrossVM operations.

5. **Does the cancel path (pre-arm) return 100% of the depositor's principal without a fee deduction?**
   Why it matters: a cancel that deducts a fee before any execution has occurred is not a refund — it is a tax on changing one's mind.
   See: multi-tx-escrow.md — `cancel()` is available in `Phase.Funding` only; returns full balance.

6. **When a scheduled refund transaction cancels a future tick, does the code account for the 50% cancel-fee refund rather than assuming 100%?**
   Why it matters: cancelling a scheduled transaction returns only 50% of the paid fee; using the refund to fund the next reschedule will panic because the vault is half-funded.
   See: forte-anti-patterns.md §A4 — assuming 100% fee refund on cancel.

7. **Is the admin recovery path (`adminRecover`) restricted to the `Expired` phase, logged with a reason, and not callable while the executor's claim window is still open?**
   Why it matters: admin recovery in `Claimable` phase can front-run a legitimate executor claim; an unrestricted admin recovery is a Critical finding.
   See: multi-tx-escrow.md — `adminRecover` requires `Phase.Expired`.

8. **Are all refund and admin-recovery events emitted with the depositor address, recovered amount, and a timestamp?**
   Why it matters: refund events are the only on-chain record that depositors were made whole after a failed intent. Missing fields prevent reconciliation.
   See: strategy-registry-anti-patterns.md §S9 — event completeness.

---

## Composite Audit Flow

An auditor approaching an async intent system for the first time should read
in the following order. Each step narrows the trust boundary before the
next step expands it.

1. **Read the intent struct or resource definition.** Identify: the lifecycle phase enum and every legal state transition; the deadline, deposit type, deposit amount, output minimum, recipient, and fee ceiling fields; whether any of these are mutable after creation; and whether the `init` pre-conditions reject invalid states at creation time.

2. **Read the strategy registry contract.** Map the trust boundary: what is `addStrategy`'s flow (single-call vs. two-step propose/delay/finalize); does it call `cap.check()`; what lifecycle states exist; is there a `revokeStrategy`; are usage caps enforced per-strategy; is the `kind` identifier recorded and re-checked on every execute. Apply the full strategy-registry-anti-patterns.md checklist (S1–S9) here.

3. **Read the escrow resource.** Verify: every entry point has a phase pre-condition; the phase enum has at least five states; `expireIfStale()` is callable by anyone; `claim()` checks the deadline; `adminRecover` requires `Expired` phase; `close()` requires zero balance; capabilities are issued with one-entitlement-per-actor-role. Apply the multi-tx-escrow.md anti-patterns check here.

4. **Read the scheduled-handler implementation.** Check: handler capabilities are the narrowest possible set; `executeTransaction` uses `return` for skippable conditions; `data` is informational only; self-rescheduling has a kill switch, iteration cap, and reserve floor; `estimate()` is called before rescheduling; the Execute capability is not published. Apply the full forte-anti-patterns.md checklist (A1–A10) here.

5. **Read the settlement transaction and every `coa.call` site.** Verify: `result.status` is checked for every EVM call; decoded return values are range-validated; slippage floor is asserted post-composition; output recipient matches the signed intent; fee deduction is bounded; settlement events are complete. Apply the crossvm-anti-patterns.md checklist (C1–C10) here.

6. **Read the refund and admin-recovery paths.** Verify: deadline-based refund is trustless; depositor can call `refund()` unilaterally after expiry; CrossVM reclaim uses `coa.withdraw`; EVM stuck-funds have a time-based escape; cancel-fee accounting is correct (50%, not 100%); admin recovery is phase-gated and audit-logged.

7. **Cross-check with all four per-domain anti-pattern references.** Walk the full anti-pattern lists — forte-anti-patterns.md (A1–A10), strategy-registry-anti-patterns.md (S1–S9), crossvm-anti-patterns.md (C1–C10), randomness-vulns.md (V1–V6) — and confirm every item that could apply to the system under review has been addressed or explicitly scoped out. Document any deliberate exceptions.

---

## See Also

- [forte-anti-patterns.md](forte-anti-patterns.md) — 10 scheduled-transaction anti-patterns: unbounded self-rescheduling (A1), missing state machine (A2), tick-serialisation assumption (A3), cancel-refund accounting (A4), privilege escalation (A5), handler panic (A6, A7), COA capability via `data` (A8), manual-vs-scheduled race (A9), indexer collection-walk miss (A10)
- [strategy-registry-anti-patterns.md](strategy-registry-anti-patterns.md) — 9 registry anti-patterns: single-call approval (S1), missing revocation (S2), missing `cap.check()` (S3), binary lifecycle (S4), stored reference vs capability (S5), uncapped execution (S6), silent upgrades (S7), single-key admin (S8), absent events (S9)
- [crossvm-anti-patterns.md](crossvm-anti-patterns.md) — 10 CrossVM anti-patterns: ignored `result.status` (C1), unbounded loop (C2), false atomicity assumption (C3), shared COA cap (C4), auth cap at `/public/evm` (C5), UFix64 attoflow conversion (C6), unvalidated EVM return values (C7), re-entrancy via EVM callback (C8), no CrossVM timeout (C9), EVM address as String (C10)
- [randomness-vulns.md](randomness-vulns.md) — 6 randomness vulnerabilities: abort-on-bad-roll (V1), modulo bias (V2), same-block reveal (V3), public Consumer capability (V4), single-request misuse (V5), script-based randomness in tests (V6)
- [scheduled-transactions.md](../../cadence-lang/references/scheduled-transactions.md) — canonical `FlowTransactionScheduler` API: slot model, fee formula, failure state machine, optimistic-Executed flip
- [multi-tx-escrow.md](../../cadence-lang/references/multi-tx-escrow.md) — canonical multi-transaction escrow with phase enum, one-entitlement-per-actor-role, and admin recovery
- [strategy-registry.md](../../cadence-lang/references/strategy-registry.md) — canonical strategy registry pattern: capability validation, lifecycle enum, multi-sig admin, kind identifier
- [coa-lifecycle.md](../../flow-crossvm/references/coa-lifecycle.md) — COA entitlements, canonical storage paths, custody pattern for cross-VM escrow, capability revocation
- [composition-patterns.md](../../flow-actions/references/composition-patterns.md) — DeFiActions Source → Swapper → Sink patterns, slippage guards, residual-vault assertion, atomicity guarantees
- [scheduled-integration.md](../../flow-actions/references/scheduled-integration.md) — DCAExecutor reference: UniqueIdentifier-per-tick, kill switch, fee reserve, CU budget sizing, panic-vs-return discipline
