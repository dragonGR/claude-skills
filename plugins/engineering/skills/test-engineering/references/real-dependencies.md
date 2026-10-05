# Real databases, isolation and HTTP fakes

Read this when setting up integration tests against a database, choosing how tests are isolated, or faking a third-party HTTP API.

## Run the production engine

Use the same engine and major version as production, with the schema built by the project's migration runner. Take the image from configuration so it tracks production instead of a literal buried in a helper.

Node (Testcontainers, node-postgres):

```ts
import { PostgreSqlContainer } from "@testcontainers/postgresql";
import type { Pool } from "pg";
import { runMigrations } from "../../src/db/migrate";

export type TestDatabase = { url: string; stop: () => Promise<void> };

// Your migration runner's bookkeeping table; truncating it would make the next run re-apply migrations.
const MIGRATIONS_TABLE = "schema_migrations";

function requiredEnv(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} must be set to run integration tests`);
  return value;
}

export async function startTestDatabase(): Promise<TestDatabase> {
  const container = await new PostgreSqlContainer(requiredEnv("TEST_POSTGRES_IMAGE")).start();
  const url = container.getConnectionUri();
  await runMigrations(url);
  return {
    url,
    stop: async () => {
      await container.stop();
    },
  };
}

export async function truncateAll(pool: Pool): Promise<void> {
  const { rows } = await pool.query<{ name: string }>(
    `SELECT format('%I.%I', schemaname, tablename) AS name
       FROM pg_tables
      WHERE schemaname = 'public' AND tablename <> $1`,
    [MIGRATIONS_TABLE],
  );
  if (rows.length === 0) return;
  await pool.query(`TRUNCATE ${rows.map((r) => r.name).join(", ")} RESTART IDENTITY CASCADE`);
}
```

Python (testcontainers, psycopg 3):

```python
import os

import psycopg
import pytest
from psycopg import sql
from testcontainers.community.postgres import PostgresContainer  # testcontainers 4.15+; older releases: testcontainers.postgres

from app.db.migrate import run_migrations

MIGRATIONS_TABLE = "schema_migrations"
OWNER_USER_ID = 1


@pytest.fixture(scope="session")
def pg_url():
    with PostgresContainer(os.environ["TEST_POSTGRES_IMAGE"], driver=None) as pg:
        url = pg.get_connection_url()
        run_migrations(url)
        yield url


@pytest.fixture
def clean_db(pg_url):
    with psycopg.connect(pg_url, autocommit=True) as conn:
        tables = conn.execute(
            "SELECT schemaname, tablename FROM pg_tables "
            "WHERE schemaname = 'public' AND tablename <> %s",
            (MIGRATIONS_TABLE,),
        ).fetchall()
        if tables:
            conn.execute(
                sql.SQL("TRUNCATE {} RESTART IDENTITY CASCADE").format(
                    sql.SQL(", ").join(sql.Identifier(s, t) for s, t in tables)
                )
            )


@pytest.fixture
def committed_wallet(pg_url, clean_db):
    def create(balance_cents: int) -> int:
        with psycopg.connect(pg_url) as conn:
            (wallet_id,) = conn.execute(
                "INSERT INTO wallets (user_id, balance_cents) VALUES (%s, %s) RETURNING id",
                (OWNER_USER_ID, balance_cents),
            ).fetchone()
        return wallet_id

    return create
```

`clean_db` truncates before each test rather than after, so a test that crashed halfway cannot poison the next one. Leaving a `with psycopg.connect(...)` block commits the transaction and closes the connection.

## What SQLite and in-memory fakes hide

| Behavior | PostgreSQL | SQLite (default settings) |
| --- | --- | --- |
| Column types | Enforced | Non-`STRICT` tables store `'abc'` in an `INTEGER` column |
| Foreign keys | Enforced | Ignored unless each connection runs `PRAGMA foreign_keys = ON` |
| `LIKE` | Case-sensitive | Case-insensitive for ASCII only |
| `SELECT ... FOR UPDATE` | Row locks | Not supported |
| Concurrent writers | Row-level locking, real races possible | One writer at a time, races disappear |
| Big integers from the driver | node-postgres returns `int8` as a string | Drivers differ; often a JavaScript number |

An in-memory repository fake has none of the constraints, so uniqueness races, foreign key violations and check constraints never fire. Keep in-memory fakes for unit tests of logic above the data layer, and test the data layer itself against the real engine.

Build the test schema with the migrations, not with an ORM's `create_all()` or `sync()`. Constraints, partial indexes, triggers and defaults that exist only in migrations vanish from a model-generated schema, and so do the bugs they would have caught.

## Choosing isolation

**Rollback per test** (wrap each test in a transaction, roll back at the end) is fast and fine for most tests. It hides everything that happens at or after commit:

- deferred constraints are checked at commit, which never comes;
- `on_commit` hooks, outbox relays and anything triggered after commit never run;
- a second connection (a background worker, a thread, the code under test opening its own connection) cannot see the test's uncommitted rows;
- two "concurrent" requests share one connection and one transaction, so they never contend for locks;
- serialization failures and deadlocks cannot happen;
- PostgreSQL `now()` is fixed at transaction start, so every row gets the same timestamp.

**Truncation or a fresh database per test** is slower and required whenever the behavior under test involves commit, a second connection, locks or concurrency. Mark those tests and give them committed data.

**Django.** `TestCase` wraps each test in two nested `atomic()` blocks and checks deferrable constraints at the end of each test. `transaction.on_commit` callbacks do not run on their own inside it; use `self.captureOnCommitCallbacks(execute=True)` to run them. Django's docs state that some behavior cannot be tested in `TestCase`, such as code that must run inside a transaction for `select_for_update()`; use `TransactionTestCase`, which truncates tables after each test, for those and for any concurrency test.

## Faking third-party HTTP

Fake at the network edge, never by mocking your own client module, so the production client code (URL building, headers, serialization, error mapping, retries) runs in the test.

Build response fixtures from real recorded responses of the provider's sandbox, with secrets removed, and parse them with the schema the production client uses. A fixture that the production parser rejects fails at load time instead of drifting silently.

The examples use MSW 3. On MSW 2, import `http` and `HttpResponse` from `msw` and pass `onUnhandledRequest` instead of `onUnhandledFrame`; the strategies are the same.

```ts
import { http, HttpResponse } from "msw/http";
import { setupServer } from "msw/node";
import { afterAll, afterEach, beforeAll } from "vitest";
import payoutCreated from "./fixtures/provider/payout-created.json";
import { PayoutResponse } from "../src/provider/schema";
import { testConfig } from "./support/config";

const payoutsUrl = `${testConfig.providerBaseUrl}/payouts`;

export const server = setupServer(
  http.post(payoutsUrl, () => HttpResponse.json(PayoutResponse.parse(payoutCreated))),
);

beforeAll(() => server.listen({ onUnhandledFrame: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

Override per test for the failure shapes; each one needs a test asserting where the operation ends up:

```ts
import rateLimited from "./fixtures/provider/rate-limited.json";

server.use(http.post(payoutsUrl, () => HttpResponse.json(rateLimited, { status: 429 })));
server.use(http.post(payoutsUrl, () => HttpResponse.error()));
server.use(http.post(payoutsUrl, () => HttpResponse.text("<html>Bad gateway</html>", { status: 502 })));
```

`HttpResponse.error()` produces a network error: the case where the provider may or may not have acted. The code under test must treat it as an unknown outcome, not a failure.

`onUnhandledFrame: "error"` makes any request without a handler fail the test, so a missing fake cannot silently reach a real endpoint. MSW also sees requests your test sends to your own app over loopback (supertest does this); bypass those in a callback, as in `concurrency-and-idempotency.md`. In Python, pytest-socket does the same at the socket level: `--allow-hosts` blocks connections to any host not listed, so list only the database container's host.

Recorded cassettes (vcrpy) are an alternative. Scrub at record time and replay only in CI:

```python
import vcr

provider_vcr = vcr.VCR(
    cassette_library_dir=CASSETTE_DIR,
    filter_headers=["authorization"],
    record_mode="none",
)
```

`record_mode="none"` replays and raises on any request not in the cassette. Filter every header, query parameter and body field that carries a credential (`filter_query_parameters`, `filter_post_data_parameters`, `before_record_response` for response bodies), and run the repository's secret scanner over the cassette directory.

Recorded fixtures go stale when the provider changes. A scheduled job that calls the provider's sandbox and validates live responses against the same schema catches the drift before production does.
