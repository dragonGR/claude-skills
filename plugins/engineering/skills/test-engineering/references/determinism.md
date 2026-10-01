# Time, timers, randomness and order

Read this when a test touches the current time, expiry, time zones, timers, retries with backoff, random data or test ordering, or when a test fails only sometimes.

## Prefer an injected clock

A fake global clock is a patch over code that reads the time directly. Code that takes a clock as a dependency is simpler to test and can be tested at exact boundaries without any library magic.

```python
from datetime import datetime, timedelta, timezone
from typing import Protocol


class Clock(Protocol):
    def now(self) -> datetime: ...


class FixedClock:
    def __init__(self, at: datetime):
        self._at = at

    def now(self) -> datetime:
        return self._at

    def advance(self, delta: timedelta) -> None:
        self._at += delta


def test_reset_token_expires_exactly_at_ttl():
    clock = FixedClock(datetime(2025, 1, 31, 23, 59, 59, tzinfo=timezone.utc))
    tokens = ResetTokens(clock=clock, ttl=RESET_TOKEN_TTL)
    token = tokens.issue(user_id=USER_ID)

    clock.advance(RESET_TOKEN_TTL - timedelta(microseconds=1))
    assert tokens.is_valid(token)
    clock.advance(timedelta(microseconds=1))
    assert not tokens.is_valid(token)
```

The start instant here is chosen to cross a day and month boundary during the test. Pick start times that stress the code: end of month, 29 February, the hour a DST change happens in the zones you serve, just before midnight UTC.

## Fake clocks per tool

**Vitest**

```ts
import { afterEach, beforeEach, expect, it, vi } from "vitest";

const FIXED_NOW = new Date("2025-03-30T00:30:00Z");

beforeEach(() => {
  vi.useFakeTimers();
  vi.setSystemTime(FIXED_NOW);
});

afterEach(() => {
  vi.useRealTimers();
});

it("releases the hold exactly when the TTL elapses", async () => {
  const hold = holds.create(ITEM_ID);
  await vi.advanceTimersByTimeAsync(HOLD_TTL_MS - 1);
  expect(holds.isActive(hold.id)).toBe(true);
  await vi.advanceTimersByTimeAsync(1);
  expect(holds.isActive(hold.id)).toBe(false);
});
```

`vi.useFakeTimers()` fakes `Date` and the timer functions but not `process.nextTick` or `queueMicrotask` unless listed in `toFake`. `vi.setSystemTime` moves the clock without firing timers; `advanceTimersByTimeAsync` fires them and lets promise callbacks run between them. Jest's equivalents are `jest.useFakeTimers`, `jest.setSystemTime` and `jest.advanceTimersByTimeAsync`; Jest fakes `queueMicrotask` by default.

Fake timers apply to everything in the process, including a database pool's idle timeout, an HTTP client's timeout and retry backoff in libraries. A test that awaits real I/O with timers faked can hang or time out for reasons unrelated to the code under test. Keep real I/O out of fake-timer tests, or fake only what the test needs with `toFake`.

**Python, time-machine**

```python
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo

import time_machine

DST_START_ATHENS = datetime(2025, 3, 30, 2, 30, tzinfo=ZoneInfo("Europe/Athens"))


def test_session_ttl_counts_real_elapsed_time_across_dst():
    with time_machine.travel(DST_START_ATHENS, tick=False) as traveller:
        session = sessions.create(user_id=USER_ID)
        traveller.shift(SESSION_TTL - timedelta(seconds=1))
        assert sessions.is_valid(session)
        traveller.shift(timedelta(seconds=1))
        assert not sessions.is_valid(session)
```

`tick=False` freezes time at the destination until the test moves it with `shift` or `move_to`. Without it the clock keeps running from the destination and boundary assertions become racy.

**Rust, tokio**

```rust
use std::time::Duration;

#[tokio::test(start_paused = true)]
async fn retries_back_off_before_giving_up() {
    let fake = FakeUpstream::always_unavailable();
    let client = Client::new(fake.clone(), RetryPolicy::default());

    let result = client.fetch().await;

    assert!(matches!(result, Err(FetchError::Unavailable)));
    assert_eq!(fake.request_count(), RetryPolicy::default().max_attempts);
    assert_eq!(fake.gaps(), RetryPolicy::default().expected_delays());
}
```

`start_paused` needs tokio's `test-util` feature and the current-thread runtime (the default for `#[tokio::test]`). With time paused, the runtime jumps the clock to the next timer whenever it has no other work, so a backoff of minutes runs instantly. That same rule means a task waiting on a real socket counts as idle, so the clock can jump past your timeouts while I/O is in flight; keep the fake upstream in memory (a trait implementation, as here) for paused-time tests. Only tokio's `Instant` is paused; `std::time::Instant` and `SystemTime` keep running. Use `tokio::time::advance` to move time by an exact amount.

**Playwright**

```ts
await page.clock.install({ time: new Date("2025-01-31T23:50:00Z") });
await page.goto(sessionPageUrl);
await page.clock.fastForward("15:00");
await expect(page.getByRole("dialog", { name: "Session expiring" })).toBeVisible();
```

`install` must run before the page's scripts read the time. `setFixedTime` pins `Date.now()` without controlling timers; `runFor` advances and fires timers in order; `fastForward` jumps and fires due timers once.

## The database has its own clock

Faking the application's clock does nothing to SQL. `WHERE expires_at > now()` uses the database server's time, and in PostgreSQL `now()` is the start time of the current transaction. Inside a rolled-back test transaction, every `now()` call returns the same instant, so rows inserted in sequence get identical `created_at` values and ordering by that column is arbitrary.

When SQL logic depends on the current time, pass the time in as a parameter from the injected clock (`WHERE expires_at > $1`). When that is not possible, set the stored timestamps explicitly in fixtures relative to the database's time rather than the fake application time.

## Time zones

A suite that runs only in UTC hides every bug that converts through local time. Run at least one CI job with a zone far from UTC and with DST (for Node, set `TZ` in the environment before the process starts; Python reads `TZ` at startup too). Assert on instants or on explicitly zoned values, never on strings produced with the machine's local zone.

## Randomness and order

- Random test data: seed from a known value and print it, so a failure can be replayed. pytest-randomly resets `random.seed()` before each test and prints the seed; replay with `--randomly-seed=<n>` or `--randomly-seed=last`.
- Order: run the suite shuffled in CI so hidden dependencies surface: pytest-randomly (on by default once installed), Jest `--randomize` with `--seed`, Vitest `--sequence.shuffle` with `--sequence.seed`.
- Unique values: factories generate emails, slugs and external ids from a sequence or UUID so parallel workers sharing a database do not collide.
- Never assert on the order of UUIDv4s, set iteration, or rows returned without `ORDER BY`.

## Diagnosing a flaky test

1. Reproduce by repetition: Playwright `--repeat-each`, a shell loop around the single test, or the shuffled suite with the failing seed.
2. Check whether it fails alone or only with others. Alone points to time, randomness, a real race or an async wait; only with others points to shared state or order.
3. Look for the usual causes in this order: sleeps and non-retrying checks of async state; wall-clock time; shared rows, ids or module state; unawaited promises; real races in the code under test.
4. If the flake is on a money or auth path, assume a real race until proven otherwise and write the deterministic test from `concurrency-and-idempotency.md`.
5. Fix the cause, then run the repetition again to show it is gone. A passing single rerun is not evidence.
