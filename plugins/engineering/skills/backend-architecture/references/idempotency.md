# Idempotency keys and unknown outcomes

Read this when building or reviewing an endpoint or job that creates something, charges, transfers, or calls a provider that must not act twice.

Templates use PostgreSQL and node-postgres (`pg`). The shape carries over to any stack: claim the key durably before the work, bind it to the caller and the request, pass a stable key downstream, and finish the key in the same transaction as the local effect.

## What the key protects and what it does not

A client idempotency key makes one logical request safe to repeat. It does not stop two different keys from paying the same order; that is the job of the order's state guard. Always pair the key with a guarded transition on the business row (see jobs-and-state-machines.md), and bind the business row to the key that started it, so a second key for the same order gets a 409 instead of a second charge.

## Table

```sql
CREATE TABLE idempotency_keys (
  scope            text        NOT NULL,
  idempotency_key  text        NOT NULL,
  request_hash     bytea       NOT NULL,
  state            text        NOT NULL CHECK (state IN ('in_progress', 'completed')),
  attempt          integer     NOT NULL DEFAULT 1,
  downstream_key   uuid        NOT NULL DEFAULT gen_random_uuid(),
  locked_until     timestamptz NOT NULL,
  response_status  integer,
  response_body    jsonb,
  created_at       timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (scope, idempotency_key),
  CHECK (state <> 'completed' OR response_status IS NOT NULL)
);

CREATE INDEX idempotency_keys_open_idx ON idempotency_keys (locked_until) WHERE state = 'in_progress';
CREATE INDEX idempotency_keys_created_idx ON idempotency_keys (created_at);
```

- `scope` is the authenticated principal (account, tenant or API key id) taken from the verified credential. A key from one caller never matches another caller's row.
- `request_hash` binds the key to one request. A known key with a different hash is a client bug or an attack, answered with 422.
- `attempt` is a fencing token. A takeover after an abandoned attempt increments it, and completion is conditional on it, so a slow original attempt cannot overwrite the result.
- `downstream_key` is the idempotency key sent to the provider. It is created with the row and never changes, so every attempt, takeover and reconciliation sends the same one.
- `locked_until` is a lease so a crashed attempt does not block the key forever.

## Shared helpers

```ts
import { createHash } from 'node:crypto';
import type { Pool, PoolClient } from 'pg';

export async function withTransaction<T>(pool: Pool, fn: (tx: PoolClient) => Promise<T>): Promise<T> {
  const tx = await pool.connect();
  let rollbackFailed = false;
  try {
    await tx.query('BEGIN');
    const result = await fn(tx);
    await tx.query('COMMIT');
    return result;
  } catch (err) {
    await tx.query('ROLLBACK').catch(() => {
      rollbackFailed = true;
    });
    throw err;
  } finally {
    tx.release(rollbackFailed);
  }
}

function canonicalize(value: unknown): unknown {
  if (Array.isArray(value)) return value.map(canonicalize);
  if (value !== null && typeof value === 'object') {
    return Object.fromEntries(
      Object.entries(value)
        .sort(([a], [b]) => (a < b ? -1 : a > b ? 1 : 0))
        .map(([k, v]) => [k, canonicalize(v)]),
    );
  }
  return value;
}

export function fingerprint(operation: string, input: unknown): Buffer {
  return createHash('sha256').update(JSON.stringify([operation, canonicalize(input)])).digest();
}
```

Hash the parsed, schema-validated input (path parameters plus body, unknown keys already rejected), not raw bytes and not headers. That way whitespace and key order do not matter, and fields the handler never reads cannot make two different requests look equal. The rollback failure is not swallowed: it destroys the connection instead of returning a broken one to the pool, and the original error is rethrown.

A `COMMIT` that fails with a connection error is itself an unknown outcome. The next attempt sees either the completed key or the `in_progress` row and behaves correctly in both cases.

## Claiming a key

```ts
export type Claim =
  | { kind: 'owned'; attempt: number; downstreamKey: string }
  | { kind: 'replay'; status: number; body: unknown }
  | { kind: 'mismatch' }
  | { kind: 'busy'; retryAfterSeconds: number };

type KeyRow = {
  state: 'in_progress' | 'completed';
  request_hash: Buffer;
  response_status: number | null;
  response_body: unknown;
  lease_seconds_left: number;
};

export async function claimKey(
  pool: Pool,
  scope: string,
  key: string,
  requestHash: Buffer,
  leaseSeconds: number,
): Promise<Claim> {
  const inserted = await pool.query<{ attempt: number; downstream_key: string }>(
    `INSERT INTO idempotency_keys (scope, idempotency_key, request_hash, state, locked_until)
     VALUES ($1, $2, $3, 'in_progress', now() + make_interval(secs => $4::double precision))
     ON CONFLICT (scope, idempotency_key) DO NOTHING
     RETURNING attempt, downstream_key`,
    [scope, key, requestHash, leaseSeconds],
  );
  const fresh = inserted.rows[0];
  if (fresh) return { kind: 'owned', attempt: fresh.attempt, downstreamKey: fresh.downstream_key };

  const existing = await pool.query<KeyRow>(
    `SELECT state, request_hash, response_status, response_body,
            GREATEST(0, EXTRACT(EPOCH FROM locked_until - now()))::float8 AS lease_seconds_left
       FROM idempotency_keys
      WHERE scope = $1 AND idempotency_key = $2`,
    [scope, key],
  );
  const row = existing.rows[0];
  if (!row) return { kind: 'busy', retryAfterSeconds: 0 };
  if (!row.request_hash.equals(requestHash)) return { kind: 'mismatch' };
  if (row.state === 'completed' && row.response_status !== null) {
    return { kind: 'replay', status: row.response_status, body: row.response_body };
  }
  if (row.lease_seconds_left > 0) {
    return { kind: 'busy', retryAfterSeconds: Math.ceil(row.lease_seconds_left) };
  }

  const taken = await pool.query<{ attempt: number; downstream_key: string }>(
    `UPDATE idempotency_keys
        SET attempt = attempt + 1,
            locked_until = now() + make_interval(secs => $3::double precision)
      WHERE scope = $1 AND idempotency_key = $2
        AND state = 'in_progress' AND locked_until <= now()
      RETURNING attempt, downstream_key`,
    [scope, key, leaseSeconds],
  );
  const takeover = taken.rows[0];
  return takeover
    ? { kind: 'owned', attempt: takeover.attempt, downstreamKey: takeover.downstream_key }
    : { kind: 'busy', retryAfterSeconds: 0 };
}

export class LeaseLostError extends Error {}

export async function completeKey(
  tx: PoolClient,
  scope: string,
  key: string,
  attempt: number,
  status: number,
  body: unknown,
): Promise<void> {
  const result = await tx.query(
    `UPDATE idempotency_keys
        SET state = 'completed', response_status = $4, response_body = $5, locked_until = now()
      WHERE scope = $1 AND idempotency_key = $2 AND attempt = $3 AND state = 'in_progress'`,
    [scope, key, attempt, status, JSON.stringify(body)],
  );
  if (result.rowCount !== 1) throw new LeaseLostError(`idempotency key ${scope}/${key} was taken over`);
}
```

The claim is its own committed statement, before any work. The row vanishing between the two statements means retention pruned it; the client retries and gets a fresh claim. A takeover is only safe because the provider sees the same `downstream_key`; if the downstream has no idempotency support, a takeover must reconcile (look the operation up) before calling again.

## Handler: charge an order

```ts
const IDEMPOTENCY_KEY_PATTERN = /^[A-Za-z0-9_-]{16,128}$/;
const PAY_OPERATION = 'POST /v1/orders/:orderId/payments';

app.post('/v1/orders/:orderId/payments', async (req, res) => {
  const accountId = authenticatedAccountId(req);
  const key = req.get('Idempotency-Key');
  if (key === undefined || !IDEMPOTENCY_KEY_PATTERN.test(key)) {
    return sendProblem(res, 400, 'invalid-idempotency-key');
  }
  const input = parsePaymentRequest({ ...req.body, orderId: req.params.orderId });
  const claim = await claimKey(pool, accountId, key, fingerprint(PAY_OPERATION, input), config.idempotencyLeaseSeconds);

  if (claim.kind === 'replay') return res.status(claim.status).json(claim.body);
  if (claim.kind === 'mismatch') return sendProblem(res, 422, 'idempotency-key-reused');
  if (claim.kind === 'busy') {
    res.set('Retry-After', String(claim.retryAfterSeconds));
    return sendProblem(res, 409, 'request-in-progress');
  }

  const order = await beginPayment(pool, accountId, input.orderId, claim.downstreamKey);
  if (!order) {
    const body = problemBody(409, 'order-not-payable');
    await withTransaction(pool, (tx) => completeKey(tx, accountId, key, claim.attempt, 409, body));
    return res.status(409).json(body);
  }

  let outcome: ChargeOutcome;
  try {
    const charge = await psp.createCharge(
      { amountMinor: order.amountMinor, currency: order.currency, reference: order.id },
      { idempotencyKey: claim.downstreamKey, signal: AbortSignal.timeout(config.pspTimeoutMs) },
    );
    outcome = { kind: 'succeeded', chargeId: charge.id };
  } catch (err) {
    if (!isDefinitiveDecline(err)) {
      res.set('Retry-After', String(config.idempotencyLeaseSeconds));
      return sendProblem(res, 503, 'payment-outcome-unknown');
    }
    outcome = { kind: 'declined', reason: declineReason(err) };
  }

  const status = outcome.kind === 'succeeded' ? 201 : 402;
  const body = await withTransaction(pool, async (tx) => {
    const result = await finishPayment(tx, order.id, claim.downstreamKey, outcome);
    await completeKey(tx, accountId, key, claim.attempt, status, result);
    return result;
  });
  return res.status(status).json(body);
});
```

`beginPayment` is the guarded transition that makes a second key harmless:

```sql
UPDATE orders
   SET status = 'payment_pending', payment_key = $3, version = version + 1
 WHERE id = $2 AND account_id = $1
   AND (status IN ('pending', 'payment_failed')
        OR (status = 'payment_pending' AND payment_key = $3))
RETURNING id, amount_minor, currency;
```

The amount and currency come from the order row, never from the request. `payment_failed` is allowed so the customer can try again with a new key after a decline. The second branch lets a takeover of the same key re-enter; a different key finds `payment_pending` with another `payment_key` and gets zero rows. `finishPayment` is the same pattern from `payment_pending` (with the matching `payment_key`) to `paid` or `payment_failed`.

On an unknown outcome the key stays `in_progress`, the order stays `payment_pending`, and the lease stays in force, which gives a provider still working on the timed-out request time to finish. A retry inside the lease gets 409. The client's retry after `Retry-After` takes the lease over and calls the provider again with the same `downstream_key`. After a timeout or a lost response that returns the original result instead of charging twice; after an error the provider stored under the key it returns the same error (next section). If the client never returns, the reconciler below finishes the row.

## Classifying the provider's answer

Only a definitive answer ends the attempt. A card decline, a validation rejection or an explicit "not found" is definitive. A timeout, a connection reset after the request was written, a 500, 502 or 504, and a response body you cannot parse are all unknown. A 429 or 503 is not processed only where the provider documents it that way. Put this classification in one function per provider and test it against recorded responses.

A keyed retry recovers a lost response; it does not get past an error the provider stored under the key. Stripe saves the status and body of every request that began executing, 500s included, and replays them for the same key, so after a Stripe 500 every takeover sees the same 500 and the client loops on 503. Treat that as a reconciliation case: enqueue the reconcile for the key at once and resolve it by lookup (your own reference in the object's `metadata`, or the webhook Stripe sends for objects its own incident reconciliation creates). Never retry it under a new key, which Stripe warns may duplicate the side effect.

## Reconciler

Some attempts end unknown and the client never retries. A scheduled job resolves them:

```sql
SELECT scope, idempotency_key, attempt, downstream_key
  FROM idempotency_keys
 WHERE state = 'in_progress' AND locked_until < now() - make_interval(secs => $1::double precision)
 ORDER BY locked_until
 LIMIT $2;
```

For each row, take the lease with the same conditional update as a takeover, look the operation up at the provider (by the downstream key, or by your own reference stored on the provider object), then finish the business row and complete the key in one transaction. A definitive "never happened" moves the business row back to a retryable state or to failed; a lookup that still cannot decide leaves the row alone, increments a counter and alerts once it is older than your limit.

Providers keep idempotency keys for a limited time. Stripe may prune keys once they are at least 24 hours old, after which the same key creates a new request. The reconciler has to run well inside the provider's window, and anything older than it must be resolved by lookup, never by blindly calling again.

## Retention

Delete completed keys in batches once they are older than the window you promise clients for retries, and make that window longer than any client's retry horizon, including mobile apps that retry after coming back online. Never delete `in_progress` rows by age alone; they are unresolved operations.
