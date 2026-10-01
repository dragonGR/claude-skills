# SQLite and Cloudflare D1 specifics

Read this for any SQLite or D1 work: connection setup, write locking, types and dates, ALTER TABLE limits, D1 batch semantics, limits, read replicas and retries.

## SQLite connection setup

Per connection, before any transaction:

- `PRAGMA foreign_keys = ON`
- `PRAGMA busy_timeout = N`, with N in milliseconds from configuration. Pragmas do not take bound parameters, so validate N as an integer before building the statement.

Why:

- `foreign_keys` is off by default and must be set on every connection, including ones opened by migration tools, admin scripts and test fixtures. Inside a multi-statement transaction the pragma is silently ignored.
- Without `busy_timeout`, a writer that finds the database locked fails with `SQLITE_BUSY` at once.
- `PRAGMA journal_mode = WAL` is persistent: set it once per database file. In WAL mode readers and the writer do not block each other, but there is still only one writer at a time. WAL does not work over a network filesystem. With readers that never pause, checkpoints cannot finish and the WAL file grows without bound, so avoid permanently open read transactions.

## Write locking

A plain `BEGIN` is deferred: it starts as a read transaction and upgrades on the first write. If another connection wrote in between, the upgrade fails with `SQLITE_BUSY` (`SQLITE_BUSY_SNAPSHOT` in WAL mode). `busy_timeout` does not save you: waiting cannot make a stale snapshot current, and in rollback-journal mode SQLite returns `SQLITE_BUSY` without calling the busy handler when waiting could deadlock.

```sql
-- Before: read then write in a deferred transaction; fails under concurrent writers
BEGIN;
SELECT stock FROM products WHERE id = ?;
UPDATE products SET stock = ? WHERE id = ?;
COMMIT;

-- After: take the write lock up front, or skip the transaction and use one guarded statement
BEGIN IMMEDIATE;
UPDATE products SET stock = stock - ? WHERE id = ? AND stock >= ?;
COMMIT;
```

Use `BEGIN IMMEDIATE` for every transaction that might write, and check what your driver's transaction helper issues. One writer per file means a long write transaction stalls every other writer; keep write transactions short and batch large jobs.

## Types

- Declared types are affinities. A non-`STRICT` `INTEGER` column stores `'abc'` without complaint. `STRICT` tables (SQLite 3.37.0+) allow only `INT`, `INTEGER`, `REAL`, `TEXT`, `BLOB` and `ANY` and reject values that cannot be converted losslessly.
- A `PRIMARY KEY` that is not `INTEGER PRIMARY KEY`, in a table that is neither `STRICT` nor `WITHOUT ROWID`, accepts NULL unless the column is declared `NOT NULL`.
- Without `AUTOINCREMENT`, an `INTEGER PRIMARY KEY` can reuse the id of a deleted row. If ids escape into URLs, caches or other systems, a new row can inherit references meant for a deleted one. Use `AUTOINCREMENT` there or an external id.
- Integers are signed 64-bit. `sum()` raises "integer overflow" when all inputs are integers and the total overflows; `total()` and `avg()` return floating point. There is no exact decimal type: money is integer minor units, large amounts are decimal `TEXT` computed in application code.
- Booleans are stored as `0` and `1`.

## Dates

There is no date type. The built-in functions accept ISO-8601 text, Julian day numbers and Unix epoch integers, and `datetime('now')` returns UTC text as `YYYY-MM-DD HH:MM:SS`. Text comparisons are lexical, so values written in different text formats do not compare correctly: JavaScript's `toISOString()` gives `2025-01-01T09:00:00.000Z`, which sorts after `2025-01-01 10:00:00` because `'T'` is greater than `' '`.

```ts
// Before: ISO text compared with datetime('now'); expired sessions stay valid until midnight UTC
await db.prepare("INSERT INTO sessions (token, user_id, expires_at) VALUES (?, ?, ?)")
  .bind(token, userId, new Date(Date.now() + sessionTtlMs).toISOString()).run();
await db.prepare("SELECT user_id FROM sessions WHERE token = ? AND expires_at > datetime('now')")
  .bind(token).first();

// After: integer epoch milliseconds, the current time bound from one clock
await db.prepare("INSERT INTO sessions (token, user_id, expires_at_ms) VALUES (?, ?, ?)")
  .bind(token, userId, Date.now() + sessionTtlMs).run();
await db.prepare("SELECT user_id FROM sessions WHERE token = ? AND expires_at_ms > ?")
  .bind(token, Date.now()).first();
```

For existing text columns, normalizing both sides with `unixepoch(expires_at) > unixepoch()` gives correct results but cannot use an index on the column; migrate to one stored format instead.

## NULLs, uniqueness and matching

- NULLs are distinct in `UNIQUE` constraints. Partial indexes (`CREATE UNIQUE INDEX ... WHERE deleted_at IS NULL`) solve the soft-delete and optional-key cases.
- `IS` and `IS NOT` are the NULL-safe equality operators.
- `LIKE` is case-insensitive for ASCII by default, unlike PostgreSQL. Code ported between the two changes behavior silently. A leading `%` cannot use an index; use FTS5 for text search.
- `EXPLAIN QUERY PLAN` shows `SCAN` for a full table scan and `SEARCH ... USING INDEX` for index lookups.

## ALTER TABLE limits

`ADD COLUMN` cannot add `PRIMARY KEY` or `UNIQUE`, cannot default to `CURRENT_TIMESTAMP` or an expression, needs a non-NULL default when `NOT NULL`, and (with foreign keys on) needs a NULL default for a `REFERENCES` column. From 3.37.0 an added `CHECK` is tested against existing rows. `DROP COLUMN` fails if the column is indexed, unique, part of the primary key, a foreign key, or used by a view, trigger, generated column or another check.

Anything else is the documented table rebuild: disable foreign keys, begin, create `new_x`, copy rows, drop `x`, rename `new_x` to `x`, recreate indexes, triggers and views, run `PRAGMA foreign_key_check`, commit, re-enable foreign keys. Create under the new name and rename into place; the reverse order corrupts references. Because `PRAGMA foreign_keys` is ignored inside a transaction, the disable must happen before `BEGIN`.

## Cloudflare D1

D1 runs SQLite behind a Worker binding. What changes:

**No interactive transactions.** There is no `BEGIN`/`COMMIT` over the binding; D1's import instructions have you remove them from dumps because D1 already runs statements inside its own transaction. Statements that must commit together go in one `db.batch([...])`. A batch runs its statements sequentially as one transaction; if any statement errors, the whole batch rolls back.

**Zero rows is not an error.** A guarded `UPDATE ... WHERE uses < max_uses` that matches nothing succeeds with `meta.changes === 0`, and the next statement in the batch still runs. Make the guard abort the batch, or repeat the guard in every dependent statement.

```ts
// Before: the check runs in a separate call, and nothing in the batch enforces it
const coupon = await env.DB.prepare("SELECT uses, max_uses FROM coupons WHERE code = ?").bind(code).first();
if (!coupon || coupon.uses >= coupon.max_uses) return { ok: false };
await env.DB.batch([
  env.DB.prepare("UPDATE coupons SET uses = uses + 1 WHERE code = ?").bind(code),
  env.DB.prepare("INSERT INTO redemptions (code, user_id, redeemed_at_ms) VALUES (?, ?, ?)").bind(code, userId, Date.now()),
]);
```

```sql
-- After, schema: the limit, the one-per-user rule and the coupon's existence are constraints
CREATE TABLE coupons (
  code     TEXT PRIMARY KEY NOT NULL,
  max_uses INTEGER NOT NULL,
  uses     INTEGER NOT NULL DEFAULT 0,
  CHECK (uses <= max_uses)
) STRICT;

CREATE TABLE redemptions (
  id             INTEGER PRIMARY KEY,
  code           TEXT NOT NULL REFERENCES coupons (code),
  user_id        TEXT NOT NULL,
  redeemed_at_ms INTEGER NOT NULL,
  UNIQUE (code, user_id)
) STRICT;
```

```ts
function isConstraintError(err: unknown): boolean {
  return err instanceof Error && /\b(UNIQUE|CHECK|FOREIGN KEY) constraint failed\b/.test(err.message);
}

// After: any violated constraint fails its statement and rolls back the whole batch
try {
  await env.DB.batch([
    env.DB.prepare("UPDATE coupons SET uses = uses + 1 WHERE code = ?").bind(code),
    env.DB.prepare("INSERT INTO redemptions (code, user_id, redeemed_at_ms) VALUES (?, ?, ?)").bind(code, userId, Date.now()),
  ]);
  return { ok: true };
} catch (err) {
  if (isConstraintError(err)) return { ok: false, reason: "not_redeemable" };
  throw err;
}
```

Exhausted coupons fail the `CHECK`, a second redemption by the same user fails the `UNIQUE`, and an unknown code fails the foreign key; each rolls back the counter increment. Match constraint errors narrowly and rethrow everything else. If the response must say which rule failed, read the state after the rollback; do not decide anything from that read. When a constraint cannot express the guard, make each dependent statement read state that only the successful guarded write could have produced, and back it with a unique constraint:

```sql
UPDATE orders SET status = 'paid', payment_id = ?2 WHERE id = ?1 AND status = 'pending';
INSERT INTO ledger_entries (order_id, payment_id, amount_minor)
  SELECT id, payment_id, total_minor FROM orders WHERE id = ?1 AND payment_id = ?2;
-- ledger_entries has UNIQUE (order_id), so a replay of the same payment aborts the batch
```

Test the zero-row path explicitly; it is the one that silently does the wrong thing.

**Reads before writes are stale.** A `first()` in one call and a `batch()` in the next are separate transactions, and other requests' writes land between them. D1 processes one query at a time per database, which serializes statements, not your multi-call logic.

**Result shape.** `run()` and each `batch()` element return `meta.changes`, `meta.last_row_id`, `meta.rows_read` and `meta.rows_written`. `first()` returns `null` when no row matches. `exec()` takes no bound parameters and is for one-off maintenance only; never pass request data to it.

**Values.** Integers come back as JavaScript numbers, exact only up to `Number.MAX_SAFE_INTEGER`; the binding does not accept `BigInt`. Booleans are written as `1`/`0` and read back as numbers. Binding `undefined` fails with `D1_TYPE_ERROR`; map optional fields to `null` explicitly.

**Limits that shape designs.** The documented limits include 100 bound parameters per query, 100 KB per SQL statement, 2 MB per row or value, a 30 second query duration that also applies to an entire `batch()` call, and a cap on queries per Worker invocation (lower on the free plan). Recheck the limits page before designing near any of them. For long `IN` lists, bind one JSON array instead of one parameter per value:

```ts
await env.DB.prepare("SELECT id, email FROM users WHERE id IN (SELECT value FROM json_each(?))")
  .bind(JSON.stringify(userIds))
  .all();
```

**Throughput.** Each database is single-threaded and runs queries one at a time, so throughput is roughly the inverse of average query time, and one slow scan delays every request. Excess concurrent requests queue, then fail as overloaded. Index for every hot query and check `meta.rows_read`.

**Foreign keys.** D1 enforces foreign keys by default and does not let you switch enforcement off outside a single transaction. Migrations that rebuild a table start with `PRAGMA defer_foreign_keys = true`, which postpones checks until the end of that transaction.

**Migrations.** Wrangler records applied files in the `d1_migrations` table. A file that errors is rolled back and earlier files stay applied. A dropped connection during apply is still an unknown outcome, so write each migration to be rerunnable (`IF NOT EXISTS`, guarded data steps) and apply it to a local copy (`wrangler d1 migrations apply <db> --local`) before production.

**Read replication.** Without the Sessions API every query goes to the primary. With `env.DB.withSession(...)`, reads may be served by replicas; the session gives read-your-writes and monotonic reads within itself. Across requests, return `session.getBookmark()` to the client and start the next request's session from it, or a user can write and then read older data. Start from `"first-primary"` when the first read must be current. Writes always go to the primary.

**Retries.** D1 retries read-only queries (only `SELECT`, `EXPLAIN`, `WITH`) itself. Writes are not retried for you. Errors such as "Network connection lost" on a write leave the outcome unknown; retry only writes that are idempotent by construction (upserts keyed on an idempotency key, guarded updates) with exponential backoff and jitter.
