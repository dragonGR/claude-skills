# Numbers, money and time in JavaScript

Read this for code that handles 64-bit ids, token amounts, `bigint`, money, rounding, allocation, timestamps, calendar dates or time zones.

JavaScript has one number type for everything below `bigint`: an IEEE 754 double. Integers are exact only up to `Number.MAX_SAFE_INTEGER` (2^53 - 1), decimals like `0.1` are never exact, and nothing warns when either limit is crossed.

## Large integers

### Where precision is lost

| Step | What happens |
|---|---|
| `JSON.parse('{"id":9007199254740993}')` | `id` becomes `9007199254740992`, no error |
| `Number("9007199254740993")`, `parseInt`, unary `+` | same rounding |
| node-postgres `int8` (OID 20) | returned as a string by default, which is safe; `types.setTypeParser(20, parseInt)` reintroduces the rounding |
| node-postgres `numeric` | returned as a string; converting with `Number` or `parseFloat` loses digits |
| `Number(big)` on a `bigint` | silent rounding |
| `JSON.stringify({ amount: 1n })` | throws `TypeError` |
| `1n + 1` | throws `TypeError` (no implicit mixing) |
| `BigInt(1.5)` | throws `RangeError`; `BigInt("12abc")` throws `SyntaxError` |

Values at risk: Snowflake-style ids (Twitter, Discord and many internal id generators), Postgres `bigint`/`bigserial`, token amounts in base units (wei has 18 decimals, so one token is already 10^18), nanosecond timestamps, and money totals in minor units for large ledgers.

### Rules

- Identifiers you never do arithmetic on are strings from the database to the client. Type them as branded strings so they cannot be mixed up.
- Amounts you compute with are `bigint` inside the process and decimal strings on the wire.
- A producer that emits a bare JSON number above 2^53 has already lost the value for every JavaScript consumer. Fix the producer to send a string.

### Parsing and serializing

```ts
const DECIMAL_INTEGER = /^(0|[1-9][0-9]*)$/;

export const BaseUnits = z
  .string()
  .max(limits.amountMaxDigits)
  .regex(DECIMAL_INTEGER)
  .transform((s) => BigInt(s));

const Payout = z.strictObject({
  recipient: z.string().regex(EVM_ADDRESS),
  amountWei: BaseUnits,
});
```

For output, convert at the DTO boundary rather than patching `BigInt.prototype.toJSON`. The global patch changes behavior for every library in the process and hides the places where nobody decided on a wire format.

```ts
function toPayoutDto(p: PayoutRow): PayoutDto {
  return { id: p.id, amountWei: p.amountWei.toString(), status: p.status };
}
```

If you need a generic serializer for logs, use a replacer: `JSON.stringify(value, (_k, v) => (typeof v === "bigint" ? v.toString() : v))`.

### Token amounts

- Convert between human units and base units with the library's helpers (`parseUnits` / `formatUnits` in viem and ethers), passing the token's decimals as read from the token contract or your token registry, never an assumed 18. USDC and USDT use 6 on Ethereum mainnet, and the same symbol on another chain can use a different number.
- Never divide by `1e18` or `10 ** decimals` as a `number`. `Number(balance) / 1e18` loses precision twice.
- `bigint` division truncates toward zero: `7n / 2n === 3n` and `-7n / 2n === -3n`. When a fee or share must round a particular way, write it explicitly:

```ts
function mulDivUp(value: bigint, numerator: bigint, denominator: bigint): bigint {
  if (value < 0n || numerator < 0n || denominator <= 0n) throw new RangeError("mulDivUp expects non-negative inputs");
  return (value * numerator + denominator - 1n) / denominator;
}
```

Fees the protocol collects round up; amounts paid out round down. Rounding the wrong way on either side is a slow leak that someone will eventually loop.

## Money

### Float failures

```ts
0.1 + 0.2;                 // 0.30000000000000004
Math.round(1.005 * 100);   // 100, because 1.005 * 100 is 100.49999999999999
(1.005).toFixed(2);        // "1.00"
```

Each of these has shipped as a one-cent discrepancy that fails reconciliation, a total that disagrees with the payment provider, or a refund of the wrong amount.

### Representation

- Integer minor units when the currency has a fixed exponent and the code only adds, subtracts and multiplies by integers. Use `number` only if the largest total you can ever hold is below 2^53 minor units; otherwise `bigint`.
- A decimal library (`decimal.js`, `big.js`, or what the project already uses) when rates, percentages, FX or proration are involved. Construct from strings, not from `number` literals that are already inexact.
- The currency travels with the amount. A bare `amount` field with the currency implied by the account is how a JPY amount gets multiplied by 100.
- Minor-unit exponents differ by currency (JPY has none, several currencies have three). Take the exponent from your currency table or payment provider, not from a literal `100`.
- Prisma maps `Decimal` columns to a `Decimal` object and `BigInt` columns to `bigint`; `Number(decimal)` at the edge of the ORM undoes both.

### Rounding and allocation

Round at defined points with the mode the business rule names, and make splits sum to the total.

```ts
export function allocate(totalMinor: bigint, weights: readonly bigint[]): bigint[] {
  if (totalMinor < 0n || weights.length === 0 || weights.some((w) => w < 0n)) {
    throw new RangeError("allocate expects a non-negative total and weights");
  }
  const weightSum = weights.reduce((a, b) => a + b, 0n);
  if (weightSum === 0n) throw new RangeError("weights sum to zero");

  const shares = weights.map((w) => (totalMinor * w) / weightSum);
  // Each share lost less than one unit to truncation, so the leftover is smaller than weights.length.
  const leftover = Number(totalMinor - shares.reduce((a, b) => a + b, 0n));
  return shares.map((s, i) => (i < leftover ? s + 1n : s));
}
```

This gives leftover units to the first parts. If the rule says largest remainder or a named party absorbs the difference, implement that rule instead; the invariant to test is that the parts sum to the total.

### Display

`Intl.NumberFormat` accepts `bigint` and decimal strings, and formats strings exactly instead of converting them to `number` first. Format with an explicit `currency` and locale; never build currency strings by concatenation.

## Dates and time

### Parsing

| Input | Interpreted as |
|---|---|
| `"2024-03-10"` (date-only ISO) | UTC midnight |
| `"2024-03-10T00:00"` or `"2024-03-10T00:00:00"` (no offset) | local time of the process |
| `"2024-03-10T00:00:00Z"` or `+01:00` | that exact instant |
| `"03/10/2024"`, `"10 Mar 2024"`, other non-ISO | implementation-specific |

Consequences seen in production:

- `new Date("2024-03-10").getDate()` returns 9 on a server or browser west of UTC. Birthdays, due dates and report ranges shift a day.
- A string without an offset means different instants on a laptop, in CI and in a container, so tests pass locally and fail in CI or the other way round.
- `new Date("garbage")` does not throw. It yields an Invalid Date whose `getTime()` is `NaN`, and `toISOString()` throws a `RangeError` later, far from the input.
- `new Date(2024, 1, 30)` is March 1st: months are zero-based and out-of-range values roll over silently.

### Representation

- Instants (created at, expires at, paid at): epoch milliseconds or ISO 8601 with `Z` or an offset. Validate with `z.iso.datetime()` (accepts only `Z` by default; `offset: true` also allows offsets). Store in Postgres as `timestamptz`.
- Calendar dates (birthday, invoice date, business day): `YYYY-MM-DD` strings (`z.iso.date()`), never a `Date`, which is an instant and drags a time zone into a value that has none.
- Wall-clock times in a place (store opens at 09:00 in Athens): the local time plus an IANA zone name. Convert to an instant only when needed.
- node-postgres converts `timestamp without time zone` and `date` columns using the process's local time zone (`process.env.TZ`). A server and a worker with different `TZ` values read different instants from the same row. Prefer `timestamptz`, and for `date` columns register a parser that keeps the string.

### Arithmetic and display

- Adding `24 * 60 * 60 * 1000` ms is not "the next day" across a DST change. Calendar arithmetic in a zone needs a library that understands zones, or UTC-only calendar math where the business rule is defined in UTC. `Temporal.ZonedDateTime` does zone-aware arithmetic natively in Node 26, Chrome 144+, Firefox 139+, Deno 2.7+ and Bun 1.4+, but not yet in Safari or Node 24, and some Linux distribution builds of Node 26 compile it out. Check `typeof Temporal` on the runtime you deploy to and load a polyfill where it is missing.
- Format for users with `Intl.DateTimeFormat` and an explicit `timeZone`. Formatting without one uses the process's zone on the server and the user's zone in the browser, which is also a React hydration mismatch (frontend-engineering).
- Durations and timeouts use `performance.now()` (monotonic). `Date.now()` jumps when the clock is corrected.
- Comparing ISO strings lexicographically works only when both are UTC with the same precision and format. Compare parsed epoch values otherwise.
- Expiry checks compare against the server's clock, never a client-supplied "now".
