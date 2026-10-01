# Outbox, relay and consumers

Read this when a state change has to produce an event or message, or when writing a consumer that applies events to its own state.

Templates use PostgreSQL and node-postgres. `withTransaction` is the helper from idempotency.md. The broker client is whatever the codebase uses; the relay only needs "publish with a message id and a partition key, reject on failure".

## Outbox table

```sql
CREATE TABLE outbox (
  id                 bigint      GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  event_id           uuid        NOT NULL UNIQUE DEFAULT gen_random_uuid(),
  aggregate_type     text        NOT NULL,
  aggregate_id       text        NOT NULL,
  aggregate_version  bigint      NOT NULL,
  event_type         text        NOT NULL,
  payload            jsonb       NOT NULL,
  created_at         timestamptz NOT NULL DEFAULT now(),
  claimed_until      timestamptz,
  published_at       timestamptz
);

CREATE INDEX outbox_unpublished_idx ON outbox (id) WHERE published_at IS NULL;
```

`event_id` is what consumers dedupe on. `aggregate_version` is the owner's version of the entity after the change; consumers use it for ordering. `id` is only a rough processing order and is never used as a cursor.

## Writing an event

The event row and the state change commit together or not at all.

```ts
await withTransaction(pool, async (tx) => {
  const updated = await tx.query<{ version: string }>(
    `UPDATE orders SET status = 'confirmed', version = version + 1
      WHERE id = $1 AND status = 'pending'
      RETURNING version`,
    [orderId],
  );
  const row = updated.rows[0];
  if (!row) throw new ConflictError('order is not pending');

  await tx.query(
    `INSERT INTO outbox (aggregate_type, aggregate_id, aggregate_version, event_type, payload)
     VALUES ('order', $1, $2, 'order.confirmed', $3)`,
    [orderId, row.version, JSON.stringify({ orderId, version: row.version })],
  );
});
```

node-postgres returns `bigint` columns as strings; keep versions as strings or `BigInt` in application code rather than converting to `number`.

## Relay

The relay claims unpublished rows with a short lease, commits the claim, publishes outside any transaction, then marks each row published. `FOR UPDATE SKIP LOCKED` lets several relay instances share the table; the PostgreSQL documentation describes it as unsuitable for general work because it gives an inconsistent view, and suitable for queue-like tables with several consumers, which is this case. Rows from transactions that have not committed are invisible and get picked up on a later pass, which is exactly what a high-water-mark cursor gets wrong.

```ts
import { setTimeout as delay } from 'node:timers/promises';

type OutboxRow = {
  id: string;
  event_id: string;
  aggregate_type: string;
  aggregate_id: string;
  aggregate_version: string;
  event_type: string;
  payload: unknown;
};

const CLAIM_BATCH = `
  WITH claimed AS (
    UPDATE outbox
       SET claimed_until = now() + make_interval(secs => $2::double precision)
     WHERE id IN (
       SELECT id FROM outbox
        WHERE published_at IS NULL
          AND (claimed_until IS NULL OR claimed_until < now())
        ORDER BY id
        LIMIT $1
        FOR UPDATE SKIP LOCKED)
    RETURNING id, event_id, aggregate_type, aggregate_id, aggregate_version, event_type, payload)
  SELECT * FROM claimed ORDER BY id`;

export async function relayOnce(pool: Pool, broker: Broker, cfg: RelayConfig): Promise<number> {
  const { rows } = await pool.query<OutboxRow>(CLAIM_BATCH, [cfg.batchSize, cfg.claimSeconds]);
  let published = 0;
  for (const row of rows) {
    await broker.publish(`${row.aggregate_type}.events`, row.payload, {
      messageId: row.event_id,
      partitionKey: row.aggregate_id,
      headers: { eventType: row.event_type, aggregateVersion: row.aggregate_version },
      signal: AbortSignal.timeout(cfg.publishTimeoutMs),
    });
    await pool.query('UPDATE outbox SET published_at = now() WHERE id = $1', [row.id]);
    published += 1;
  }
  return published;
}

export async function idle(ms: number, stop: AbortSignal): Promise<void> {
  try {
    await delay(ms, undefined, { signal: stop });
  } catch (err) {
    if (!stop.aborted) throw err;
  }
}

export async function runRelay(pool: Pool, broker: Broker, cfg: RelayConfig, stop: AbortSignal): Promise<void> {
  while (!stop.aborted) {
    let published = 0;
    try {
      published = await relayOnce(pool, broker, cfg);
    } catch (err) {
      log.error({ err }, 'outbox relay batch failed');
    }
    if (published === 0) await idle(cfg.idleDelayMs, stop);
  }
}
```

Why it is shaped this way:

- Publish first, mark second. A crash between the two republishes the row, which consumers absorb by deduping on `event_id`. Marking first would lose the event on a crash, which is the failure the outbox exists to prevent.
- A failed publish aborts the rest of the batch. Continuing would publish later events for the same aggregate ahead of the failed one. The failed row's claim expires and it is retried.
- The loop runs sequentially, so a slow pass never overlaps the next one, unlike a `setInterval` poller.
- Several relay instances can each claim different rows, so two events for one aggregate can be published out of order by different instances. Consumers must order by `aggregate_version` regardless. If a downstream truly cannot cope, run one active relay behind a lease (jobs-and-state-machines.md) or switch to CDC.
- Published rows are deleted in batches by a separate job once they are older than the replay window you want. Watch the count and age of unpublished rows; a growing age means the relay is stuck, and that is the alert.

Logical decoding (CDC, for example Debezium reading the write-ahead log) replaces the polling relay when volume is high or strict commit order matters. The outbox table and consumer rules stay the same.

## Consumer: dedupe inbox

```sql
CREATE TABLE processed_messages (
  consumer      text        NOT NULL,
  message_id    uuid        NOT NULL,
  processed_at  timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (consumer, message_id)
);
```

A delta event (points earned, stock reserved) is not naturally idempotent, so it goes through the inbox and a strict version check in one transaction:

```ts
const CONSUMER = 'loyalty-points';

export async function applyPointsEarned(pool: Pool, msg: PointsEarned): Promise<'applied' | 'duplicate'> {
  return withTransaction(pool, async (tx) => {
    const fresh = await tx.query(
      `INSERT INTO processed_messages (consumer, message_id) VALUES ($1, $2)
       ON CONFLICT DO NOTHING`,
      [CONSUMER, msg.eventId],
    );
    if (fresh.rowCount === 0) return 'duplicate';

    await tx.query(
      `INSERT INTO loyalty_accounts (account_id, points, version) VALUES ($1, 0, 0)
       ON CONFLICT (account_id) DO NOTHING`,
      [msg.accountId],
    );
    const applied = await tx.query(
      `UPDATE loyalty_accounts
          SET points = points + $2, version = $3::bigint
        WHERE account_id = $1 AND version = $3::bigint - 1`,
      [msg.accountId, msg.points, msg.version],
    );
    if (applied.rowCount === 0) throw new OutOfOrderError(msg.accountId, msg.version);
    return 'applied';
  });
}
```

`OutOfOrderError` rolls back the inbox row too, so the message is redelivered later and applied once its predecessor has arrived. After the configured number of attempts it goes to the dead-letter queue, which is where a genuinely lost predecessor surfaces. The account row is created at version 0 on first sight, so the first event for a new account applies instead of being dropped.

The message is acknowledged only after `withTransaction` resolves. Acknowledging earlier, or running the broker client in auto-ack mode, loses the message if the process dies mid-handler.

## Consumer: snapshot projection

When each event carries the full new state, the version gate alone makes the handler idempotent and order-tolerant, and no inbox is needed:

```sql
INSERT INTO order_view (order_id, status, total_minor, version)
VALUES ($1, $2, $3, $4)
ON CONFLICT (order_id) DO UPDATE
   SET status = EXCLUDED.status, total_minor = EXCLUDED.total_minor, version = EXCLUDED.version
 WHERE order_view.version < EXCLUDED.version;
```

An older event arriving late changes nothing; an event for an order the view has never seen creates it.

## Side effects outside the consumer's database

The inbox cannot make an email or a provider call atomic with the dedupe row. Either write an outbox row in the consumer's transaction and let a relay do the call, or pass the `event_id` (or a key derived from it) as the downstream idempotency key so a repeat is absorbed there.

## Poison messages

Cap delivery attempts using the broker's delivery count, then dead-letter the message with the last error, the consumer name and the attempt count. Alert on dead-letter depth. Keep a replay tool that re-publishes a dead-lettered message with its original `event_id`, so replays are deduplicated like any other redelivery.
