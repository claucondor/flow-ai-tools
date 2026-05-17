# Cadence Fixed-Point Numerics: Fix128 and UFix128

Cadence represents decimal numbers using *fixed-point* integers — the value is stored as a single 128-bit (or 64-bit) integer, scaled by a fixed factor `10^scale`. There is no floating-point type at any width. `Fix128` (signed) and `UFix128` (unsigned) were added in FLIP 341 (accepted 2025-08-29) and shipped in Cadence v1.7.0 (Sep 2025). They give a scale of 24 decimal places of fractional precision and a value range deep into the trillions, so high-precision financial math (compound interest, oracle prices, AMM invariants, vesting curves) no longer needs custom 256-bit integer scaffolding around `UFix64`.

Verified live on emulator, testnet, and mainnet via the bundled Flow CLI (v2.17.1, Cadence v1.10.x at the time of writing).

## At-a-Glance

| Type    | Width    | Scale (decimals) | Min                                                | Max                                              |
|---------|----------|------------------|----------------------------------------------------|--------------------------------------------------|
| `Fix64`  | 64-bit signed   | 8  | `-92233720368.54775808`                        | `92233720368.54775807`                       |
| `UFix64` | 64-bit unsigned | 8  | `0.0`                                          | `184467440737.09551615`                      |
| `Fix128` | 128-bit signed   | 24 | `-170141183460469.231731687303715884105728`    | `170141183460469.231731687303715884105727`   |
| `UFix128`| 128-bit unsigned | 24 | `0.0`                                          | `340282366920938.463463374607431768211455`   |
| `UInt256`| 256-bit unsigned integer | n/a | `0`                                       | `2^256 - 1` (no fractional component)        |

`Fix128Scale == 24`, so the storage factor is `1e24`. `Fix128One` is `1.000000000000000000000000`.

## Literals, Initialization, and Type Methods

### Literal syntax

Fixed-point literals require a decimal point. The fractional part can have up to `scale` digits — more triggers a static error.

```cadence
let a: Fix128 = -1.5                                     // signed literal
let b: UFix128 = 3.141592653589793238462643              // 24-digit precision
let c: UFix128 = 0.000000000000000000000001              // smallest positive UFix128
// let bad: UFix128 = 0.1234567890123456789012345        // static error: scale out of range (25 > 24)
```

### Type-level members

Each type exposes the same surface as other Cadence numeric types:

```cadence
let lo = UFix128.min                  // 0.000000000000000000000000
let hi = UFix128.max                  // 340282366920938.463463374607431768211455
let parsed = UFix128.fromString("3.14159265358979323846")  // Optional<UFix128>
let bytes = UFix128.fromBigEndianBytes(buf)               // Optional<UFix128>. Returns nil if input length > 16. Inputs of length 0-16 are left-padded with zeros (a short array silently decodes to zero — validate caller-side).
```

`fromString` returns `nil` on parse failure: non-numeric input, missing decimal point, scale > 24, out of range, any sign other than implicit-or-explicit-plus (so `"-1.0"` returns `nil` for `UFix128`), leading/trailing whitespace, and decimals with no digits on one side (`".5"`, `"1."`).

`fromBigEndianBytes` returns `nil` for inputs longer than 16 bytes. Inputs shorter than 16 bytes are zero-extended on the high side and decode without error — `fromBigEndianBytes([])` is `UFix128(0)`, not `nil`. Validate the byte-array length before calling if your protocol requires exact 16-byte fields. **Footgun**: oracle code, cross-VM ABI decoders, and any boundary that receives bytes from an untrusted source must check `bytes.length == 16` before calling, otherwise a malformed message decodes silently to zero.

### Constructor / conversion surface

```cadence
let x = UFix128(42)             // from any integer type (Int, UInt, Int*/UInt*, Int128/UInt128/Int256/UInt256)
let y = UFix128(1.5 as UFix64)  // exact widening conversion (UFix64 -> UFix128 is lossless)
let z = UFix128(Fix128(5.5))    // signed -> unsigned: reverts if source is negative
let w = Fix128(-1.5 as Fix64)   // exact widening conversion (Fix64 -> Fix128 is lossless)
let asU64 = UFix64(UFix128(1.5))// narrowing: truncates fractional digits beyond 8 toward zero; reverts on integer-or-fractional overflow above UFix64.max (e.g. UFix128(184467440737.095516159999999999999999) reverts)
let asI = Int(UFix128(42.99))   // takes integer part (42); reverts only if .integerPart doesn't fit
let asBI = UInt256(UFix128(42.99)) // takes integer part as UInt256 (42)
```

There is **no implicit conversion** between numeric types. Mixing `UFix64` and `UFix128` in `+`, `*`, comparisons, etc., is a static error. You must explicitly cast.

## Arithmetic, Comparison, and Saturation

All arithmetic on Cadence numeric types **reverts on overflow/underflow** by default. Cadence does not wrap or saturate silently.

Verified operations on `Fix128`:
- `+`, `-`, `*`, `/`, `%`, unary `-` (negation)
- `==`, `!=`, `<`, `<=`, `>`, `>=`
- `.saturatingAdd(_:)`, `.saturatingSubtract(_:)`, `.saturatingMultiply(_:)`, `.saturatingDivide(_:)`
- `.toString()`, `.toBigEndianBytes()` (returns 16 bytes)

Verified operations on `UFix128`:
- `+`, `-`, `*`, `/`, `%`
- `==`, `!=`, `<`, `<=`, `>`, `>=`
- `.saturatingAdd(_:)`, `.saturatingSubtract(_:)`, `.saturatingMultiply(_:)`
- `.toString()`, `.toBigEndianBytes()` (16 bytes)
- **No `.saturatingDivide`** — `UFix128` only saturates Add/Sub/Mul (confirmed in `sema/type.go` lines 2155-2159 and reproduced on emulator: `the member is not defined on the type`).

`Mul` and `Div` always use `RoundTruncate` rounding (towards zero) per the underlying `onflow/fixed-point` library.

### Overflow / underflow behaviour (empirically verified)

| Op                                    | Result          |
|---------------------------------------|-----------------|
| `UFix128.max + 1.0`                   | reverts with `error: overflow`           |
| `UFix128(0.0) - 1.0`                  | reverts with `error: underflow`          |
| `Fix128.max + 1.0`                    | reverts with `error: overflow`           |
| `Fix128.min - 1.0`                    | reverts with `error: underflow`          |
| `UFix64(UFix128.max)`                 | reverts with `error: overflow`           |
| `UFix128(Fix128(-5.5))`               | reverts with `error: underflow`          |
| `UFix128.max.saturatingAdd(1.0)`      | returns `UFix128.max` (no revert)         |
| `UFix128(0.0).saturatingSubtract(1.0)`| returns `UFix128(0.0)`                    |
| `tiny * tiny` (product `< 1e-24`)     | silently truncates to `0.0` — does **not** revert |

That last row is the most dangerous: the result of `UFix128(1e-24) * UFix128(0.5)` is `0.0` exactly. Multiplication does not revert on underflow; it rounds toward zero.

## Decision Matrix

| Use case                                                  | Recommended type | Why                                                              |
|-----------------------------------------------------------|------------------|------------------------------------------------------------------|
| Vault balances for a standard FT today                    | `UFix64`         | All Flow FT/NFT interfaces are typed `UFix64`. Don't fight the ecosystem. |
| New token contract requiring high-precision per-share accounting | `UFix128` internal, `UFix64` external | Compute in `UFix128`, expose `UFix64` at the interface seam.     |
| Compound-interest accrual over many periods               | `UFix128`        | `UFix64` accumulates ~1e-8 rounding error per step.                |
| AMM constant-product invariant `x * y = k` near limits     | `UFix128`        | Two `UFix64` reserves of 1e9 multiply to 1e18 — over `UFix64.max` (~1.8e11). |
| Oracle price feeds with 18-decimal external values         | `UFix128`        | Native 24-decimal scale absorbs 18-decimal inputs lossless.        |
| Counting whole-token units (no fractional)                 | `UInt64`/`UInt256` | Integers are simpler, cheaper, and have higher range when no decimals are needed. |
| Counting wei-style integer amounts from EVM (>2^64)        | `UInt256`         | No fractional component; spans the full EVM `uint256`.             |
| Vesting schedule with per-second linear release            | `UFix128`         | Per-second slope at low rates (e.g. 1e-12 tokens/sec) underflows `UFix64`. |
| Signed PnL or delta tracking with both directions          | `Fix128`         | Need signed and need more than `Fix64`'s ~9.2e10 range or 8-decimal precision. |
| Cheap, low-stakes percentages and fees                     | `UFix64`         | 8 decimals are plenty for "0.30% swap fee."                        |

Rule of thumb: **stay in `UFix64` until you have an empirical reason not to**. Most reasons are (a) you do >10 sequential multiplications, (b) intermediate products overflow `UFix64`, or (c) you need to interoperate with 18-decimal external feeds.

## Precision Pitfall: 100x Compound Interest

```cadence
// ❌ UFix64: 8-decimal precision drifts under repeated multiplication
fun compound64(): UFix64 {
    var amount: UFix64 = 1000.0
    let rate: UFix64 = 1.05
    var i = 0
    while i < 100 {
        amount = amount * rate
        i = i + 1
    }
    return amount    // 131501.25783526   (off by ~1.05e-5 relative)
}

// ✅ UFix128: 24-decimal precision retains the trailing digits
fun compound128(): UFix128 {
    var amount: UFix128 = 1000.0
    let rate: UFix128 = 1.05
    var i = 0
    while i < 100 {
        amount = amount * rate
        i = i + 1
    }
    return amount    // 131501.257846303455025597531408
}
```

The `UFix64` answer is `131501.25783526`. The `UFix128` answer is `131501.257846303455...`. The error after 100 compounds is `~1.05e-5` absolute — small in this scenario, but it grows with the number of multiplications and the magnitude of fractional inputs. For tight invariants (AMM, lending health factors, share accounting), this drift is the difference between "books balance" and "audit finding."

## Migration Guidance

**When upgrading from `UFix64` to `UFix128` is worth it:**
- New contract, no compat constraints, and you do iterated multiplication, AMM math, oracle scaling, or per-second vesting.
- Internal calculations on an existing contract where the **external interface stays `UFix64`** (compute in `UFix128`, narrow back at the boundary). This keeps you wire-compatible with the FT/NFT standards while gaining precision internally.

**When NOT to upgrade:**
- Public token balance fields exposed via `FungibleToken.Vault.balance: UFix64`. You can't change the field type without breaking the standard interface and every downstream integrator. Don't.
- Code that only does additions and subtractions on bounded amounts. There is no precision win — `UFix64` arithmetic is exact for `add`/`sub` within range.
- Hot-path code where CU cost matters and the math is simple. `UFix128` ops use 128-bit limbs in the underlying library, so any added cost depends on the active execution-effort schedule — measure before assuming. On emulator (flow-cli v2.17.1), 1000 `UFix64` mul and 1000 `UFix128` mul both metered at `executionEffort = 0.00000089` — but the published mainnet schedule may differ.

**Migration pattern at the seam:**

```cadence
// External UFix64 amount comes in
let amountIn: UFix64 = 100.0

// Widen losslessly to UFix128
let amountIn128 = UFix128(amountIn)

// All internal math in UFix128
let priceWide: UFix128 = self.getPrice()      // 24-decimal oracle output
let amountOut128 = amountIn128 * priceWide / UFix128(1.0)

// Narrow back at the exit. Reverts on overflow; truncates fractional digits 9-24.
let amountOut: UFix64 = UFix64(amountOut128)
return amountOut
```

The narrowing `UFix64(...)` reverts on integer-part overflow and silently truncates fractional digits beyond 8. Always validate the narrowed value before using it as a refund or output — if `amountOut128 > UFix64.max`, the transaction aborts.

## Common Pitfalls

- **No implicit conversion.** `someUFix64 + someUFix128` is a compile error. You must cast: `UFix128(someUFix64) + someUFix128`.
- **Negative-to-unsigned reverts at runtime.** `UFix128(myFix128)` where `myFix128 < 0` reverts. Always check sign first if the source could be negative, or use a saturating workaround (`if myFix128 < Fix128(0.0) { return UFix128(0.0) }`).
- **Multiplication truncates toward zero.** Products `< 1e-24` become `UFix128(0.0)` silently. Do not rely on equality with zero to detect "no value moved" — also check the inputs.
- **`UFix128` has no `saturatingDivide`.** Only `Fix128` does. If you need saturating division on unsigned, branch on the divisor yourself.
- **Literal scale is strict.** `let x: UFix128 = 0.1234567890123456789012345` (25 digits) is a *static* error: "maximum scale 24." Trim or pre-multiply.
- **`.toBigEndianBytes()` returns 16 bytes for both 128-bit types,** but the encodings are not equivalent — the high bit of `Fix128` is sign. Don't interpret raw bytes across signed/unsigned without going through a typed conversion.
- **`fromBigEndianBytes` silently zero-pads short inputs.** Any input of length 0-16 decodes successfully (`fromBigEndianBytes([])` returns `UFix128(0.0)`, not `nil`). Only inputs longer than 16 bytes return `nil`. For oracle adapters, cross-VM ABI decoders, and any boundary that takes untrusted bytes, validate `bytes.length == 16` yourself before calling — otherwise a malformed payload decodes to zero and your downstream comparisons read "no value" instead of "rejected message".
- **Storage size doubles vs `UFix64`.** A `UFix128` field in storage uses 16 bytes plus CBOR overhead, vs 8 for `UFix64`. Bulk arrays of balances see real storage-fee impact.
- **Resource interface fields can't be re-typed.** If you've published a contract with `pub var price: UFix64`, you cannot change it to `UFix128` in an upgrade — Cadence rejects the type change. The exact validator message is `mismatching field <name>: incompatible types. expected <Old>, found <New>`. Plan precision at design time, not after deploy.
- **Comparison-against-zero precision.** `tiny < UFix128(1e-24)` and `tiny == UFix128(0.0)` can both be true for the same truncated product. Use a slightly larger epsilon, or check the inputs to your math.

## Sources

- FLIP 341 — Add 128-bit Fixed-point Types to Cadence (`onflow/flips/cadence/20250815-128-bit-fixed-point-types.md`, accepted 2025-08-29).
- Implementation: `onflow/cadence/interpreter/value_fix128.go`, `interpreter/value_ufix128.go`, `sema/type.go` (lines 2118-2160), `fixedpoint/check.go`, `fixedpoint/parse.go`.
- Underlying numeric library: `github.com/onflow/fixed-point` (provides `Fix128`/`UFix128` with `Sqrt`, `Pow`, `Exp`, `Ln`, `Sin`, `Cos` — but these are **not** currently exposed in Cadence sema; only `+`, `-`, `*`, `/`, `%` and saturating variants are reachable from Cadence code).
- Released in Cadence v1.7.0 (first tag containing `value_fix128.go`); available on emulator, testnet, and mainnet as of 2026-05.

- Cadence language reference: `https://cadence-lang.org/docs/language/values-and-types/fixed-point-nums-ints` — covers `Fix128`/`UFix128` ranges, `toString`/`toBigEndianBytes` examples, and the maximum-scale rule.
