---
name: database-engineering
description: Schemas, migrations, backfills, transactions, locking, indexes and query performance for PostgreSQL, SQLite and Cloudflare D1. Load it before writing or reviewing any SQL, ORM query, schema or migration, even one statement, and when debugging races, deadlocks, lost updates, duplicate rows, lock timeouts or slow queries.
license: MIT
metadata:
  author: Alex Tsanis
---

# Database engineering

The database is the one component that sees every concurrent request, so invariants belong there: constraints, conditional writes and short transactions. Most data bugs that reach production are code that is correct when one copy runs and wrong when two run at once, or DDL that is instant on a laptop and takes a lock that stops production. Establish the engine, version, isolation level, migration runner behavior and table sizes before judging anything, because PostgreSQL, SQLite and D1 give different answers to the same question.

## Before you judge or change anything

- Engine and major version (`SELECT version()`, `SELECT sqlite_version()`), the isolation level the code actually runs at, and whether the migration runner wraps each file in a transaction.
- Row counts and write rates for every table a migration touches. A table is big when a full scan or rewrite takes longer than clients will wait on a lock.
- The SQL the ORM really sends (query log, `EXPLAIN`), not the model code.
- For a claimed race: the two interleavings that break it, and whether a constraint, lock or later guard already stops them. Drop the claim if something does.

## Failure catalogue

### Concurrency and transactions

**Check-then-insert.** `SELECT ... WHERE email = $1`, then `INSERT` if nothing came back. Two requests both see nothing and both insert. `SELECT ... FOR UPDATE` does not help: it locks rows that exist, and the row you fear does not exist yet. ORM uniqueness validators and get-or-create helpers are the same pattern. Fix: a unique constraint or unique index, then `INSERT ... ON CONFLICT` and treat the conflict as the "already exists" answer. A pre-check may stay for a friendlier error; it is never the guard.

**Read-modify-write lost update.** Read `balance`, compute in application code, write `SET balance = $computed`. Without a lock held from the read to the write, a concurrent change is overwritten. PostgreSQL's default Read Committed does not prevent this. Fix, in order of preference: express the change in SQL with the guard in `WHERE` and check the affected row count; or `SELECT ... FOR UPDATE` in the same transaction before computing; or a version column.

```sql
-- Before: two concurrent debits both pass the app-side check; one is lost
SELECT balance_cents FROM wallets WHERE id = $1;
UPDATE wallets SET balance_cents = $2 WHERE id = $1;

-- After: one statement; the caller treats rowCount 0 as insufficient funds or not found
UPDATE wallets
SET balance_cents = balance_cents - $2
WHERE id = $1 AND balance_cents >= $2
RETURNING balance_cents;
```

This is safe under PostgreSQL Read Committed: when a concurrent transaction changed the row, the second `UPDATE` waits for it, then re-evaluates its `WHERE` against the committed version. SQLite and D1 serialize writers, so the same statement is safe there too.

**Ignored affected-row count.** A guarded `UPDATE ... WHERE status = 'pending'` that changes zero rows means another request won, the row is gone, or the caller does not own it. Code that ignores `rowCount` (node-postgres) or `meta.changes` (D1) and goes on to send the email or charge the card has the race back. Zero rows must become an explicit conflict or not-found result.

**Trusting Read Committed with multi-row invariants.** Read Committed takes a new snapshot per statement, so two reads in one transaction can disagree. A rule over a set of rows ("one doctor stays on call", "allocations never exceed the budget", "bookings never overlap") breaks when two transactions each read the set and write a different row (write skew). PostgreSQL Repeatable Read allows write skew too. Fix: a constraint when the rule is expressible (unique, `CHECK`, a PostgreSQL exclusion constraint for overlaps); otherwise lock a row every writer must touch (the budget, the shift) with `FOR UPDATE`, or run `SERIALIZABLE` with a retry loop.

**Serializable without retries.** PostgreSQL `SERIALIZABLE` and `REPEATABLE READ` abort a transaction with SQLSTATE `40001` where Read Committed would have carried on, and under `SERIALIZABLE` the error can arrive on any statement including `COMMIT`. Without a loop that reruns the whole transaction from `BEGIN` (reads included) you have swapped a data bug for random 500s. Keep HTTP calls, emails and publishes out of the retried block. Template in `references/postgresql.md`.

**Inconsistent lock order.** Transfer A to B locks A then B; a concurrent B to A locks B then A. PostgreSQL detects the deadlock and aborts one with `40P01`. Multi-row `UPDATE ... WHERE id = ANY($1)` over overlapping sets does the same, because rows lock in scan order. Fix: lock everything first in key order (`SELECT ... WHERE id = ANY($1) ORDER BY id FOR UPDATE`), then write, and treat `40P01` as retryable.

**Transaction spanning a network call.** `BEGIN`, lock the order, call the payment provider or a fraud API, `COMMIT`. Row locks and a pooled connection are held for the remote call's latency and full timeout, so a slow dependency exhausts the pool and blocks every writer of those rows. If `COMMIT` fails after the provider succeeded, money moved and the database says it did not. Fix: commit a `pending` row with an operation id, call the provider outside any transaction with that id as its idempotency key, record the result in a second short transaction guarded on `status = 'pending'`, and have a reconciler resolve rows stuck in `pending`. Messages that must follow a commit go through an outbox table written in the same transaction.

**Treating a failed commit as a rollback.** A timeout or dropped connection during `COMMIT`, or a D1 write that fails with "Network connection lost", is an unknown outcome. Retrying a non-idempotent write double-applies it. Retry only writes carrying an idempotency key backed by a unique constraint, or re-read to learn what happened.

**Leaked and idle transactions.** An error path that skips `ROLLBACK`, a pool checkout never released, or a slow report holds locks, blocks DDL behind it and, in PostgreSQL, stops vacuum from removing dead rows. Release connections in `finally`. Set `statement_timeout` and `idle_in_transaction_session_timeout` per role or session.

**Polling by id or timestamp over live writes.** A consumer reading `WHERE id > $last_seen` (or `created_at >`) skips rows: sequence values and PostgreSQL `now()` (transaction start time) are assigned before commit, so a row with a smaller id can become visible after the consumer moved past it. Fix: an outbox or jobs table with explicit status, claimed with `FOR UPDATE SKIP LOCKED`, or change data capture. Keyset pagination for people browsing is fine; this is about consumers that must see every row.

**Read-after-write against a replica.** Write to the primary, redirect, read from a replica, show stale data or re-run the action. Route reads that follow a write in the same flow to the primary, or use the engine's session mechanism (D1 Sessions API bookmarks).

### Types, NULLs and schema

**Float money.** `REAL`, `float`, `double precision` or fractional JavaScript numbers for money. Rounding drifts and totals stop reconciling. Store integer minor units (`bigint`, SQLite `INTEGER`) or PostgreSQL `numeric`. SQLite has no decimal type (`NUMERIC` is only an affinity), and D1 returns integers as JavaScript numbers, exact only up to `Number.MAX_SAFE_INTEGER`; token base units and other amounts that can exceed it go in as decimal `TEXT` and are computed with `BigInt`.

**Timezone-naive timestamps.** PostgreSQL `timestamp` (without time zone) silently ignores any offset in the input, so the stored value's zone is a guess. Use `timestamptz`. SQLite and D1 have no date type: `datetime('now')` yields `YYYY-MM-DD HH:MM:SS` while JavaScript `toISOString()` yields `YYYY-MM-DDTHH:MM:SS.sssZ`, and `TEXT` comparison is lexical with `'T'` sorting after `' '`. So `expires_at > datetime('now')` on an ISO value treats a session that expired this morning as valid until midnight UTC. Pick one representation per schema; integer epoch with the unit in the column name (`expires_at_ms`) is the hardest to get wrong.

**32-bit keys.** PostgreSQL `serial`/`integer` keys stop at 2,147,483,647 and inserts fail when the sequence runs out; widening later rewrites the table. Use `bigint GENERATED ALWAYS AS IDENTITY` or UUIDs for tables that grow with traffic.

**NULLs in unique constraints.** PostgreSQL (by default) and SQLite treat NULLs as distinct, so `UNIQUE (tenant_id, external_id)` accepts any number of rows with `external_id` NULL, and `UNIQUE (email, deleted_at)` never stops a second live row. Fix: `NOT NULL`, a partial unique index, or `UNIQUE NULLS NOT DISTINCT` on PostgreSQL 15+.

**NULLs in comparisons.** `status <> 'archived'` drops rows where status is NULL. `NOT IN (SELECT parent_id ...)` returns nothing if the subquery yields one NULL; use `NOT EXISTS`. `CHECK (amount > 0)` passes when `amount` is NULL. Declare `NOT NULL` unless absence means something, and compare with `IS DISTINCT FROM` (PostgreSQL) or `IS NOT` (SQLite) where NULL is a real value.

**Soft delete breaking uniqueness and filters.** After adding `deleted_at`, `UNIQUE (email)` blocks re-registration, and every query, join or count that forgets `deleted_at IS NULL` resurrects deleted data. Fix: a partial unique index `WHERE deleted_at IS NULL`, and the filter in one view or repository function rather than at each call site. Consider an archive table instead.

**Missing tenant scoping.** `SELECT * FROM invoices WHERE id = $1` in a multi-tenant schema returns another tenant's invoice for a guessed id. Every query on tenant-owned data filters by a tenant id taken from the authenticated session, never from the request. Carry `tenant_id` into child tables with composite foreign keys (`FOREIGN KEY (tenant_id, invoice_id) REFERENCES invoices (tenant_id, id)`, which needs `UNIQUE (tenant_id, id)` on the parent) so rows cannot point across tenants. PostgreSQL row-level security has sharp edges; see `references/postgresql.md`.

**SQLite defaults left in place.** Non-`STRICT` tables let an `INTEGER` column hold `'abc'`. A non-integer `PRIMARY KEY` without `NOT NULL` accepts NULL. Foreign keys are unenforced unless every connection runs `PRAGMA foreign_keys = ON` (D1 enforces them by default).

### Migrations

**DDL stuck in the lock queue.** Most PostgreSQL `ALTER TABLE` forms take `ACCESS EXCLUSIVE`, including instant ones such as adding a nullable column or renaming. If any transaction holds a lock on the table, the `ALTER` waits, and every query that arrives after it queues behind the `ALTER`. A metadata-only change takes the application down. Fix: `SET lock_timeout` for the migration so it gives up quickly, retry it, and check `pg_stat_activity` for long transactions first.

**SET NOT NULL on a populated table.** Scans the whole table under `ACCESS EXCLUSIVE`. Fix on PostgreSQL 12+: add `CHECK (col IS NOT NULL) NOT VALID`, `VALIDATE CONSTRAINT` (which takes only `SHARE UPDATE EXCLUSIVE`), then `SET NOT NULL`, which skips the scan because the valid check proves it, then drop the check. Foreign keys and other checks use the same `NOT VALID` then `VALIDATE` split. Not a problem: `ADD COLUMN ... NOT NULL DEFAULT <constant>` on PostgreSQL 11+ stores the default in the catalog without a rewrite. A volatile default (`clock_timestamp()`, `gen_random_uuid()`) does rewrite the table.

**Rewrites that look small.** `ALTER COLUMN ... TYPE` rewrites the table and rebuilds indexes unless the old type is binary-coercible to the new one; `integer` to `bigint` is a full rewrite. Add a new column, dual-write, backfill, swap.

**Blocking index builds.** Plain `CREATE INDEX` blocks writes for the whole build. Use `CREATE INDEX CONCURRENTLY`, which cannot run inside a transaction block, so that migration must opt out of the runner's transaction wrapper. A failed concurrent build leaves an `INVALID` index that still slows writes; drop it and retry. For a new unique constraint, build the unique index concurrently, then `ADD CONSTRAINT ... UNIQUE USING INDEX`.

**Rename or drop in the same release as the code change.** During a rolling deploy old and new code run together, and the old code still queries the old column. Dropping a column that running code or an ORM's cached column list still selects fails the same way. Use expand and contract across releases (`references/patterns.md`).

**Backfill in one statement.** `UPDATE users SET email_normalized = lower(email)` over a large table in one transaction locks every row it touches until the end, produces the table's worth of WAL and replica lag at once, bloats the table, and starts over from zero if it fails. Backfill in key-ordered batches, one transaction each, idempotent (`WHERE new_col IS NULL`), resumable from a stored cursor, throttled, and run as a job separate from the schema migration.

**Migrations that cannot be rerun.** Non-transactional steps (`CREATE INDEX CONCURRENTLY`, SQLite table rebuilds with pragmas, D1 statements hitting the time limit) fail halfway. Make every step safe to repeat: `IF NOT EXISTS` on creates, `ON CONFLICT DO NOTHING` on seed rows, `WHERE` guards on data fixes. Once a migration has run in any shared environment it is immutable; corrections go in a new file.

### Queries and indexes

**N+1.** A list endpoint loads 50 orders, then lazily loads each customer and each order's items: over 100 queries. Invisible in tests, slow in production, and on D1 each query is a round trip. Load each relation with one join or one `WHERE id = ANY($1)` / `IN (...)` query, and assert query counts in tests for list endpoints.

**OFFSET pagination over live data.** `ORDER BY created_at DESC LIMIT 20 OFFSET 40` repeats or skips rows when rows are inserted or deleted between requests, and the server still reads every skipped row. Sorting on a non-unique column with no tiebreaker makes page order arbitrary. Use keyset pagination with a unique tiebreaker, backed by a matching index (`references/patterns.md`).

**Unindexed foreign keys.** PostgreSQL does not index the referencing columns of a foreign key, and SQLite recommends indexing them for the same reason: every parent delete or key update scans the child table, per row and per cascade level. Index foreign key columns unless the parent is never deleted or re-keyed and the child is never looked up by parent.

**Composite index in the wrong order.** An index on `(created_at, tenant_id)` barely helps `WHERE tenant_id = $1 ORDER BY created_at DESC`. Equality columns first, then the range or sort column: `(tenant_id, created_at)`. An index on `(a, b)` serves filters on `a` alone, not on `b` alone.

**Predicates that cannot use the index.** `LIKE '%term%'` cannot use a B-tree (PostgreSQL: `pg_trgm` GIN index; SQLite and D1: FTS5). `lower(email) = $1` needs an expression index on `lower(email)`. `created_at::date = $1` or `date(created_at) = ?` hides the column inside a function; rewrite as `created_at >= $1 AND created_at < $2`. A bound parameter of a different type than the column can put a cast on the column side; look for it in `EXPLAIN`. On PostgreSQL with a non-C collation, `LIKE 'prefix%'` needs a `text_pattern_ops` index.

**ORM surprises.** Lazy loading in loops. `save()` writing every column, which overwrites a concurrent change to a column this request never touched (Django `save()` without `update_fields` does this). Uniqueness validators and `get_or_create`/`findOrCreate` with no unique constraint behind them. Bulk update helpers that skip validation and hooks. Transactions that are opened, or not, differently from what the code suggests. Read the query log.

**Dynamic identifiers.** Column, table and sort-direction names cannot be bound as parameters. Map request values through an allowlist to fixed SQL fragments; bind all values.

## Decision rules

Concurrency control for a write:
- New value computable from the current row: one `UPDATE` with the guard in `WHERE`, check the row count. Default choice.
- User edited a copy read earlier (form, `PATCH` with `If-Match`): version column, zero rows returns a conflict.
- Several statements need stable rows you can name: one short transaction, `SELECT ... ORDER BY id FOR UPDATE` first.
- Rule over rows that may not exist yet, or over a set: a constraint if expressible; otherwise lock a parent row every writer touches, or `SERIALIZABLE` with whole-transaction retry.
- SQLite and D1: one statement or one D1 `batch()` is atomic and serialized; a read in an earlier call is stale by the time the write runs.

Pagination: keyset for feeds, infinite scroll, exports and sync jobs. `OFFSET` only for small, slow-changing admin lists that need page numbers.

Timestamps: PostgreSQL `timestamptz`. SQLite and D1: integer epoch with the unit in the name, or ISO-8601 text written by exactly one function in exactly one format.

Money: integer minor units when the currency exponent is fixed and amounts fit the client's integer range; PostgreSQL `numeric` for rates and fractional quantities; decimal `TEXT` plus `BigInt` for large token amounts on SQLite and D1.

Idempotent create: unique key, `INSERT ... ON CONFLICT DO NOTHING RETURNING`. No row returned means a previous attempt exists: load it and compare the stored request fingerprint; same request returns the stored result, different request returns a conflict.

Schema change on a live table: if the previous release breaks against the new schema, or the step scans or rewrites a big table, it is several steps across releases, never one migration.

## Review checklist

- Is every check-then-write a single conditional statement, backed by a constraint, or under a lock or isolation level that makes it safe?
- Is the affected row count of every guarded write checked, with zero rows handled?
- Where a transaction locks several rows, are they locked in key order?
- Is anything inside an open transaction waiting on the network or a user?
- Is every write retry backed by an idempotency key and a unique constraint, and does `40001`/`40P01` handling rerun the whole transaction?
- Is money stored as integers or exact decimals, and time as `timestamptz` or one fixed SQLite representation?
- Do nullable columns in unique constraints, `<>` filters and `NOT IN` subqueries behave as intended?
- Does every query on tenant-owned data filter by a tenant id derived from the session?
- For each migration statement: which lock, does it scan or rewrite, how long at production size, is `lock_timeout` set?
- Can the previous release run against the new schema?
- Are backfills batched, idempotent, resumable and outside the schema migration?
- Can every migration be rerun after failing halfway?
- Do new foreign keys have indexes, and do new queries match an index in column order according to `EXPLAIN`?
- Are list endpoints free of N+1 and of `OFFSET` over live data?
- SQLite: `foreign_keys` and `busy_timeout` set per connection, `BEGIN IMMEDIATE` for read-then-write? D1: multi-statement writes in one `batch()`, with guards that abort the batch rather than silently matching zero rows?
- Has the concurrent path been exercised against the real engine with two connections?

## References

- `references/patterns.md`: read when writing a guarded status transition, idempotent insert, keyset pagination, expand-contract migration or batched backfill.
- `references/postgresql.md`: read before writing or reviewing PostgreSQL migrations, locking, isolation levels, retry loops, job queues, row-level security or pool settings.
- `references/sqlite-d1.md`: read for any SQLite or Cloudflare D1 work: connection pragmas, write locking, types and dates, ALTER TABLE limits, D1 batch semantics, limits, replicas and retries.

Related skills: security-engineering for authorization and injection beyond SQL, backend-architecture for idempotency, outbox and reconciliation across services, performance-benchmarking for measuring query changes.
