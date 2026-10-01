# Concurrency and idempotency tests

Read this when writing tests for races, double submits, idempotency keys or lost responses on paths that move money or change durable state. Database setup helpers (`startTestDatabase`, `truncateAll`, `committed_wallet`) are in `real-dependencies.md`.

All of these run against the production database engine with committed data and one connection per concurrent request. Inside a rolled-back test transaction, or on one shared connection, the requests never contend and the test proves nothing.

## Burst test: many conflicting requests at once

Cheap to write and good at catching the common races (check-then-act, read-modify-write). A pass is weak evidence, because the scheduler may not have produced the bad interleaving on that run; a failure is a real bug. Pair it with the deterministic test below for anything important.

TypeScript (Vitest, node-postgres):

```ts
import { Pool } from "pg";
import { afterAll, beforeAll, beforeEach, expect, it } from "vitest";
import { InsufficientFunds, withdraw } from "../src/wallets";
import { startTestDatabase, truncateAll, type TestDatabase } from "./support/db";

const CONCURRENT_REQUESTS = 10;
const OPENING_BALANCE_CENTS = 1_000;
const WITHDRAWAL_CENTS = 300;
const OWNER_USER_ID = 1;

let db: TestDatabase;
let pool: Pool;

beforeAll(async () => {
  db = await startTestDatabase();
  // One connection per request; a smaller pool queues the calls and hides the race.
  pool = new Pool({ connectionString: db.url, max: CONCURRENT_REQUESTS });
});

afterAll(async () => {
  await pool.end();
  await db.stop();
});

beforeEach(async () => {
  await truncateAll(pool);
});

it("never overdraws a wallet under concurrent withdrawals", async () => {
  const inserted = await pool.query(
    "INSERT INTO wallets (user_id, balance_cents) VALUES ($1, $2) RETURNING id",
    [OWNER_USER_ID, OPENING_BALANCE_CENTS],
  );
  const walletId = inserted.rows[0].id;
  const expectedSuccesses = Math.floor(OPENING_BALANCE_CENTS / WITHDRAWAL_CENTS);

  const results = await Promise.allSettled(
    Array.from({ length: CONCURRENT_REQUESTS }, () => withdraw(pool, walletId, WITHDRAWAL_CENTS)),
  );

  const rejected = results.filter((r) => r.status === "rejected");
  expect(results.length - rejected.length).toBe(expectedSuccesses);
  for (const r of rejected) expect(r.reason).toBeInstanceOf(InsufficientFunds);

  const after = await pool.query("SELECT balance_cents FROM wallets WHERE id = $1", [walletId]);
  // int8 arrives as a string from node-postgres
  expect(BigInt(after.rows[0].balance_cents)).toBe(
    BigInt(OPENING_BALANCE_CENTS - expectedSuccesses * WITHDRAWAL_CENTS),
  );
  const entries = await pool.query(
    "SELECT count(*)::int AS n FROM ledger_entries WHERE wallet_id = $1",
    [walletId],
  );
  expect(entries.rows[0].n).toBe(expectedSuccesses);
});
```

What the assertions pin down: the exact number of successes (not "at least one"), that every failure is the expected domain error (a deadlock or serialization error fails the test unless the code retries it), the final balance, and that the ledger agrees with the balance.

Python (pytest, psycopg 3). A `threading.Barrier` releases all workers together, after each has its own connection open, so connection setup does not spread the requests out:

```python
import threading
from concurrent.futures import ThreadPoolExecutor

import psycopg

from app.wallets import InsufficientFunds, withdraw

CONCURRENT_REQUESTS = 10
OPENING_BALANCE_CENTS = 1_000
WITHDRAWAL_CENTS = 300
BARRIER_TIMEOUT_S = 10


def test_concurrent_withdrawals_never_overdraw(pg_url, committed_wallet):
    wallet_id = committed_wallet(balance_cents=OPENING_BALANCE_CENTS)
    barrier = threading.Barrier(CONCURRENT_REQUESTS, timeout=BARRIER_TIMEOUT_S)

    def attempt() -> str:
        with psycopg.connect(pg_url) as conn:
            barrier.wait()
            try:
                withdraw(conn, wallet_id, WITHDRAWAL_CENTS)
            except InsufficientFunds:
                return "insufficient"
            return "ok"

    with ThreadPoolExecutor(max_workers=CONCURRENT_REQUESTS) as executor:
        futures = [executor.submit(attempt) for _ in range(CONCURRENT_REQUESTS)]
        outcomes = [f.result() for f in futures]

    expected_ok = OPENING_BALANCE_CENTS // WITHDRAWAL_CENTS
    assert outcomes.count("ok") == expected_ok
    assert outcomes.count("insufficient") == CONCURRENT_REQUESTS - expected_ok
    with psycopg.connect(pg_url) as conn:
        (balance,) = conn.execute(
            "SELECT balance_cents FROM wallets WHERE id = %s", (wallet_id,)
        ).fetchone()
    assert balance == OPENING_BALANCE_CENTS - expected_ok * WITHDRAWAL_CENTS
```

Any unexpected exception in a worker re-raises from `f.result()` and fails the test.

For an HTTP-level version, run the same burst against the running app with a real HTTP client so middleware, transaction handling and connection pooling are included.

## Deterministic interleaving with a lock holder

Forces the exact interleaving that breaks naive code, so the test fails every time on the bug instead of occasionally. The idea: a second connection takes the row lock and makes an uncommitted change, the operation under test starts and must block on that lock, then the holder commits and the operation must see the committed state.

```python
import time
from concurrent.futures import ThreadPoolExecutor

import psycopg
import pytest

from app.wallets import InsufficientFunds, withdraw

OPENING_BALANCE_CENTS = 1_000
WITHDRAWAL_CENTS = 300
LOCK_WAIT_TIMEOUT_S = 10
POLL_INTERVAL_S = 0.02


def wait_until_blocked(observer: psycopg.Connection, pid: int) -> None:
    deadline = time.monotonic() + LOCK_WAIT_TIMEOUT_S
    while time.monotonic() < deadline:
        (blocked,) = observer.execute(
            "SELECT cardinality(pg_blocking_pids(%s)) > 0", (pid,)
        ).fetchone()
        if blocked:
            return
        time.sleep(POLL_INTERVAL_S)
    raise AssertionError(f"backend {pid} never waited on the row lock")


def test_withdraw_rechecks_balance_after_concurrent_debit(pg_url, committed_wallet):
    wallet_id = committed_wallet(balance_cents=OPENING_BALANCE_CENTS)
    with (
        psycopg.connect(pg_url) as holder,
        psycopg.connect(pg_url) as worker,
        psycopg.connect(pg_url, autocommit=True) as observer,
    ):
        holder.execute(
            "UPDATE wallets SET balance_cents = 0 WHERE id = %s", (wallet_id,)
        )
        with ThreadPoolExecutor(max_workers=1) as executor:
            pending = executor.submit(withdraw, worker, wallet_id, WITHDRAWAL_CENTS)
            wait_until_blocked(observer, worker.info.backend_pid)
            holder.commit()
            with pytest.raises(InsufficientFunds):
                pending.result(timeout=LOCK_WAIT_TIMEOUT_S)
```

How it catches the bug: a correct `withdraw` (conditional `UPDATE ... WHERE balance_cents >= $amount`, or `SELECT ... FOR UPDATE` then check) blocks on the holder's lock, then sees balance 0 and rejects. A read-modify-write version reads 1,000 without blocking (plain reads do not wait under MVCC), blocks only at its `UPDATE`, then overwrites the holder's committed 0 with 700. The test fails on every run.

The same shape tests check-then-insert: the holder inserts the row with the unique key and keeps its transaction open; the operation must block on the unique index, and after the holder commits it must return the "already exists" answer rather than a 500 or a second row.

When the race is inside application code rather than at a database lock, add a seam: an injectable hook the code awaits between its read and its write. The test holds the first request at the hook, runs the second to completion, then releases the first.

## In-process async races with fast-check

`fc.scheduler()` lets fast-check choose the order in which scheduled promises resolve and shrinks to the smallest failing order. Use it for races inside one process: caches, single-flight token refresh, client-side request ordering, in-memory queues. It does not replace database tests.

```ts
import fc from "fast-check";
import { expect, it } from "vitest";
import { createTokenCache } from "../src/auth/token-cache";

const MIN_CALLERS = 2;
const MAX_CALLERS = 5;
const NEVER_EXPIRES_MS = Number.MAX_SAFE_INTEGER;

it("concurrent callers share a single token refresh", async () => {
  await fc.assert(
    fc.asyncProperty(
      fc.scheduler(),
      fc.integer({ min: MIN_CALLERS, max: MAX_CALLERS }),
      async (s, callers) => {
        let refreshes = 0;
        const refresh = s.scheduleFunction(async () => {
          refreshes += 1;
          return { token: `token-${refreshes}`, expiresAtMs: NEVER_EXPIRES_MS };
        });
        const cache = createTokenCache({ refresh });

        const tokens = Promise.all(Array.from({ length: callers }, () => cache.getToken()));
        await s.waitIdle();

        expect(new Set(await tokens).size).toBe(1);
        expect(refreshes).toBe(1);
      },
    ),
  );
});
```

`refresh` stands in for the call to the identity provider, so counting it is counting an external side effect, which is the outcome here.

## Idempotency tests

An idempotency key is only as good as the tests of its edge cases. The sequential replay is the case everyone writes and the one least likely to break. Cover all of these for any endpoint that takes a key:

| Case | Expected |
| --- | --- |
| Same key, same body, sequential | Stored response returned; one side effect |
| Same key, same body, concurrent | One side effect; others get the stored response or an "in progress" conflict |
| Same key, different body | Rejected; original operation untouched |
| Same key, different user | Keys are scoped per caller; no collision, no leak of the other user's response |
| Provider applied the charge, response lost, client retries | One charge at the provider |
| No key on an endpoint that requires one | Rejected before any side effect |

The provider fake must implement the provider's own idempotency (dedupe by the key header), or the lost-response test cannot tell a correct retry from a double charge. Parse request bodies with the production schema so the fake rejects what the real provider would. The example uses the MSW 3 API; `real-dependencies.md` lists the MSW 2 names.

```ts
import { http, HttpResponse } from "msw/http";
import { setupServer } from "msw/node";
import supertest from "supertest";
import { afterAll, afterEach, beforeAll, beforeEach, expect, it } from "vitest";
import { app } from "../src/app";
import { ChargeRequest } from "../src/provider/schema";
import { testConfig } from "./support/config";
import { tokenFor, users } from "./support/auth";

const ORDER_AMOUNT_CENTS = 2_500;
// supertest calls the app over loopback, and MSW intercepts those requests too.
const LOOPBACK_HOSTS = new Set(["127.0.0.1", "[::1]", "localhost"]);

const providerCharges = new Map<string, number>();
let dropNextProviderResponse = false;

const server = setupServer(
  http.post(`${testConfig.providerBaseUrl}/charges`, async ({ request }) => {
    const key = request.headers.get("Idempotency-Key");
    if (key === null) {
      return HttpResponse.json({ error: { code: "missing_idempotency_key" } }, { status: 400 });
    }
    const body = ChargeRequest.parse(await request.json());
    const amount = providerCharges.get(key) ?? body.amount;
    providerCharges.set(key, amount);
    if (dropNextProviderResponse) {
      dropNextProviderResponse = false;
      return HttpResponse.error();
    }
    return HttpResponse.json({ id: `ch_${key}`, amount, status: "succeeded" });
  }),
);

beforeAll(() =>
  server.listen({
    onUnhandledFrame({ frame, defaults }) {
      if (frame.protocol === "http" && LOOPBACK_HOSTS.has(new URL(frame.data.request.url).hostname)) {
        return;
      }
      defaults.error();
    },
  }),
);
beforeEach(() => {
  providerCharges.clear();
  dropNextProviderResponse = false;
});
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

async function pay(user: keyof typeof users, key: string, orderId: string, amountCents = ORDER_AMOUNT_CENTS) {
  return supertest(app)
    .post("/payments")
    .set("Authorization", `Bearer ${await tokenFor(user)}`)
    .set("Idempotency-Key", key)
    .send({ orderId, amountCents });
}

it("replays the stored result for a repeated key", async () => {
  const first = await pay("alice", "key-1", "order-a");
  const second = await pay("alice", "key-1", "order-a");
  expect(first.status).toBe(201);
  expect(second.status).toBe(201);
  expect(second.body).toEqual(first.body);
  expect(providerCharges.size).toBe(1);
});

it("charges once when the same key arrives concurrently", async () => {
  const responses = await Promise.all([1, 2, 3].map(() => pay("alice", "key-2", "order-b")));
  for (const r of responses) expect([201, 409]).toContain(r.status);
  expect(providerCharges.size).toBe(1);
});

it("rejects a reused key with a different body", async () => {
  await pay("alice", "key-3", "order-c");
  const reused = await pay("alice", "key-3", "order-c", ORDER_AMOUNT_CENTS + 1);
  expect(reused.status).toBe(422);
  expect(reused.body.error.code).toBe("idempotency_key_reused");
  expect(providerCharges.size).toBe(1);
});

it("scopes keys per user", async () => {
  const alice = await pay("alice", "shared-key", "order-d");
  const bob = await pay("bob", "shared-key", "order-e");
  expect(bob.status).toBe(201);
  expect(bob.body.orderId).toBe("order-e");
  expect(bob.body.paymentId).not.toBe(alice.body.paymentId);
});

it("does not double charge when the provider response is lost", async () => {
  dropNextProviderResponse = true;
  const first = await pay("alice", "key-4", "order-f");
  expect(first.status).not.toBe(201);
  const retry = await pay("alice", "key-4", "order-f");
  expect(retry.status).toBe(201);
  expect(providerCharges.size).toBe(1);
});
```

The status codes and error code are this example's contract; use your API's. The "concurrent" case accepts either 201 (stored result) or 409 (still in progress) because both are correct designs; what must never vary is the single provider charge. The lost-response case only passes if the service reuses one provider idempotency key per operation, derived from stored state, rather than generating a fresh key per attempt.
