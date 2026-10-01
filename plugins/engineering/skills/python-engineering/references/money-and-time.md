# Money and time

Read this for any code that computes money, rates, proration, rounding, timestamps, time zones or durations.

## Money

### Representation

- At the API boundary, money is a string (`"19.99"`) or an integer in minor units (`1999`) with a currency code. A JSON number is parsed as a float by most clients and many servers before your code sees it.
- In Pydantic, type amounts as `Decimal` (with `Field(max_digits=..., decimal_places=...)` where the contract fixes them) or `int` minor units. A `float` field is the defect even if the value is converted to `Decimal` a line later.
- In the database, `numeric` or `bigint` minor units. See the database-engineering skill.
- The number of minor units depends on the currency (ISO 4217: most have 2, some 0, some 3). Take the exponent from a currency table, not a literal `Decimal("0.01")` scattered through the code.

### Construction

```python
Decimal("19.99")      # exact
Decimal(1999) / 100   # exact
Decimal(19.99)        # Decimal('19.989999999999998436805981327779591083526611328125')
Decimal(str(19.99))   # "works", but the float already existed; fix the source instead
```

To catch float leaks in money code, trap them in a local context:

```python
from decimal import Decimal, FloatOperation, localcontext

with localcontext() as ctx:
    ctx.traps[FloatOperation] = True
    total = sum((line.amount for line in lines), Decimal(0))
```

With the trap set, `Decimal(3.14)` and `Decimal("3.5") < 3.7` raise; equality comparisons and explicit `Decimal.from_float` stay silent. Arithmetic between `Decimal` and `float` raises `TypeError` regardless, which is why float leaks usually enter through the constructor or a comparison.

### Rounding

- Default context: 28 significant digits, `ROUND_HALF_EVEN`. Nothing is rounded to cents until you `quantize`.
- Pick quantize points deliberately (per line, or per invoice total) and name the rounding mode from the business rule. Rounding per line and then summing gives a different total from summing and then rounding; which one is correct is a finance decision, and it must be the same everywhere.
- `getcontext()` returns a context object that is mutated in place. It is shared with later work on the same thread (threaded WSGI workers) and, when set before tasks were created, with every asyncio task that inherited it. `getcontext().rounding = ROUND_UP` in one request handler can change rounding for others. Use `localcontext()` or pass `rounding=` to `quantize`.
- Builtin `round()` rounds exact halves to even (`round(2.5) == 2`) and operates on the binary float (`round(2.675, 2) == 2.67`). Do not use it for money.

### Allocation

Splitting a total (installments, refunds across line items, fees across payees) by multiplying and rounding each part loses or invents cents. Allocate in minor units and hand out the remainder:

```python
def allocate(total_minor: int, weights: list[int]) -> list[int]:
    if total_minor < 0 or not weights or any(w < 0 for w in weights) or sum(weights) == 0:
        raise ValueError("invalid allocation input")
    weight_sum = sum(weights)
    shares = [total_minor * w // weight_sum for w in weights]
    remainder = total_minor - sum(shares)
    by_fraction = sorted(
        range(len(weights)),
        key=lambda i: (total_minor * weights[i]) % weight_sum,
        reverse=True,
    )
    for i in by_fraction[:remainder]:
        shares[i] += 1
    return shares
```

The parts always sum to the total. Tie-breaking order must be deterministic so a retry allocates the same way.

### Serialization

`json.dumps(Decimal("1.10"))` raises `TypeError`. The fix is to emit a string (`str(amount)`, or Pydantic's JSON serialization), not `float(amount)`, which reintroduces the error on the way out.

## Time

### Aware everywhere

```python
# Before
now = datetime.utcnow()                                             # naive, deprecated since 3.12
created = datetime.fromtimestamp(event["created"]).replace(tzinfo=timezone.utc)  # local time relabelled

# After
now = datetime.now(timezone.utc)
created = datetime.fromtimestamp(event["created"], tz=timezone.utc)
```

`fromtimestamp` without `tz` returns local wall time. On a host set to UTC the relabelled version happens to be right, which is why this bug survives until the code runs on a laptop, a VM with a local zone, or a container whose `TZ` someone set.

- Naive vs aware ordering (`<`, `>`) raises `TypeError`. Equality between them is always `False`, silently: dedupe keys and "already processed?" checks never match.
- `.astimezone()` and `.timestamp()` on a naive value assume local time.
- `datetime.fromisoformat` accepts a trailing `Z` only from 3.11. On older versions it raises, and code that "fixes" this by stripping the `Z` produces a naive value.
- `date.today()` is the local date. Business dates ("which day's report", "which invoice month") come from an aware timestamp converted to the business's zone: `ts.astimezone(ZoneInfo(tenant.tz)).date()`.
- Pydantic v2 `AwareDatetime` rejects naive input at the boundary.
- Epoch units: JavaScript and many APIs send milliseconds; Python's `fromtimestamp` takes seconds. Put the unit in the field name (`created_at_ms`) and convert once.

### Zones and DST

Store and compute in UTC; convert to a named zone (`zoneinfo.ZoneInfo("Europe/Athens")`) for display and for rules defined in local time. Never use fixed offsets (`timezone(timedelta(hours=2))`) for places that observe DST.

Arithmetic between datetimes sharing the same `tzinfo` is wall-clock arithmetic and ignores DST transitions:

```python
athens = ZoneInfo("Europe/Athens")
start = datetime(2026, 3, 28, 12, tzinfo=athens)
end = start + timedelta(days=1)    # 2026-03-29 12:00 local: DST began that night
end - start                        # timedelta(days=1), though 23 hours elapsed
```

For elapsed time, billing by the hour, or token expiry, convert to UTC first. For "same local time tomorrow" (a daily 09:00 job), the wall-clock behavior is what you want. Local times that do not exist (spring forward) or occur twice (fall back, disambiguated by `fold`) need an explicit rule for recurring schedules.

### Clocks

- `time.monotonic()` for timeouts, rate limits and durations; `time.perf_counter()` for benchmarks. `time.time()` and `datetime.now()` jump when NTP corrects the clock.
- Wall-clock timestamps from different machines are not ordered; do not use them to decide which of two writes happened first.
- Make "now" injectable (a parameter or a clock object) in code with time-based rules, so tests can cover month ends, DST transitions and expiry boundaries without sleeping or patching `datetime`.
