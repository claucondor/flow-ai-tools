# Strategy Registry Anti-Patterns

A strategy registry is a contract that maintains a whitelist of approved `Capability` references to external strategy contracts (yield strategies, vault adapters, routing modules, DeFi actions, etc.) and dispatches user funds or protocol calls through them. The whitelist is the security boundary: every approved capability becomes part of the protocol's trusted compute base, and the admin who curates the list becomes a trust anchor. Ordinary Cadence audits — entitlements, resource handling, capability hygiene — apply, but registry-specific failures (silent strategy upgrades, missing revocation paths, capability type confusion, governance compromise) sit one layer above and are easy to miss. This reference enumerates the nine anti-patterns auditors must check, with severity ratings, detection hints, and Cadence 1.0 fixes. Cross-reference [strategy-registry.md](../../cadence-lang/references/strategy-registry.md) for the canonical pattern.

---

## S1 — Admin approval without code review (Critical)

The registry exposes a one-line `addStrategy(cap:)` call gated only by an admin entitlement. Nothing on-chain forces the admin to audit the strategy contract's `execute()` body, its imports, its upgrade history, or its capability surface. Approval is a rubber stamp.

### Bad

```cadence
import "StrategyInterface"

access(all) contract StrategyRegistry {
    access(all) entitlement Admin

    access(self) let strategies: {String: Capability<&{StrategyInterface}>}

    // BAD — single-call approval, no audit window, no rationale captured
    access(Admin) fun addStrategy(
        name: String,
        cap: Capability<&{StrategyInterface}>,
    ) {
        self.strategies[name] = cap
    }
}
```

### Why it's bad

A compromised holder of `Admin` installs a draining strategy and routes user funds through it in a single block. No off-chain window exists for the community, an indexer, or a second admin to observe the pending change and react. Even a well-intentioned admin has no on-chain record of "I reviewed commit X before approving" — the action is opaque after the fact.

### Correct

```cadence
import "StrategyInterface"

access(all) contract StrategyRegistry {
    access(all) entitlement Propose
    access(all) entitlement Finalize
    access(all) let auditWindow: UFix64   // e.g. 86_400.0 (24 h)

    access(all) struct Proposal {
        access(all) let cap: Capability<&{StrategyInterface}>
        access(all) let proposedAt: UFix64
        access(all) let rationale: String   // commit hash / audit report URI
        init(cap: Capability<&{StrategyInterface}>, rationale: String) {
            self.cap = cap
            self.proposedAt = getCurrentBlock().timestamp
            self.rationale = rationale
        }
    }
    access(self) let pending: {String: Proposal}
    access(self) let strategies: {String: Capability<&{StrategyInterface}>}

    access(all) event StrategyProposed(name: String, rationale: String)
    access(all) event StrategyFinalized(name: String)

    access(Propose) fun propose(name: String, cap: Capability<&{StrategyInterface}>, rationale: String) {
        pre {
            cap.check(): "capability does not resolve"
            self.pending[name] == nil && self.strategies[name] == nil: "duplicate"
        }
        self.pending[name] = Proposal(cap: cap, rationale: rationale)
        emit StrategyProposed(name: name, rationale: rationale)
    }
    access(Finalize) fun finalize(name: String) {
        let p = self.pending.remove(key: name) ?? panic("no such proposal")
        assert(getCurrentBlock().timestamp >= p.proposedAt + self.auditWindow, message: "audit window not elapsed")
        self.strategies[name] = p.cap
        emit StrategyFinalized(name: name)
    }
}
```

Two distinct entitlements (`Propose`, `Finalize`) split the workflow; the mandatory delay forces an off-chain audit window observable through `StrategyProposed`.

### Detection hint

Grep for `addStrategy`, `register`, `approve`, `whitelist` in registry-style contracts. If any of these write into the strategies map in a single call with no `proposedAt + window <= now` check (or equivalent delay), flag it. The presence of `pending` + `finalize` is the marker for a healthy two-step flow.

---

## S2 — Missing revocation path (High)

The registry has `addStrategy()` but no `revokeStrategy()`. Once a strategy is approved, it is in the whitelist permanently. A bug discovered after approval can only be mitigated by upgrading the registry contract, which itself may be governed and slow.

### Bad

```cadence
access(all) contract StrategyRegistry {
    access(self) let strategies: {String: Capability<&{StrategyInterface}>}

    access(Admin) fun addStrategy(name: String, cap: Capability<&{StrategyInterface}>) {
        self.strategies[name] = cap
    }
    // BAD — no revokeStrategy, no setLifecycle, no way to remove
}
```

### Why it's bad

Once a strategy is shown to be vulnerable, the protocol has no fast lever. Hiding it in the frontend doesn't help — scripts and other contracts still call into the registry and execute it. Every block between "we know it's broken" and "upgraded registry deployed" is full exposure time.

### Correct

```cadence
access(all) contract StrategyRegistry {
    access(all) entitlement Admin
    access(self) let strategies: {String: Capability<&{StrategyInterface}>}

    access(all) event StrategyAdded(name: String, admin: Address)
    access(all) event StrategyRevoked(name: String, admin: Address, reason: String)

    access(Admin) fun addStrategy(name: String, cap: Capability<&{StrategyInterface}>) {
        pre { self.strategies[name] == nil: "already exists" }
        self.strategies[name] = cap
        emit StrategyAdded(name: name, admin: self.account.address)
    }
    access(Admin) fun revokeStrategy(name: String, reason: String) {
        pre { self.strategies[name] != nil: "not found" }
        self.strategies.remove(key: name)
        emit StrategyRevoked(name: name, admin: self.account.address, reason: reason)
    }
}
```

Every `add` has a paired `revoke`, callable by the same entitlement, so the same governance path that approves can un-approve. Revocation captures a `reason` for downstream observability.

### Detection hint

For every `addStrategy` / `register` / `approveStrategy` in the registry, search for a matching `revokeStrategy` / `removeStrategy` / `deregister` with the **same entitlement**. Absence is the smell. Bonus: if revocation exists but requires a stricter entitlement than addition (e.g. `Admin` adds but `SuperAdmin` revokes), that's also a smell — the same governance authority should be able to undo its own approvals.

---

## S3 — Not validating capability `Type` at registration (Critical)

The admin can register any capability that compiles. There is no on-chain check that the capability actually resolves to a value implementing the expected `StrategyInterface`. A malformed or hostile capability is accepted and stored, then panics or misbehaves at execute time.

### Bad

```cadence
import "StrategyInterface"

access(all) contract StrategyRegistry {
    access(all) entitlement Admin
    access(self) let strategies: {String: Capability<&{StrategyInterface}>}

    // BAD — accepts any capability, no resolution check
    access(Admin) fun addStrategy(
        name: String,
        cap: Capability<&{StrategyInterface}>,
    ) {
        self.strategies[name] = cap
    }
}
```

### Why it's bad

The declared type on the parameter is a static promise, but `Capability` values are runtime entities created against any storage path. A capability against `/storage/NonExistent` stores fine and `borrow()` returns `nil` only at execute time. A capability against the wrong type panics at the user's call site. Either way, invalid state is stored and the failure surfaces inside the user transaction that tries to use it.

### Correct

```cadence
access(Admin) fun addStrategy(name: String, cap: Capability<&{StrategyInterface}>) {
    pre {
        cap.check(): "capability does not resolve to a StrategyInterface"
        self.strategies[name] == nil: "duplicate name"
    }
    // Borrow once to confirm the concrete type also conforms.
    let probe = cap.borrow() ?? panic("borrow failed despite check()")
    // Optional: assert a strategy-specific identifier (kind, version).
    assert(probe.kind.length > 0, message: "missing strategy kind")
    self.strategies[name] = cap
    emit StrategyAdded(name: name, admin: self.account.address, kind: probe.kind)
}
```

`cap.check()` exercises the resolution path and returns `false` if the storage path is empty or the stored value does not statically conform. Borrowing once on registration surfaces type mismatches at registration time, not at first user call.

### Detection hint

In every `addStrategy`-like function, look for `cap.check()` (or `cap.borrow() != nil`) in a `pre` block. Its absence — or a comment saying "we trust the admin" — is the smell. Also verify the function's parameter type is the **most specific** capability type the strategy must conform to, never `Capability<AnyResource>` or `Capability<&AnyResource>`.

---

## S4 — Treating "active" as the only state (Medium)

Strategies are binary: either in the map or not. There is no way to pause a strategy temporarily without losing its position, its accumulated state, or its reference identity. Operators are forced to choose between "leave a known-bad strategy live" and "revoke it and tear down whatever state references the cap".

### Bad

```cadence
access(all) contract StrategyRegistry {
    access(self) let strategies: {String: Capability<&{StrategyInterface}>}

    // BAD — only Active or absent; no Paused, no Deprecated
    access(Admin) fun addStrategy(name: String, cap: Capability<&{StrategyInterface}>) {
        self.strategies[name] = cap
    }
    access(Admin) fun revokeStrategy(name: String) {
        self.strategies.remove(key: name)
    }
}
```

### Why it's bad

A binary state forces a choice between "users can still deposit into the bad strategy" and "wipe the entry, breaking any contract holding a name-based reference". Mid-incident, the right move is usually "stop new deposits, allow existing positions to withdraw" — which a binary registry cannot express. Forcing a revoke also destroys forensic context: indexers lose the cap and dashboards lose the name.

### Correct

```cadence
access(all) contract StrategyRegistry {
    access(all) entitlement Admin

    access(all) enum Lifecycle: UInt8 {
        access(all) case Active        // accepts new deposits and executes
        access(all) case Paused        // executes existing positions, refuses new entries
        access(all) case Deprecated    // refuses new entries, allows withdrawals only
        access(all) case Revoked       // refuses everything; entry kept for forensics
    }

    access(all) struct Entry {
        access(all) let cap: Capability<&{StrategyInterface}>
        access(all) var state: Lifecycle
        init(cap: Capability<&{StrategyInterface}>) {
            self.cap = cap; self.state = Lifecycle.Active
        }
        access(contract) fun setState(_ s: Lifecycle) { self.state = s }
    }
    access(self) let strategies: {String: Entry}

    access(all) event StrategyStateChanged(name: String, from: UInt8, to: UInt8)

    access(Admin) fun setLifecycle(name: String, to: Lifecycle) {
        let entry = self.strategies[name] ?? panic("not found")
        let from = entry.state
        entry.setState(to)
        self.strategies[name] = entry
        emit StrategyStateChanged(name: name, from: from.rawValue, to: to.rawValue)
    }
}
```

Execution paths consult `state`: `Active` permits any operation; `Paused` only existing-position calls; `Deprecated` only withdrawals; `Revoked` rejects everything. The cap is retained so indexers and dashboards keep their references.

### Detection hint

Grep for the strategy-state field type. If it's `Bool`, `{String: Capability<...>}`, or a presence check, that's the smell. Healthy registries carry an enum or status field with at least three states (active / paused / deprecated). Verify every execute-side function reads the state and gates behavior on it — not just on the presence of the key.

---

## S5 — Capability borrowed at registration, executed at use (High)

The registry stores `cap.borrow()` — a `&{StrategyInterface}` reference — instead of the `Capability` itself. References capture the resolution at the moment of borrow and become invalid if the underlying resource is moved, replaced, or if the strategy contract is upgraded in a way that changes its layout. Execute-time calls silently fail with `nil` dereferences or panic with stale references.

### Bad

```cadence
access(all) contract StrategyRegistry {
    access(all) entitlement Admin

    // BAD — storing the borrowed reference, not the capability
    access(self) let strategies: {String: &{StrategyInterface}}

    access(Admin) fun addStrategy(name: String, cap: Capability<&{StrategyInterface}>) {
        let ref = cap.borrow() ?? panic("borrow failed")
        self.strategies[name] = ref     // freezes the reference
    }

    access(all) fun execute(name: String, amount: UFix64) {
        let strategy = self.strategies[name] ?? panic("not found")
        strategy.execute(amount: amount)    // reference may be stale
    }
}
```

### Why it's bad

`Capability` is the durable handle; references are transient. References are meant to be borrowed within a single transaction and discarded. After a strategy upgrade, an unpublish/republish, or any controller revocation, the stored reference either dereferences into a different value (silent corruption) or panics on use. Neither is acceptable for a registry that mediates user funds.

### Correct

```cadence
access(all) contract StrategyRegistry {
    access(all) entitlement Admin
    // GOOD — store the capability; borrow at execute time
    access(self) let strategies: {String: Capability<&{StrategyInterface}>}

    access(Admin) fun addStrategy(name: String, cap: Capability<&{StrategyInterface}>) {
        pre { cap.check(): "capability does not resolve" }
        self.strategies[name] = cap
    }
    access(all) fun execute(name: String, amount: UFix64) {
        let cap = self.strategies[name] ?? panic("not found")
        let strategy = cap.borrow() ?? panic("strategy no longer resolves")
        strategy.execute(amount: amount)
    }
}
```

Capability is the unit stored. Borrowing at execute time gets a fresh reference reflecting current chain state; `nil` from `borrow()` triggers an observable failure path.

### Detection hint

In the registry's storage declaration, look for the type of the strategies map / dictionary. If it's `{K: &Something}` or `{K: AnyResource}` rather than `{K: Capability<...>}`, that's the smell. Also flag any function that captures `cap.borrow()` into a contract field or stored struct — references must never escape the transaction in which they were borrowed.

---

## S6 — No usage caps or rate limits on strategies (Medium)

Once a strategy is approved, anyone with access to the registry's execute path can call it arbitrarily often. A buggy strategy that leaks 0.1% per call becomes a 100% drain over enough calls; a malicious strategy that survived the audit window can drain TVL in a single block.

### Bad

```cadence
access(all) contract StrategyRegistry {
    access(self) let strategies: {String: Capability<&{StrategyInterface}>}

    // BAD — no per-block / per-tx / per-user limits
    access(all) fun execute(name: String, amount: UFix64) {
        let cap = self.strategies[name] ?? panic("not found")
        let strategy = cap.borrow() ?? panic("borrow failed")
        strategy.execute(amount: amount)
    }
}
```

### Why it's bad

The registry is the choke point between the protocol vault and external compute. Without usage caps, it inherits the worst-case behavior of every strategy in the whitelist. "Safe under normal usage" and "safe under adversarial repetition in a single block" are different invariants — only the registry can enforce protocol-wide limits.

### Correct

```cadence
access(all) contract StrategyRegistry {
    access(all) entitlement Admin

    access(all) struct UsageCaps {
        access(all) let perTx: UFix64
        access(all) let perBlock: UFix64
        access(all) let perUserPerBlock: UFix64
    }
    access(self) let caps: {String: UsageCaps}
    access(self) let blockUsage: {String: {UInt64: UFix64}}
    access(self) let userBlockUsage: {String: {Address: {UInt64: UFix64}}}

    access(Admin) fun setCaps(name: String, caps: UsageCaps) { self.caps[name] = caps }

    access(all) fun execute(name: String, user: Address, amount: UFix64) {
        let cap = self.caps[name] ?? panic("uncapped strategy refused")
        assert(amount <= cap.perTx, message: "perTx cap exceeded")
        let height = getCurrentBlock().height
        let blockUsed = self.blockUsage[name]?[height] ?? 0.0
        assert(blockUsed + amount <= cap.perBlock, message: "perBlock cap exceeded")
        let userUsed = self.userBlockUsage[name]?[user]?[height] ?? 0.0
        assert(userUsed + amount <= cap.perUserPerBlock, message: "perUserPerBlock cap exceeded")
        // ... update usage maps, then dispatch ...
    }
}
```

The registry enforces three orthogonal caps. Each is bypassable individually (split across wallets), but together they bound damage: per-tx caps a single mistake, per-block caps a coordinated drain, per-user-per-block raises Sybil cost.

### Detection hint

In the registry's `execute` path, search for `getCurrentBlock().height` or `perTx`/`perBlock` constants. Absent — strategy executes are uncapped. Also look for whether the cap is per-strategy (good) or a single global cap (weaker — high-volume strategies push the cap up and bad ones inherit the same headroom).

---

## S7 — Strategy upgrades silently (High)

A previously-audited strategy contract is replaced via `account.contracts.update()` (or `add()` overwriting the same name on the deployer's account). The capability in the registry continues to resolve, the type still conforms, and the registry has no idea that the executable code behind it changed. A strategy that was safe at audit time can become a drainer after one deployer-side update.

### Bad

```cadence
access(all) contract StrategyRegistry {
    access(self) let strategies: {String: Capability<&{StrategyInterface}>}

    // BAD — no upgrade detection
    access(Admin) fun addStrategy(name: String, cap: Capability<&{StrategyInterface}>) {
        self.strategies[name] = cap
    }
}
```

### Why it's bad

Cadence contracts are upgradeable by their deployer. Upgrade rules forbid removing public symbols or breaking stored-value types, but within those rules the executable body of a function can change completely — a `withdraw` that yesterday paid the user can today pay an attacker. The registry's stored capability still resolves and the interface check still passes; nothing in on-chain state signals that re-approval is needed.

### Correct

```cadence
access(all) contract StrategyRegistry {
    access(all) entitlement Admin

    access(all) struct Entry {
        access(all) let cap: Capability<&{StrategyInterface}>
        access(all) let approvedKind: String       // strategy-reported version/hash
        init(cap: Capability<&{StrategyInterface}>, kind: String) {
            self.cap = cap; self.approvedKind = kind
        }
    }
    access(self) let strategies: {String: Entry}

    access(all) event StrategyKindMismatch(name: String, approved: String, observed: String)

    access(Admin) fun addStrategy(name: String, cap: Capability<&{StrategyInterface}>) {
        pre { cap.check(): "no resolve" }
        let probe = cap.borrow() ?? panic("borrow")
        self.strategies[name] = Entry(cap: cap, kind: probe.kind)
    }
    access(all) fun execute(name: String, amount: UFix64) {
        let entry = self.strategies[name] ?? panic("not found")
        let strategy = entry.cap.borrow() ?? panic("borrow")
        if strategy.kind != entry.approvedKind {
            emit StrategyKindMismatch(name: name, approved: entry.approvedKind, observed: strategy.kind)
            panic("strategy code identifier changed since approval; re-approve required")
        }
        strategy.execute(amount: amount)
    }
}
```

The strategy publishes a self-reported `kind` (version string, commit hash, build identifier) through the interface. The registry records the value at approval and re-checks on every execute. Any mismatch halts execution and emits an observable event. The strategy contract is responsible for bumping `kind` on every meaningful upgrade — the strategy's own audit checklist must include "does `kind` change on upgrade".

### Detection hint

In the registry, look for any persisted identifier-of-truth derived from the strategy (kind, version, hash). Absence means the registry is blind to upgrades. In the strategy interface, look for a `view` field or function (`access(all) let kind: String`) that downstream registries can use. If the interface is silent on versioning, every registry that consumes it inherits this anti-pattern.

---

## S8 — Admin-key compromise = full takeover (Critical)

Admin actions (add, revoke, pause, set caps) execute atomically with no time delay and no second-signature requirement. A single compromised key can install a draining strategy and execute it in the same block — the audit window is zero.

### Bad

```cadence
access(all) contract StrategyRegistry {
    access(all) entitlement Admin
    access(self) let strategies: {String: Capability<&{StrategyInterface}>}

    // BAD — one key, one call, immediate effect
    access(Admin) fun addStrategy(name: String, cap: Capability<&{StrategyInterface}>) {
        self.strategies[name] = cap
    }
}
```

### Why it's bad

Strategy registries are high-value targets: an attacker who controls the registry controls the dispatch of user funds across every protocol that depends on it. Bearer-key admin models concentrate that authority in a single signature, which becomes the single point of failure. Entitlement narrowing (see `forte-anti-patterns.md` A5) helps but is upstream of the problem: even narrow entitlements are too broad when one key holds them.

### Correct

```cadence
access(all) contract StrategyRegistry {
    access(all) entitlement Propose
    access(all) entitlement Approve
    access(all) let minApprovals: Int          // e.g. 2
    access(all) let actionDelay: UFix64        // e.g. 86_400.0

    access(all) struct PendingAction {
        access(all) let kind: String           // "add" | "revoke" | "setCaps"
        access(all) let payload: AnyStruct
        access(all) let proposedAt: UFix64
        access(all) var approvers: [Address]
        init(kind: String, payload: AnyStruct) {
            self.kind = kind; self.payload = payload
            self.proposedAt = getCurrentBlock().timestamp; self.approvers = []
        }
        access(contract) fun addApprover(_ a: Address) {
            assert(!self.approvers.contains(a), message: "duplicate approval")
            self.approvers.append(a)
        }
    }
    access(self) let pending: {UInt64: PendingAction}

    access(Propose) fun propose(id: UInt64, kind: String, payload: AnyStruct) {
        self.pending[id] = PendingAction(kind: kind, payload: payload)
    }
    access(Approve) fun approve(id: UInt64, approver: Address) {
        let a = self.pending[id] ?? panic("no such action")
        a.addApprover(approver); self.pending[id] = a
    }
    access(Approve) fun executeAction(id: UInt64) {
        let a = self.pending[id] ?? panic("no such action")
        assert(getCurrentBlock().timestamp >= a.proposedAt + self.actionDelay, message: "delay not elapsed")
        assert(a.approvers.length >= self.minApprovals, message: "insufficient approvals")
        self.pending.remove(key: id)
        // ... dispatch by a.kind ...
    }
}
```

The pattern combines (1) a time delay between proposal and execution, (2) a minimum number of distinct approvers, and (3) per-action granularity. An attacker must compromise both the delay window and the approver threshold simultaneously. For production, replace the per-action approver list with a multi-sig account holding the `Approve` entitlement and rotate keys regularly.

### Detection hint

Grep for `access(Admin)` in the registry. For each, check whether the function takes effect immediately or routes through a `propose` → `delay` → `execute` flow. Immediate-effect admin functions on a registry are a Critical finding unless the admin authority itself is a multi-sig account. Also look for `Capability<auth(Admin) &Registry>` anywhere it might be published — that capability is the takeover vector.

---

## S9 — Events absent or sparse (Low)

Additions, revocations, and lifecycle changes either emit no events or emit events without enough context to reconstruct who did what when. Off-chain audit trail is invisible; incident response cannot answer "which admin approved this strategy" or "when did the state change to Deprecated".

### Bad

```cadence
access(all) contract StrategyRegistry {
    access(all) entitlement Admin
    access(self) let strategies: {String: Capability<&{StrategyInterface}>}

    // BAD — no events
    access(Admin) fun addStrategy(name: String, cap: Capability<&{StrategyInterface}>) {
        self.strategies[name] = cap
    }

    // BAD — sparse event
    access(all) event Updated(name: String)
    access(Admin) fun revokeStrategy(name: String) {
        self.strategies.remove(key: name)
        emit Updated(name: name)        // no admin, no reason, no kind
    }
}
```

### Why it's bad

Strategy registries are exactly the state downstream indexers and dashboards rely on for protocol observability. An indexer subscribing to `StrategyAdded` / `StrategyRevoked` should reconstruct the current whitelist from events alone, without re-reading contract state. Sparse events (`Updated`, `Changed`) force every consumer to refetch state, defeating the purpose. Anonymous events (no `admin: Address`) make forensic attribution impossible.

### Correct

```cadence
access(all) contract StrategyRegistry {
    access(all) entitlement Admin

    access(all) event StrategyAdded(name: String, kind: String, admin: Address, approvedAt: UFix64)
    access(all) event StrategyRevoked(name: String, admin: Address, reason: String, revokedAt: UFix64)
    access(all) event StrategyStateChanged(name: String, from: UInt8, to: UInt8, admin: Address, changedAt: UFix64)

    access(Admin) fun addStrategy(name: String, cap: Capability<&{StrategyInterface}>) {
        pre { cap.check(): "no resolve" }
        let probe = cap.borrow() ?? panic("borrow")
        self.strategies[name] = cap
        emit StrategyAdded(
            name: name, kind: probe.kind,
            admin: self.account.address, approvedAt: getCurrentBlock().timestamp,
        )
    }
}
```

Every mutation emits a fully-qualified event with subject (`name`), agent (`admin`), context (`kind`, `reason`, lifecycle states), and timestamp. Past-tense `<Subject><Verb>` naming matches the event taxonomy expected by indexers.

### Detection hint

For each public mutator on the registry (`add`, `revoke`, `setLifecycle`, `setCaps`, governance actions), grep for an `emit` on the same code path. Each `emit` should reference (a) the strategy identifier, (b) the admin address, (c) the action context (`kind`, `from`/`to`, `reason`), and (d) a timestamp. Generic `Updated` / `Changed` events are smells. Missing `emit` entirely is the strongest smell — audit trail is gone.

---

## Audit Checklist for Strategy Registries

For any contract that maintains a whitelist of strategy capabilities, the auditor must answer **yes** to every question — or document a deliberate exception.

### Registration
- [ ] Does `addStrategy` (or equivalent) call `cap.check()` in a pre-condition and reject when it returns false?
- [ ] Does the function additionally `borrow()` the capability and assert a strategy-specific identifier (kind, version) before storing?
- [ ] Is the capability parameter typed as the **most specific** interface the strategy must conform to (never `Capability<AnyResource>`)?
- [ ] Does registration require a two-step `propose` → delay → `finalize` flow, or is single-call approval explicitly justified?

### Lifecycle
- [ ] Is there a `revokeStrategy` callable by the same entitlement that added it?
- [ ] Does the registry distinguish at least three lifecycle states (Active, Paused, Deprecated/Revoked) via an enum or status field?
- [ ] Do execute paths consult the lifecycle state and gate operations on it (e.g. Paused refuses new entries, Deprecated allows withdrawals only)?
- [ ] Are revoked entries retained for forensic purposes, or completely removed (and is the choice deliberate)?

### Execution
- [ ] Does the registry store `Capability<...>` values rather than borrowed `&` references?
- [ ] Is every execute path's `borrow()` checked for `nil` and routed to an observable failure (not silent skip)?
- [ ] Are per-tx, per-block, and per-user usage caps enforced by the registry (not delegated to the strategy)?
- [ ] Is the strategy's self-reported identifier (`kind` / `version`) re-checked on every execute against the approved value, with mismatch halting execution?

### Events
- [ ] Does every mutator (`add`, `revoke`, `setLifecycle`, `setCaps`, governance actions) emit an event on the same code path?
- [ ] Does each event include the strategy identifier, the admin address, the action context, and a timestamp?
- [ ] Are event names in past-tense `<Subject><Verb>` form (`StrategyAdded`, `StrategyRevoked`, `StrategyStateChanged`) — never generic (`Updated`, `Changed`)?

### Governance
- [ ] Are all `access(Admin)` (or equivalent) functions routed through a time delay before taking effect?
- [ ] Does the registry require multiple approvers (multi-sig or contract-level approver list) for strategy additions and capability-cap changes?
- [ ] Is the admin account itself a multi-sig, and are its keys rotated on a documented schedule?
- [ ] Are admin capabilities stored privately (never published), and are their controllers tracked for revocation?

## See also

- [strategy-registry.md](../../cadence-lang/references/strategy-registry.md) — canonical pattern (proposal flow, lifecycle states, capability storage, usage caps, kind identifier, governance model)
- `cadence-audit/references/audit-checklist.md` — general Cadence audit checklist (apply in addition to this file)
- `cadence-audit/references/forte-anti-patterns.md` — scheduled-transaction anti-patterns (A5 entitlement narrowing is upstream of S5 / S8 here)
- `cadence-lang/references/capabilities.md` — capability semantics, controllers, revocation
- `cadence-lang/references/entitlements.md` — entitlement design for two-step governance flows
