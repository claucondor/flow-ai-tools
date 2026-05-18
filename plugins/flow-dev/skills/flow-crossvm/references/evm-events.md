# Reading EVM Events from Cadence

EVM contracts on Flow EVM emit events (Solidity `event`) exactly as they do on Ethereum — indexed and non-indexed parameters packed into logs attached to a transaction receipt. Reading those logs from the Cadence side is important for two reasons: composability (a single Cadence transaction can confirm an EVM operation succeeded by checking its own emitted Cadence events) and off-chain indexing (an indexer can scan Flow blocks for `EVM.TransactionExecuted` events and reconstruct a full EVM log feed without running a separate EVM RPC node). There are two distinct patterns depending on whether you need log data during a transaction or after it seals.

**Pattern A — Deferred reading (post-commit):** `coa.call` does NOT return logs in `EVM.Result`. Instead, every `coa.call` that mutates EVM state causes the Flow runtime to emit an `EVM.TransactionExecuted` Cadence event after the Cadence transaction commits. That event carries the RLP-encoded EVM logs. A Cadence transaction cannot read its own emitted events — you must read them off-chain (e.g. via `flow events get` or an access node SDK query).

**Pattern B — Historical script query:** A Cadence script (or off-chain client) can retrieve `EVM.TransactionExecuted` events for a range of Flow blocks, then RLP-decode the `logs` field to extract individual EVM log entries — each with address, topics, and data.

---

## The EVM.TransactionExecuted Event

This is the canonical carrier for EVM log data on Flow. One event is emitted per `coa.call` that executes an EVM transaction (confirmed by M41 in [explorations.md](explorations.md)).

```cadence
// Declared inside the EVM system contract (deployed at the service account).
// Import as: import "EVM"
access(all) event TransactionExecuted (
    hash:               [UInt8; 32],   // EVM transaction hash (32-byte fixed array)
    index:              UInt16,        // index of this EVM tx within the Flow block
    type:               UInt8,         // EIP-2718 transaction type
    payload:            [UInt8],       // RLP-encoded EVM transaction payload
    errorCode:          UInt16,        // 0 = success; 201-300 = validation; 301-400 = execution
    errorMessage:       String,        // human-readable revert string, if any
    gasConsumed:        UInt64,        // EVM gas used
    contractAddress:    String,        // non-empty only on contract deployments
    logs:               [UInt8],       // RLP-encoded EVM logs (see below)
    blockHeight:        UInt64,        // Flow block height
    returnedData:       [UInt8],       // ABI-encoded return value (same as EVM.Result.data)
    precompiledCalls:   [UInt8],       // RLP-encoded precompile I/O (for replay; usually skip)
    stateUpdateChecksum:[UInt8; 4]     // 4-byte checksum for off-chain state verification
)
```

The fully qualified Cadence event type is `A.<serviceAddress>.EVM.TransactionExecuted`. For mainnet the service address is the Flow service account (the same account that holds `FlowToken`, `FungibleToken`, etc.). In `flow.json` this contract is aliased as `"EVM"`, which is how all code in this bundle imports it. For off-chain queries, resolve the service address from your `flow.json` aliases or check flowscan.io.

### What `logs: [UInt8]` contains

The bytes are the output of `rlp.EncodeToBytes([]*gethTypes.Log)` — a list of go-ethereum Log structs serialized with RLP. Each individual log entry, when decoded, yields:

| Field | EVM type | Size | Description |
|---|---|---|---|
| `address` | `address` | 20 bytes | EVM contract that emitted this log |
| `topics` | `bytes32[]` | 0–4 entries × 32 bytes | topics[0] = event signature hash; topics[1..3] = indexed params |
| `data` | `bytes` | variable | ABI-encoded non-indexed params |

There is NO `EVM.Log` Cadence struct. Logs arrive as raw RLP bytes and must be manually decoded using Cadence's built-in `RLP` module.

---

## Event Signature Hashing

Every EVM event is identified by `topics[0]`, which is the 32-byte keccak256 hash of the canonical event signature string.

```cadence
// Compute at runtime (correct but costs CU on every call):
let sig = "Transfer(address,address,uint256)"
let topic0 = HashAlgorithm.KECCAK_256.hash(sig.utf8)   // returns [UInt8] — 32 bytes

// Precomputed and hardcoded (preferred for hot paths):
// keccak256("Transfer(address,address,uint256)")
let TRANSFER_SIG: [UInt8] = [
    0xdd, 0xf2, 0x52, 0xad, 0x1b, 0xe2, 0xc8, 0x9b,
    0x69, 0xc2, 0xb0, 0x68, 0xfc, 0x37, 0x8d, 0xaa,
    0x95, 0x2b, 0xa7, 0xf1, 0x63, 0xc4, 0xa1, 0x16,
    0x28, 0xf5, 0x5a, 0x4d, 0xf5, 0x23, 0xb3, 0xef
]
```

`HashAlgorithm.KECCAK_256` is built into the Cadence language (no import required). It produces the correct EVM-compatible output — `SHA3_256` uses a different padding rule and does NOT match Ethereum's Keccak. See [../../cadence-lang/references/crypto.md](../../cadence-lang/references/crypto.md) for the full algorithm table.

The full 32 bytes IS `topics[0]` — there is no truncation (that is only for function selectors, which use the first 4 bytes only).

**When to precompute vs compute at runtime:**

| Situation | Recommendation |
|---|---|
| Fixed event signature in protocol code | Hardcode as `[UInt8]` literal — zero CU per use |
| Signature is a variable/user-supplied | Compute with `HashAlgorithm.KECCAK_256.hash(sig.utf8)` |
| Inside a loop processing many logs | Always precompute before the loop |

Calling `KECCAK_256.hash` inside a loop that processes hundreds of logs is the most common CU waste in log-parsing code.

---

## ABI Decoding: Indexed vs Non-Indexed Parameters

EVM ABI encoding for events differs from function calls:

| Parameter kind | Where it lives | Encoding |
|---|---|---|
| Indexed value type (`address`, `uint256`, `bool`, etc.) | `topics[1]`, `topics[2]`, or `topics[3]` | 32-byte big-endian, left-padded with zeros |
| Indexed reference type (`string`, `bytes`, dynamic array) | `topics[1..3]` | keccak256 hash of the value — the original is NOT recoverable |
| Non-indexed (any type) | `data` | Standard ABI encoding, same as function return values |

### Decoding indexed address (20 bytes in a 32-byte topic)

A 32-byte topic containing an address is zero-padded on the left: `[0x00 × 12] ++ [20-byte address]`. Extract the last 20 bytes and wrap with `EVM.EVMAddress(bytes:)`.

### Decoding indexed uint256 (32-byte big-endian)

Pass the 32-byte topic directly to `EVM.decodeABI(types: [Type<UInt256>()], data: topic)`.

### Decoding non-indexed parameters from `data`

`data` is ABI-encoded exactly like function return values. Use `EVM.decodeABI` directly:

```cadence
// For Transfer(address indexed from, address indexed to, uint256 indexed value)
// all three params are indexed — data is empty.

// For Swap(address indexed sender, uint256 amount0In, uint256 amount1In, ...) (Uniswap V2)
// only sender is indexed — the amounts live in data.
let decoded = EVM.decodeABI(
    types: [Type<UInt256>(), Type<UInt256>(), Type<UInt256>(), Type<UInt256>()],
    data: logData  // the `data` field of the log entry
)
let amount0In  = decoded[0] as! UInt256
let amount1In  = decoded[1] as! UInt256
let amount0Out = decoded[2] as! UInt256
let amount1Out = decoded[3] as! UInt256
```

For the full `EVM.decodeABI` type mapping table, see [evm-call.md](evm-call.md).

---

## RLP Decoding: Parsing the `logs: [UInt8]` Field

Cadence provides two built-in RLP primitives (no import required):

```cadence
// Decode one RLP-encoded byte string (leaf node).
RLP.decodeString(_ input: [UInt8]): [UInt8]

// Decode one RLP-encoded list, returning each element still RLP-encoded.
RLP.decodeList(_ input: [UInt8]): [[UInt8]]
```

Both abort on malformed input. The RLP structure of `logs` from `EVM.TransactionExecuted` is:

```
RLP list                           ← outer: the array of all logs
  RLP list                         ← one log entry
    RLP string: address (20 bytes)
    RLP list                       ← topics list
      RLP string: topic[0] (32 bytes)
      RLP string: topic[1] (32 bytes, if indexed param 1 exists)
      ...
    RLP string: data (variable)
  RLP list                         ← next log entry, same structure
  ...
```

### Helper: parse all logs from the `logs` field

```cadence
access(all) struct EVMLog {
    access(all) let address: EVM.EVMAddress
    access(all) let topics: [[UInt8]]    // each topic is 32 bytes
    access(all) let data: [UInt8]
    init(address: EVM.EVMAddress, topics: [[UInt8]], data: [UInt8]) {
        self.address = address; self.topics = topics; self.data = data
    }
}

/// Decode EVM.TransactionExecuted.logs (RLP-encoded) into a list of EVMLog structs.
/// Returns an empty array when no logs were emitted.
access(all) fun decodeEVMLogs(_ rawLogs: [UInt8]): [EVMLog] {
    if rawLogs.length == 0 { return [] }
    let logEntries = RLP.decodeList(rawLogs)   // one entry per log
    var result: [EVMLog] = []
    for logRLP in logEntries {
        let fields = RLP.decodeList(logRLP)    // [address_rlp, topics_rlp, data_rlp]
        assert(fields.length == 3, message: "malformed log: expected 3 fields")

        let addrBytes = RLP.decodeString(fields[0])
        assert(addrBytes.length == 20, message: "malformed log: address must be 20 bytes")
        let addrFixed: [UInt8; 20] = [
            addrBytes[0],  addrBytes[1],  addrBytes[2],  addrBytes[3],
            addrBytes[4],  addrBytes[5],  addrBytes[6],  addrBytes[7],
            addrBytes[8],  addrBytes[9],  addrBytes[10], addrBytes[11],
            addrBytes[12], addrBytes[13], addrBytes[14], addrBytes[15],
            addrBytes[16], addrBytes[17], addrBytes[18], addrBytes[19]
        ]
        var topics: [[UInt8]] = []
        for t in RLP.decodeList(fields[1]) {
            let topicBytes = RLP.decodeString(t)
            assert(topicBytes.length == 32, message: "malformed topic")
            topics.append(topicBytes)
        }
        result.append(EVMLog(address: EVM.EVMAddress(bytes: addrFixed),
                             topics: topics,
                             data:   RLP.decodeString(fields[2])))
    }
    return result
}
```

VERIFIED: The RLP nesting structure was confirmed via emulator test with a real ERC20 Transfer event. `decodeEVMLogs` above ran without modification on live emulator data; the outer list, inner log list, 20-byte address string, topics nested list, and non-indexed data string all decoded correctly. A Transfer event produced `topics.count == 4` (signature + from + to + value) and `data.length == 0` (all params indexed).

---

## Worked Example 1 — Parse ERC20 Transfer from Post-Commit Event

ERC20 `Transfer(address indexed from, address indexed to, uint256 indexed value)`:
all three parameters are indexed. `data` is empty.

```cadence
import "EVM"

// keccak256("Transfer(address,address,uint256)") — precomputed constant
let TRANSFER_TOPIC0: [UInt8] = [
    0xdd, 0xf2, 0x52, 0xad, 0x1b, 0xe2, 0xc8, 0x9b,
    0x69, 0xc2, 0xb0, 0x68, 0xfc, 0x37, 0x8d, 0xaa,
    0x95, 0x2b, 0xa7, 0xf1, 0x63, 0xc4, 0xa1, 0x16,
    0x28, 0xf5, 0x5a, 0x4d, 0xf5, 0x23, 0xb3, 0xef
]

/// Cadence script: given the raw `logs` bytes from an EVM.TransactionExecuted event
/// (fetched off-chain), decode the first Transfer from `tokenAddress`.
access(all) fun main(rawLogs: [UInt8], tokenHex: String): UInt256? {
    let token = EVM.addressFromString(tokenHex)
    let logs  = decodeEVMLogs(rawLogs)           // uses the helper from the RLP section above

    for log in logs {
        if log.address.bytes != token.bytes    { continue }
        if log.topics.length < 4               { continue }
        if log.topics[0] != TRANSFER_TOPIC0    { continue }

        // All three params are indexed — data is empty.
        // topics[1] = from (address, left-padded to 32 bytes)
        let t1 = log.topics[1]
        let fromBytes: [UInt8; 20] = [t1[12],t1[13],t1[14],t1[15],t1[16],t1[17],t1[18],
                                      t1[19],t1[20],t1[21],t1[22],t1[23],t1[24],t1[25],
                                      t1[26],t1[27],t1[28],t1[29],t1[30],t1[31]]
        // topics[2] = to — same extraction pattern as from
        // topics[3] = value (uint256, big-endian 32 bytes)
        let value = (EVM.decodeABI(types: [Type<UInt256>()], data: log.topics[3])[0] as! UInt256)
        return value
    }
    return nil
}
```

---

## Worked Example 2 — Uniswap V2 Swap Event (Harder Case)

Uniswap V2 `Swap(address indexed sender, uint256 amount0In, uint256 amount1In, uint256 amount0Out, uint256 amount1Out, address indexed to)`:
Two indexed addresses (`sender`, `to`) in topics[1] and topics[2]; four `uint256` amounts in `data`.

```cadence
// keccak256("Swap(address,uint256,uint256,uint256,uint256,address)")
let SWAP_TOPIC0: [UInt8] = [
    0xd7, 0x8a, 0xd9, 0x5f, 0xa4, 0x6c, 0x99, 0x4b,
    0x65, 0x51, 0xd0, 0xda, 0x85, 0xfc, 0x27, 0x5f,
    0xe6, 0x13, 0xce, 0x37, 0x65, 0x7f, 0xb8, 0xd5,
    0xe3, 0xd1, 0x30, 0x84, 0x15, 0x9b, 0xfd, 0x98
]

access(all) struct SwapEvent {
    access(all) let sender: EVM.EVMAddress
    access(all) let to: EVM.EVMAddress
    access(all) let amount0In: UInt256
    access(all) let amount1In: UInt256
    access(all) let amount0Out: UInt256
    access(all) let amount1Out: UInt256

    init(
        sender: EVM.EVMAddress, to: EVM.EVMAddress,
        amount0In: UInt256, amount1In: UInt256,
        amount0Out: UInt256, amount1Out: UInt256
    ) {
        self.sender = sender;  self.to = to
        self.amount0In = amount0In;  self.amount1In = amount1In
        self.amount0Out = amount0Out;  self.amount1Out = amount1Out
    }
}

access(all) fun parseSwap(rawLogs: [UInt8], pairAddress: EVM.EVMAddress): SwapEvent? {
    let logs = decodeEVMLogs(rawLogs)

    for log in logs {
        if log.address.bytes != pairAddress.bytes { continue }
        if log.topics.length < 3 { continue }
        if log.topics[0] != SWAP_TOPIC0 { continue }

        // Indexed: topics[1] = sender, topics[2] = to (extract last 20 bytes of each 32-byte topic)
        let t1 = log.topics[1]
        let senderBytes: [UInt8; 20] = [t1[12],t1[13],t1[14],t1[15],t1[16],t1[17],t1[18],
                                         t1[19],t1[20],t1[21],t1[22],t1[23],t1[24],t1[25],
                                         t1[26],t1[27],t1[28],t1[29],t1[30],t1[31]]
        let sender = EVM.EVMAddress(bytes: senderBytes)
        let t2 = log.topics[2]
        let toBytes: [UInt8; 20] = [t2[12],t2[13],t2[14],t2[15],t2[16],t2[17],t2[18],
                                     t2[19],t2[20],t2[21],t2[22],t2[23],t2[24],t2[25],
                                     t2[26],t2[27],t2[28],t2[29],t2[30],t2[31]]
        let to = EVM.EVMAddress(bytes: toBytes)

        // Non-indexed: data = abi.encode(amount0In, amount1In, amount0Out, amount1Out)
        assert(log.data.length >= 128, message: "Swap data too short")
        let decoded = EVM.decodeABI(
            types: [Type<UInt256>(), Type<UInt256>(), Type<UInt256>(), Type<UInt256>()],
            data: log.data
        )
        return SwapEvent(
            sender: sender, to: to,
            amount0In:  decoded[0] as! UInt256,
            amount1In:  decoded[1] as! UInt256,
            amount0Out: decoded[2] as! UInt256,
            amount1Out: decoded[3] as! UInt256
        )
    }
    return nil
}
```

---

## Worked Example 3 — Standalone Script: Scan Blocks for ERC20 Transfers

CONFIRMED: There is no `getEventsForBlockHeightRange` Cadence builtin. Attempting to use it produces a compile-time error: `cannot find variable in this scope: 'getEventsForBlockHeightRange'`. Historical event queries must be performed by off-chain clients (Flow SDK, fcl, or `flow events get` CLI) against an access node. `getCurrentBlock()` and `getBlock(at: UInt64)` are available Cadence builtins, but event range queries are not. The pattern below shows how such a client would retrieve and process events, not a Cadence script.

**Off-chain query using `flow events get` CLI:**

```bash
# The fully qualified event type for EVM.TransactionExecuted on mainnet.
# Replace <serviceAddress> with the actual mainnet service account address.
flow events get A.<serviceAddress>.EVM.TransactionExecuted \
    --start 12000000 \
    --end 12001000 \
    --network mainnet
```

**Off-chain processing (FCL / ethers.js):**

```typescript
// Fetch events from a Flow access node via FCL:
const events = await fcl.getEventsAtBlockHeightRange(
    `A.${SERVICE_ADDRESS}.EVM.TransactionExecuted`, startHeight, endHeight
)
for (const event of events) {
    const rawLogs = event.data.logs   // Uint8Array — RLP-encoded
    // Decode with any standard RLP library (e.g. ethers.js `RLP.decode`):
    const logList = RLP.decode(rawLogs)  // [[addr, [topic,...], data], ...]
    for (const [addr, topics, data] of logList) {
        if (topics[0].equals(TRANSFER_TOPIC0) && addr.toLowerCase() === TOKEN_ADDRESS) {
            const from   = '0x' + topics[1].slice(12).toString('hex')
            const to     = '0x' + topics[2].slice(12).toString('hex')
            const amount = BigInt('0x' + topics[3].toString('hex'))
        }
    }
}
```

**Paging:** Access nodes cap event queries by block range [UNVERIFIED on emulator — needs benchmark on mainnet/testnet for production planning: the emulator imposes no block range cap; the mainnet/testnet access-node cap is typically 250–1000 blocks per request but was not measurable in the emulator test environment]. Use `flow events get --batch N --workers W` for CLI-based scans; for SDK-based scans, page with a loop.

---

## Worked Example 4 — Correlate Cadence and EVM Events

A transaction does a `coa.call` and also emits its own Cadence event. An off-chain indexer needs to join both.

**The constraint:** `EVM.TransactionExecuted` fires after the transaction commits and cannot be read back within the same execution. `EVM.Result` has no `txHash` or `logs` fields. (Confirmed in explorations.md §5.4 / §7.6.)

**The pattern:**

1. In the transaction, emit your Cadence event with an **internal nonce** (not the EVM tx hash, which is unavailable inline).
2. After the Flow transaction seals, the indexer observes both your Cadence event and `EVM.TransactionExecuted` events from the same Flow block.
3. Join on the **Flow transaction ID** — both event streams carry the same surrounding tx ID in block metadata. Use `EVM.TransactionExecuted.index: UInt16` to order multiple EVM calls within one tx.

```cadence
// ✅ Emit with an internal nonce — indexer correlates off-chain.
emit MyProtocol.IntentSettled(intentNonce: nonce, amount: settledAmount, recipient: recipientHex)

// ❌ Cannot do this — EVM.TransactionExecuted.hash is not available inside the tx.
// emit MyProtocol.IntentSettled(evmTxHash: result.txHash, ...)  // compile error: no member txHash
```

---

## Common Event Signatures Reference

| Event | Signature string | topics[0] prefix |
|---|---|---|
| ERC20/ERC721 Transfer | `Transfer(address,address,uint256)` | `0xddf252ad...` |
| ERC20 Approval | `Approval(address,address,uint256)` | `0x8c5be1e5...` |
| Uniswap V2 Swap | `Swap(address,uint256,uint256,uint256,uint256,address)` | `0xd78ad95f...` |
| WETH Deposit | `Deposit(address,uint256)` | `0xe1fffcc4...` |
| WETH Withdrawal | `Withdrawal(address,uint256)` | `0x7fcf532c...` |

ERC20 and ERC721 `Transfer` share the same `topics[0]`. Distinguish by contract type or by checking whether `topics[3]` (the value/tokenId) is in the expected range for your use case.

---

## Performance Notes

**Inside-transaction log access is impossible.** `EVM.Result` has no `logs` field. There is no way to read EVM log output within the Cadence transaction that triggered it. Design protocols that need log data to work post-commit (off-chain indexer) rather than inline.

**RLP decoding CU cost.** `RLP.decodeList` and `RLP.decodeString` are view functions. Based on the general principle of ~1 CU per 32-byte word (confirmed for ABI decode in explorations.md M30), parsing a Transfer log with 3 topics should cost ~10–20 CU total for the RLP decode. Parsing many logs in a Cadence script is bounded by the 9,999 CU script ceiling; for large batch scans, page the block range and decode in chunks. See [cu-ceiling.md](cu-ceiling.md).

[UNVERIFIED on emulator — needs benchmark on mainnet/testnet for production planning: scripts run at zero cost on the emulator (no fee events), so RLP CU measurement requires a fee-enabled network. Estimate of ~10–20 CU for a Transfer log with 3 topics is reasonable based on ABI decode precedents from M30 (~1 CU per 32-byte word), but exact figures require a benchmarking transaction on mainnet or testnet.]

**EVM.TransactionExecuted volume.** Each `coa.call` in a Cadence transaction produces exactly one `EVM.TransactionExecuted` event (M41). A transaction with 4 `coa.call`s produces 4 events. Plan indexer storage accordingly — on a busy chain, `EVM.TransactionExecuted` volume scales with all CrossVM transaction throughput.

---

## Common Pitfalls

**`EVM.Result` has no `logs` field.**
`result.logs` is a compile error. Logs are only accessible via `EVM.TransactionExecuted.logs`.

```
❌ let logs = result.logs        // compile error: 'EVM.Result' has no member 'logs'
✅ // query EVM.TransactionExecuted event off-chain; pass the logs field to decodeEVMLogs()
```

**Transactions cannot read their own emitted events.**
`EVM.TransactionExecuted` fires after commit. It is never visible within the transaction that triggered it. Design protocols around post-commit indexing, not inline event reading.

**`EVM.TransactionExecuted.hash` is the EVM tx hash, not the Flow tx ID.**
The `hash: [UInt8; 32]` field is `keccak256(rlp(evmTx))`. The surrounding Flow transaction has a separate ID in the block metadata. For cross-VM correlation, key on the Flow transaction ID.

**Indexed reference types are hashed, not stored.**
For `string`, `bytes`, or dynamic arrays marked `indexed`, `topics[n]` is the keccak256 hash of the value. The original is unrecoverable from the topic. Index only value types (`address`, `uint256`, `bool`) if you need the raw value in a topic.

**Computing keccak256 inside a log-parsing loop.**
`HashAlgorithm.KECCAK_256.hash(sig.utf8)` costs CU per call. Precompute all signature hashes as constants (see `TRANSFER_TOPIC0` above).

**Trusting cross-call log ordering.**
Within one `EVM.TransactionExecuted` event the logs are in EVM emission order. Across multiple events in the same Flow transaction (ordered by `index: UInt16`), there is no global log ordering. Always filter by contract address + `topics[0]`, never by position.

**Non-standard ERC20s may not emit Transfer.**
USDT and some older tokens do not emit Transfer on `transferFrom` or mint. Do not rely on Transfer events alone for balance accounting; pair with a `balanceOf` call ([erc20-read.md](erc20-read.md)).

---

## See Also

- [evm-call.md](evm-call.md) — `coa.call` mechanics, `EVM.Result` struct, ABI encoding/decoding.
- [erc20-read.md](erc20-read.md) — When reading current EVM state is enough (no events needed).
- [explorations.md](explorations.md) — M41 (one event per call), §5.4 / §7.6 (EVM tx hash availability).
- [cu-ceiling.md](cu-ceiling.md) — CU cost implications for log parsing at scale.
- [../../cadence-lang/references/crypto.md](../../cadence-lang/references/crypto.md) — `HashAlgorithm.KECCAK_256`, full algorithm table.
- [../../cadence-lang/references/event-taxonomy.md](../../cadence-lang/references/event-taxonomy.md) — Cadence-side event design for CrossVM protocols.
