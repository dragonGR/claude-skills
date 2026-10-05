# PostgreSQL specifics

Read this before writing or reviewing a PostgreSQL migration, locking code, isolation-level change, retry loop, job queue, row-level security policy or connection-pool setting.

## What common DDL does to a live table

`ALTER TABLE` takes `ACCESS EXCLUSIVE` unless the documentation for that form says otherwise. `ACCESS EXCLUSIVE` conflicts with everything, including plain `SELECT`, and a waiting request blocks every later request on the table. Duration matters less than whether the statement has to wait.

| Change | Lock | Work done | Safe approach |
| --- | --- | --- | --- |
| `ADD COLUMN` nullable, with a constant default, or virtual generated (18+, the default kind of generated column) | `ACCESS EXCLUSIVE`, brief | catalog only | `lock_timeout` and retry |
| `ADD COLUMN` with volatile default (`clock_timestamp()`, `gen_random_uuid()`), stored generated or identity column, or a domain type with constraints | `ACCESS EXCLUSIVE` | table and index rewrite | add nullable, backfill in batches, then set the default for new rows |
| `ALTER COLUMN SET NOT NULL` | `ACCESS EXCLUSIVE` | full scan, skipped if a valid `CHECK (col IS NOT NULL)` exists | check `NOT VALID`, validate, set, drop check |
| `ADD CONSTRAINT ... CHECK` / `FOREIGN KEY` | `CHECK`: `ACCESS EXCLUSIVE`; foreign key: `SHARE ROW EXCLUSIVE` on both the table and the referenced table (blocks writes) | full scan | add `NOT VALID`, then `VALIDATE CONSTRAINT` |
| `VALIDATE CONSTRAINT` | `SHARE UPDATE EXCLUSIVE` (reads and writes continue); a foreign key also takes `ROW SHARE` on the referenced table | full scan | run in its own transaction |
| `ALTER COLUMN TYPE` | `ACCESS EXCLUSIVE` | rewrite and index rebuild unless binary-coercible | new column, dual-write, backfill, swap |
| `CREATE INDEX` | blocks writes for the whole build | full build | `CREATE INDEX CONCURRENTLY` outside a transaction block |
| `CREATE INDEX CONCURRENTLY` | `SHARE UPDATE EXCLUSIVE` | two scans, waits for older transactions | check for `INVALID` leftovers on failure |
| `RENAME COLUMN` / `RENAME TO` | `ACCESS EXCLUSIVE`, brief | catalog only | expand and contract; running code breaks, not the database |
| `DROP COLUMN` | `ACCESS EXCLUSIVE`, brief | catalog only; space not reclaimed | stop all reads first |

PostgreSQL 18 can also add a `NOT NULL` constraint as `NOT VALID` and validate it later. On 17 and earlier use the check-constraint route below.

## Migration session settings

```sql
SET lock_timeout = '3s';
SET statement_timeout = '15min';
```

The values are the migration's budget: `lock_timeout` is how long production traffic may queue behind this statement, `statement_timeout` bounds the work once the lock is held. Choose them per migration and keep them in the migration runner's configuration, not in `postgresql.conf` (the documentation discourages setting either globally). A lock timeout fails with SQLSTATE `55P03`; the runner retries the step with backoff rather than failing the deploy on the first attempt. Before a risky step, look for what it would wait behind:

```sql
SELECT pid, state, now() - xact_start AS xact_age, wait_event_type, left(query, 200) AS query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;

SELECT pid, pg_blocking_pids(pid) AS blocked_by, left(query, 200) AS query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

## Recipes

NOT NULL on a populated column. Run each statement as its own migration step:

```sql
ALTER TABLE orders ADD CONSTRAINT orders_customer_id_not_null
  CHECK (customer_id IS NOT NULL) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT orders_customer_id_not_null;
ALTER TABLE orders ALTER COLUMN customer_id SET NOT NULL;   -- no scan: the valid check proves it
ALTER TABLE orders DROP CONSTRAINT orders_customer_id_not_null;
```

Foreign key on a large table:

```sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS order_items_order_id_idx ON order_items (order_id);
ALTER TABLE order_items ADD CONSTRAINT order_items_order_id_fkey
  FOREIGN KEY (order_id) REFERENCES orders (id) NOT VALID;
ALTER TABLE order_items VALIDATE CONSTRAINT order_items_order_id_fkey;
```

`NOT VALID` still enforces the constraint for new writes immediately; only existing rows wait for `VALIDATE`.

Unique constraint without blocking writes:

```sql
CREATE UNIQUE INDEX CONCURRENTLY users_email_normalized_key ON users (email_normalized);
ALTER TABLE users ADD CONSTRAINT users_email_normalized_key
  UNIQUE USING INDEX users_email_normalized_key;
```

The index for `USING INDEX` must be a plain B-tree with default ordering, not partial and without expressions. If you need a partial unique rule (`WHERE deleted_at IS NULL`), keep the unique index itself as the enforcement; it works with `ON CONFLICT` when the conflict target repeats the predicate.

After any failed concurrent build:

```sql
SELECT indexrelid::regclass AS index_name FROM pg_index WHERE NOT indisvalid;
```

Drop what it lists (`DROP INDEX CONCURRENTLY`) and rerun. `CREATE INDEX CONCURRENTLY IF NOT EXISTS` treats an invalid leftover with the same name as existing and does nothing, so a rerunnable migration has to check `indisvalid`, not just the name. Migration tools wrap files in transactions by default; the concurrent step needs the tool's opt-out (Rails `disable_ddl_transaction!`, a Django migration with `atomic = False`, or the equivalent) and should contain nothing else.

Widening `integer` to `bigint` on a big table: add `id_new bigint`, dual-write with a trigger or the application, backfill, build the unique index concurrently, then in one short transaction under `lock_timeout` swap the primary key and sequence ownership. Plan it before the sequence gets close to its limit; the swap is the only step that needs `ACCESS EXCLUSIVE`.

## Isolation levels in practice

- Read Committed (default): each statement sees data committed before it started. `UPDATE`, `DELETE` and `SELECT ... FOR UPDATE` that meet a concurrently modified row wait for it, then re-check their `WHERE` against the new version. This is what makes single-statement guards safe, and what makes multi-statement read-then-write logic unsafe.
- Repeatable Read: one snapshot for the whole transaction. Updating a row changed by a concurrent committed transaction fails with `40001`; read-only transactions never do. Write skew is still possible.
- Serializable: Repeatable Read plus detection of dangerous read/write dependencies, failing with `40001`. Here even a read-only transaction can be cancelled, unless it is declared `SERIALIZABLE READ ONLY DEFERRABLE`, which waits for a safe snapshot first and suits long consistent reports.

Retry template (node-postgres):

```ts
import type { Pool, PoolClient } from "pg";

const RETRYABLE_SQLSTATES = new Set(["40001", "40P01"]);

interface RetryConfig {
  maxAttempts: number;
  baseDelayMs: number;
}

function sqlstate(err: unknown): string | undefined {
  if (typeof err === "object" && err !== null && "code" in err && typeof err.code === "string") {
    return err.code;
  }
  return undefined;
}

export async function inSerializableTransaction<T>(
  pool: Pool,
  cfg: RetryConfig,
  work: (client: PoolClient) => Promise<T>,
): Promise<T> {
  for (let attempt = 1; ; attempt++) {
    const client = await pool.connect();
    try {
      await client.query("BEGIN ISOLATION LEVEL SERIALIZABLE");
      const result = await work(client);
      await client.query("COMMIT");
      client.release();
      return result;
    } catch (err) {
      const rollbackError = await client.query("ROLLBACK").then(
        () => undefined,
        (e: unknown) => (e instanceof Error ? e : new Error(String(e))),
      );
      client.release(rollbackError);
      const code = sqlstate(err);
      if (code === undefined || !RETRYABLE_SQLSTATES.has(code) || attempt >= cfg.maxAttempts) throw err;
      const delay = cfg.baseDelayMs * 2 ** (attempt - 1);
      await new Promise((resolve) => setTimeout(resolve, delay / 2 + Math.random() * (delay / 2)));
    }
  }
}
```

Add `23505` or `23P01` to the set only for transactions that pick the conflicting value from their own reads (the next free key after reading the current ones); the PostgreSQL docs call that a serialization failure the server cannot detect. Anywhere else a unique violation is the answer, and retrying it hides a bug.

`work` must be free of side effects outside this transaction because it can run more than once. A connection error during `COMMIT` carries no retryable SQLSTATE and is rethrown: the outcome is unknown, and only an idempotency key makes a retry safe.

## Row locks, queues and advisory locks

Lock several rows in a stable order before writing any of them:

```sql
SELECT id, balance_minor FROM wallets WHERE id = ANY($1) ORDER BY id FOR UPDATE;
```

Job queue claim that lets many workers run without blocking each other. Expired leases are claimed directly, and every claim increments `lease_token`, the fencing token:

```sql
WITH next AS (
  SELECT id FROM jobs
  WHERE kind = $1
    AND run_after <= now()
    AND attempts < max_attempts
    AND (state = 'queued' OR (state = 'running' AND lease_expires_at < now()))
  ORDER BY run_after, id
  LIMIT $2
  FOR UPDATE SKIP LOCKED
)
UPDATE jobs j
SET state = 'running',
    attempts = j.attempts + 1,
    lease_token = j.lease_token + 1,
    leased_by = $3,
    lease_expires_at = now() + make_interval(secs => $4::double precision)
FROM next
WHERE j.id = next.id
RETURNING j.id, j.payload, j.attempts, j.lease_token;
```

Commit the claim immediately and do the work outside the transaction. A worker that stalls past its lease (GC pause, CPU throttling) keeps running after another worker has taken the job, so every later write it makes (renew, complete, fail, business rows) adds `AND lease_token = $token`, and zero rows means the lease was lost and the worker stops. The handler must still be idempotent, because a job whose worker died after its side effect runs again. `SKIP LOCKED` gives an inconsistent view by design and belongs only on queue-like tables. The jobs table, renew and finish statements, heartbeat and worker loop are in backend-architecture's `jobs-and-state-machines.md`.

Overlapping ranges, enforced by the database:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE bookings ADD CONSTRAINT bookings_no_overlap
  EXCLUDE USING gist (room_id WITH =, during WITH &&);
```

Advisory locks: use `pg_advisory_xact_lock`, which releases at transaction end. Session-level advisory locks survive a rollback and, behind a transaction-pooling proxy, stay attached to a server connection another client may receive. Avoid `pg_advisory_lock(id) ... LIMIT n` in one query level; the documentation warns the lock function can run on rows the `LIMIT` later discards.

## Timeouts and connection pools

- Set `statement_timeout` and `idle_in_transaction_session_timeout` per application role (`ALTER ROLE app_rw SET ...`), with values from your latency budget. PostgreSQL 17 adds `transaction_timeout` for total transaction length.
- A transaction-pooling proxy hands a server connection to a different client after each transaction. Session state does not survive: plain `SET`, session advisory locks, `LISTEN`, temporary tables other than `ON COMMIT DROP`, `WITH HOLD` cursors and SQL-level `PREPARE`. Protocol-level named prepared statements, which most drivers send, work through PgBouncer 1.21 and later only when `max_prepared_statements` is non-zero; it defaults to 200 from 1.24 and to 0 in 1.21 to 1.23, so check the deployed version and setting before turning prepared statements off in the driver. Worse, plain `SET app.tenant_id = ...` leaks to the next client. Use `SET LOCAL` or `set_config(name, value, true)` inside the transaction.
- Size the application pool against `max_connections` across every instance and job runner, not per process.

## Row-level security

- Superusers and roles with `BYPASSRLS` always bypass policies. The table owner bypasses them unless the table has `FORCE ROW LEVEL SECURITY`. The application must connect as a role that owns nothing and has neither attribute; run migrations as a different role.
- Views run with the view owner's privileges and RLS policies by default. A view created by the migration role, which owns the tables, returns every tenant's rows to the application role while the tables themselves look protected. On PostgreSQL 15 and later create views over RLS tables `WITH (security_invoker = true)`; on 14, have them owned by a role the policies apply to. `SECURITY DEFINER` functions bypass policies the same way unless their owner is subject to them.
- Enabling RLS with no policy denies all rows, which is the right failure mode.
- Unique, primary key and foreign key checks bypass RLS, so a constraint violation can reveal that another tenant's row exists. Scope unique constraints by `tenant_id`.
- Fail closed when the tenant setting is missing. `current_setting('app.tenant_id', true)` returns NULL if the setting was never defined in the session, and can return an empty string on a pooled connection where it was defined earlier:

```sql
CREATE POLICY tenant_isolation ON invoices
  USING (tenant_id = nullif(current_setting('app.tenant_id', true), '')::bigint)
  WITH CHECK (tenant_id = nullif(current_setting('app.tenant_id', true), '')::bigint);
```

RLS is a second line of defense; application queries still filter by tenant, and tests prove a second tenant sees nothing.

## Time, sequences and plans

- `now()` and `CURRENT_TIMESTAMP` return the transaction start time, identical for every row a transaction writes. `clock_timestamp()` is wall time. A long transaction's rows carry timestamps from when it began, which is one reason timestamp cursors miss rows.
- `nextval` is never rolled back. Sequences have gaps and commit out of order; never derive "no rows missing" from contiguous ids.
- `EXPLAIN ANALYZE` executes the statement. For `INSERT`, `UPDATE` or `DELETE`, wrap it in `BEGIN; EXPLAIN (ANALYZE, BUFFERS) ...; ROLLBACK;`. Compare estimated and actual row counts; a large gap points to stale statistics (`ANALYZE table`) or correlated columns before it points to a missing index.
