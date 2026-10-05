# Database write and migration patterns

Read this when writing a guarded status transition, idempotent insert, keyset pagination, expand-contract migration or batched backfill. PostgreSQL examples use node-postgres (`pg`); D1 examples use the Workers binding. Adapt the driver calls, keep the SQL shape.

## Guarded status transition

The state machine lives in the `WHERE` clause. The allowed statuses live in a `CHECK` so a bug elsewhere cannot write an unknown one.

```sql
ALTER TABLE orders
  ADD CONSTRAINT orders_status_valid
  CHECK (status IN ('pending', 'paid', 'cancelled', 'refunded'));
```

```ts
import type { Pool } from "pg";

type MarkPaidResult =
  | { kind: "paid"; orderId: string }
  | { kind: "not_found" }
  | { kind: "conflict"; currentStatus: string };

export async function markOrderPaid(
  db: Pool,
  tenantId: string,
  orderId: string,
  paymentId: string,
): Promise<MarkPaidResult> {
  const updated = await db.query(
    `UPDATE orders
     SET status = 'paid', paid_at = now(), payment_id = $3
     WHERE id = $1 AND tenant_id = $2 AND status = 'pending'
     RETURNING id`,
    [orderId, tenantId, paymentId],
  );
  if (updated.rowCount === 1) return { kind: "paid", orderId };

  const current = await db.query(
    `SELECT status, payment_id FROM orders WHERE id = $1 AND tenant_id = $2`,
    [orderId, tenantId],
  );
  if (current.rowCount === 0) return { kind: "not_found" };
  const row = current.rows[0];
  // A replay of the transition that already won is success, not a conflict.
  if (row.status === "paid" && row.payment_id === paymentId) return { kind: "paid", orderId };
  return { kind: "conflict", currentStatus: row.status };
}
```

The follow-up `SELECT` only classifies the outcome; it never decides whether to write. On D1 the same `UPDATE` works with `?` placeholders; read `result.meta.changes` instead of `rowCount`.

## Idempotent insert

Used for payment requests, webhook deliveries, signups keyed by external id, anything a client may retry.

```sql
CREATE TABLE payment_requests (
  tenant_id       bigint      NOT NULL,
  idempotency_key text        NOT NULL,
  request_hash    bytea       NOT NULL,
  status          text        NOT NULL CHECK (status IN ('pending', 'succeeded', 'failed')),
  response        jsonb,
  created_at      timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (tenant_id, idempotency_key)
);
```

```sql
INSERT INTO payment_requests (tenant_id, idempotency_key, request_hash, status)
VALUES ($1, $2, $3, 'pending')
ON CONFLICT (tenant_id, idempotency_key) DO NOTHING
RETURNING idempotency_key;
```

- A row returned: this request owns the operation. Do the side effect outside the transaction, then record the outcome with `UPDATE ... WHERE status = 'pending'`.
- No row returned: an earlier attempt exists. Load it. Different `request_hash`: reject (the client reused a key for a different request). `pending`: answer "in progress" and let the client retry later, or reconcile if it is older than the operation's timeout. Terminal: return the stored `response`.

`request_hash` is computed from the canonicalized request body plus the authenticated principal, so a different user cannot replay someone else's key. On PostgreSQL, `ON CONFLICT DO NOTHING` waits for an in-flight conflicting insert to commit or abort before deciding, so under Read Committed the follow-up `SELECT` sees the winner. SQLite and D1 accept the same statement (`BLOB` for the hash, `TEXT` or `INTEGER` epoch for the time).

If the key comes from a partial unique index (for example `WHERE deleted_at IS NULL`), the `ON CONFLICT` target must repeat the index predicate, or PostgreSQL cannot infer the arbiter index.

## Keyset pagination

```sql
CREATE INDEX CONCURRENTLY orders_tenant_created_id_idx
  ON orders (tenant_id, created_at DESC, id DESC);

-- first page
SELECT id, created_at, total_minor, created_at::text AS created_at_key
FROM orders
WHERE tenant_id = $1
ORDER BY created_at DESC, id DESC
LIMIT $2;

-- next page: strictly after the last row of the previous page
SELECT id, created_at, total_minor, created_at::text AS created_at_key
FROM orders
WHERE tenant_id = $1
  AND (created_at, id) < ($3::timestamptz, $4::uuid)
ORDER BY created_at DESC, id DESC
LIMIT $2;
```

Bind `$2` as the page size plus one; if the extra row comes back there is a next page, and the cursor is built from the last row you return. No `COUNT` needed.

```ts
const UUID_PATTERN = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

export type OrderCursor = { createdAt: string; id: string };

export function encodeCursor(cursor: OrderCursor): string {
  return Buffer.from(JSON.stringify([cursor.createdAt, cursor.id]), "utf8").toString("base64url");
}

export function decodeCursor(raw: string, maxLength: number): OrderCursor {
  if (raw.length > maxLength) throw new BadRequestError("invalid cursor");
  let parsed: unknown;
  try {
    parsed = JSON.parse(Buffer.from(raw, "base64url").toString("utf8"));
  } catch {
    throw new BadRequestError("invalid cursor");
  }
  if (
    !Array.isArray(parsed) ||
    parsed.length !== 2 ||
    typeof parsed[0] !== "string" ||
    typeof parsed[1] !== "string" ||
    Number.isNaN(Date.parse(parsed[0])) ||
    !UUID_PATTERN.test(parsed[1])
  ) {
    throw new BadRequestError("invalid cursor");
  }
  return { createdAt: parsed[0], id: parsed[1] };
}
```

Details that break it in practice:

- The tiebreaker (`id`) must be unique and part of both `ORDER BY` and the row comparison, or rows sharing a timestamp are skipped or repeated at page edges.
- PostgreSQL timestamps have microsecond resolution and JavaScript `Date` has milliseconds. A cursor that round-trips `created_at` through `Date` truncates it, and the next page repeats or skips rows. The cursor carries `created_at_key`, the database's own text form, bound back as `timestamptz`.
- The cursor is client input. It only says where to resume: reject malformed values with a 400, and never let it carry `tenant_id`, filters or the page size; those come from the session and the validated query on every page.
- Clamp the page size to a maximum from configuration.
- Every filter the endpoint accepts either appears as a leading equality column of the index or is cheap to apply to the rows the index returns. Confirm with `EXPLAIN`.

SQLite (3.15+) and D1 support row values, and SQLite documents `(a, b) > (?, ?) ORDER BY a, b LIMIT ?` as the efficient replacement for `OFFSET` when a matching index exists.

## Expand and contract

Example: replace `users.email` (mixed case, not unique) with `users.email_normalized` (unique, not null). Each numbered step is a separate deploy; the application must work at every step with both the previous and the next schema.

1. **Expand schema.** `ALTER TABLE users ADD COLUMN email_normalized text;` (nullable, no default, instant apart from the lock; set `lock_timeout`).
2. **Dual-write.** Release code that writes `email_normalized` on every insert and every email change, and still reads `email`. A trigger is an alternative when several applications write the table.
3. **Backfill** existing rows with the batched job below. Start it only after step 2 is fully rolled out, or rows written by old instances in between are missed.
4. **Enforce.** `CREATE UNIQUE INDEX CONCURRENTLY users_email_normalized_key ON users (email_normalized);` then the NOT NULL sequence from `postgresql.md` (`CHECK ... NOT VALID`, `VALIDATE`, `SET NOT NULL`, drop the check). If the unique build fails on duplicates, the data needs a decision, not a retry: drop the invalid index and resolve the duplicates first.
5. **Switch reads** to `email_normalized`. Keep dual-writing so rolling back this release is safe.
6. **Stop writing** `email`. For ORMs that cache or enumerate columns, mark the column ignored in this release (Rails `ignored_columns`, or remove it from the model) so no running code selects it.
7. **Contract.** After confirming nothing reads the old column (query logs or `pg_stat_statements`, other services, reports, ETL), `ALTER TABLE users DROP COLUMN email;` in a later release.

Rollback at each step is the previous release, because the schema is always a superset of what that release needs. Renames follow the same sequence: a column rename is add, dual-write, backfill, switch, drop, never `RENAME COLUMN` under running code.

## Batched backfill

Walk the primary key in fixed-size ranges rather than looping on `WHERE new_col IS NULL LIMIT n`, which rescans already-finished rows on every iteration once the matching rows get sparse.

```sql
CREATE TABLE IF NOT EXISTS backfill_progress (
  name    text PRIMARY KEY,
  last_id bigint NOT NULL
);
```

```ts
interface BackfillConfig {
  batchSize: number;
  pauseMs: number;
}

export async function backfillEmailNormalized(pool: Pool, cfg: BackfillConfig): Promise<void> {
  const name = "users.email_normalized";
  await pool.query(
    `INSERT INTO backfill_progress (name, last_id) VALUES ($1, 0) ON CONFLICT (name) DO NOTHING`,
    [name],
  );

  for (;;) {
    const client = await pool.connect();
    let lastId: string | null = null;
    try {
      await client.query("BEGIN");
      const progress = await client.query(
        `SELECT last_id FROM backfill_progress WHERE name = $1 FOR UPDATE`,
        [name],
      );
      const batch = await client.query(
        `WITH batch AS (
           SELECT id FROM users WHERE id > $1 ORDER BY id LIMIT $2
         ), updated AS (
           UPDATE users u
           SET email_normalized = lower(u.email)
           FROM batch b
           WHERE u.id = b.id AND u.email_normalized IS NULL
           RETURNING u.id
         )
         SELECT (SELECT max(id) FROM batch) AS last_id`,
        [progress.rows[0].last_id, cfg.batchSize],
      );
      lastId = batch.rows[0].last_id;
      if (lastId !== null) {
        await client.query(`UPDATE backfill_progress SET last_id = $2 WHERE name = $1`, [name, lastId]);
      }
      await client.query("COMMIT");
    } catch (err) {
      const rollbackError = await client.query("ROLLBACK").then(
        () => undefined,
        (e: unknown) => (e instanceof Error ? e : new Error(String(e))),
      );
      // Passing an error destroys the connection instead of returning a broken one to the pool.
      client.release(rollbackError);
      throw err;
    }
    client.release();
    if (lastId === null) return;
    await new Promise((resolve) => setTimeout(resolve, cfg.pauseMs));
  }
}
```

Why it is shaped this way:

- The cursor is committed in the same transaction as the batch, so a crash resumes exactly where it stopped and never double-applies.
- `email_normalized IS NULL` makes a batch a no-op when rerun, and leaves rows that dual-write already filled untouched.
- `FOR UPDATE` on the progress row stops two copies of the job from working the same range.
- Batch size and pause come from configuration. Size batches so one takes well under the application's lock wait tolerance, and watch replica lag while it runs; raise the pause if lag grows.
- `bigint` ids come back from node-postgres as strings; passing them straight back as parameters avoids precision loss.

On SQLite and D1, drive the same loop with `UPDATE users SET email_normalized = lower(email) WHERE id IN (SELECT id FROM users WHERE id > ? ORDER BY id LIMIT ?) AND email_normalized IS NULL`, followed in the same D1 `batch()` by the progress update. Keep each batch well inside D1's per-query time limit (see `sqlite-d1.md`).
