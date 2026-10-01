# Jobs, leases, fencing and state machines

Read this when writing background jobs, schedulers, workers, locks, or any `status` column that changes over time.

Templates use PostgreSQL and node-postgres. `withTransaction` is the helper from idempotency.md.

## Why a lease needs a fencing token

This timeline happens in production:

1. Worker A takes a lock or lease that expires in 30 seconds and starts a payout batch.
2. A stalls for 40 seconds: a GC pause, CPU throttling in its container, a slow DNS lookup.
3. The lease expires. Worker B takes it and starts the same batch.
4. A resumes, has no idea time passed, and keeps paying.

No TTL value fixes this, because a pause can always be longer than the TTL. What fixes it is a token that increases on every acquisition and a resource that rejects writes carrying an older token. In a database that is a conditional write. A third-party API cannot check your token, so there the protection is the idempotency key sent with each call; the lease only prevents wasted work.

The same reasoning rules out `finally { await redis.del('lock:payouts') }`: after step 3, A's `del` removes B's lock. Release only if the stored value is still yours, in one atomic operation.

## Jobs table

```sql
CREATE TABLE jobs (
  id                bigint      GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  kind              text        NOT NULL,
  dedupe_key        text        NOT NULL,
  payload           jsonb       NOT NULL,
  state             text        NOT NULL DEFAULT 'queued'
                    CHECK (state IN ('queued', 'running', 'succeeded', 'dead')),
  attempts          integer     NOT NULL DEFAULT 0,
  max_attempts      integer     NOT NULL,
  run_after         timestamptz NOT NULL DEFAULT now(),
  lease_token       bigint      NOT NULL DEFAULT 0,
  leased_by         text,
  lease_expires_at  timestamptz,
  last_error        text,
  UNIQUE (kind, dedupe_key)
);

CREATE INDEX jobs_claimable_idx ON jobs (kind, run_after) WHERE state IN ('queued', 'running');
```

`dedupe_key` is the business identity of the work (payout id, invoice id plus period, schedule slot), so enqueueing the same work twice is a no-op. `lease_token` is the fencing token; it increases on every claim, including a claim that takes over an expired lease.

## Claim, renew, finish

```ts
const CLAIM_JOB = `
  UPDATE jobs
     SET state = 'running',
         attempts = attempts + 1,
         lease_token = lease_token + 1,
         leased_by = $2,
         lease_expires_at = now() + make_interval(secs => $3::double precision)
   WHERE id = (
     SELECT id FROM jobs
      WHERE kind = $1
        AND run_after <= now()
        AND attempts < max_attempts
        AND (state = 'queued' OR (state = 'running' AND lease_expires_at < now()))
      ORDER BY run_after
      LIMIT 1
      FOR UPDATE SKIP LOCKED)
  RETURNING id, payload, lease_token, attempts, dedupe_key`;

const RENEW_LEASE = `
  UPDATE jobs SET lease_expires_at = now() + make_interval(secs => $3::double precision)
   WHERE id = $1 AND lease_token = $2 AND state = 'running'`;

const COMPLETE_JOB = `
  UPDATE jobs SET state = 'succeeded', lease_expires_at = NULL
   WHERE id = $1 AND lease_token = $2 AND state = 'running'`;

const FAIL_JOB = `
  UPDATE jobs
     SET state = CASE WHEN attempts >= max_attempts THEN 'dead' ELSE 'queued' END,
         run_after = now() + make_interval(secs => $3::double precision),
         last_error = $4,
         lease_expires_at = NULL
   WHERE id = $1 AND lease_token = $2 AND state = 'running'`;
```

All lease times come from the database clock, so worker clock skew does not matter. Every statement after the claim is conditional on `lease_token`; if another worker has taken over, the statement touches zero rows.

A job whose final attempt dies with its lease still set is never claimed again (`attempts < max_attempts` fails). A sweeper moves those to `dead` and alerts:

```sql
UPDATE jobs SET state = 'dead', last_error = 'lease expired on final attempt'
 WHERE state = 'running' AND lease_expires_at < now() AND attempts >= max_attempts;
```

## Fenced writes

Business writes made by a job check the fence in the same transaction. Locking the job row also makes a concurrent takeover wait until this transaction ends.

```ts
export class LeaseLostError extends Error {}

export async function fenced<T>(
  pool: Pool,
  job: { id: string; lease_token: string },
  fn: (tx: PoolClient) => Promise<T>,
): Promise<T> {
  return withTransaction(pool, async (tx) => {
    const held = await tx.query(
      `SELECT 1 FROM jobs WHERE id = $1 AND lease_token = $2 AND state = 'running' FOR UPDATE`,
      [job.id, job.lease_token],
    );
    if (held.rowCount === 0) throw new LeaseLostError(`job ${job.id} lease lost`);
    return fn(tx);
  });
}
```

## Worker loop

```ts
type ClaimedJob = { id: string; payload: unknown; lease_token: string; attempts: number; dedupe_key: string };
type JobSignals = { leaseLost: AbortSignal; shuttingDown: AbortSignal };

export async function runWorker(
  pool: Pool,
  kind: string,
  handler: (job: ClaimedJob, signals: JobSignals) => Promise<void>,
  cfg: WorkerConfig,
  shuttingDown: AbortSignal,
): Promise<void> {
  while (!shuttingDown.aborted) {
    let job: ClaimedJob | undefined;
    try {
      job = (await pool.query<ClaimedJob>(CLAIM_JOB, [kind, cfg.workerId, cfg.leaseSeconds])).rows[0];
    } catch (err) {
      log.error({ err, kind }, 'job claim failed');
    }
    if (!job) {
      await idle(cfg.idleDelayMs, shuttingDown);
      continue;
    }

    const claimed = job;
    const leaseLost = new AbortController();
    const heartbeat = setInterval(() => {
      pool.query(RENEW_LEASE, [claimed.id, claimed.lease_token, cfg.leaseSeconds]).then(
        (r) => {
          if (r.rowCount === 0) leaseLost.abort(new LeaseLostError(`job ${claimed.id} lease lost`));
        },
        (err) => log.warn({ err, jobId: claimed.id }, 'lease renewal failed'),
      );
    }, cfg.heartbeatMs);

    try {
      await handler(claimed, { leaseLost: leaseLost.signal, shuttingDown });
      await pool.query(COMPLETE_JOB, [claimed.id, claimed.lease_token]);
    } catch (err) {
      const delaySeconds = Math.random() * Math.min(cfg.maxBackoffSeconds, cfg.baseBackoffSeconds * 2 ** (claimed.attempts - 1));
      await pool
        .query(FAIL_JOB, [claimed.id, claimed.lease_token, delaySeconds, String(err)])
        .catch((failErr) => log.error({ err: failErr, jobId: claimed.id }, 'recording job failure failed; lease expiry will retry it'));
    } finally {
      clearInterval(heartbeat);
    }
  }
}
```

`idle` is the abortable sleep from outbox-and-consumers.md. Keep `heartbeatMs` a small fraction of the lease so one missed renewal does not lose it. The handler checks `leaseLost` and `shuttingDown` between steps and stops early; anything it has already done must be safe to repeat, because the next claimant starts from the top. On SIGTERM the loop stops claiming and the current job either finishes inside the grace period or is abandoned, in which case its lease expires and another worker takes it.

## Schedules without leader election

Every replica can run the same cron trigger. Each one inserts the job for the slot and only one row survives:

```sql
INSERT INTO jobs (kind, dedupe_key, payload, max_attempts)
VALUES ('daily-payouts', $1, '{}', $2)
ON CONFLICT (kind, dedupe_key) DO NOTHING;
```

Derive `$1` from the scheduled slot (the date in the business time zone the trigger was meant for), not from the current time when the replica happens to run, or a late replica creates tomorrow's job today.

Large batches fan out: the slot job enqueues one job per item with the item's id as `dedupe_key`, so each item gets its own lease, attempt count and dead-letter state, and one bad item does not block or repeat the rest.

## Calling providers from a job

The provider idempotency key comes from the job's business identity, for example `` `payout:${payoutId}` ``, or from a key column written when the item was created. Never generate it per attempt; a fresh key on retry is the same as no key. Classify the provider's answer as in idempotency.md: only definitive answers are terminal, unknown outcomes leave the item in its in-flight state for the next attempt or the reconciler.

## Guarded state transitions

Keep the allowed transitions in one place, keyed by target state:

```ts
export type PayoutState = 'pending' | 'processing' | 'unknown' | 'paid' | 'failed' | 'cancelled';

const ALLOWED_FROM: Record<PayoutState, readonly PayoutState[]> = {
  pending: [],
  processing: ['pending', 'unknown'],
  unknown: ['processing'],
  paid: ['processing', 'unknown'],
  failed: ['processing', 'unknown'],
  cancelled: ['pending'],
};

export type TransitionResult =
  | { kind: 'applied'; version: string }
  | { kind: 'already' }
  | { kind: 'illegal'; from: PayoutState }
  | { kind: 'not_found' };

export async function transitionPayout(
  tx: PoolClient,
  payoutId: string,
  to: PayoutState,
): Promise<TransitionResult> {
  const updated = await tx.query<{ version: string }>(
    `UPDATE payouts
        SET state = $2, version = version + 1, updated_at = now()
      WHERE id = $1 AND state = ANY($3::text[])
      RETURNING version`,
    [payoutId, to, ALLOWED_FROM[to]],
  );
  const row = updated.rows[0];
  if (row) return { kind: 'applied', version: row.version };

  const current = await tx.query<{ state: PayoutState }>('SELECT state FROM payouts WHERE id = $1', [payoutId]);
  const found = current.rows[0];
  if (!found) return { kind: 'not_found' };
  if (found.state === to) return { kind: 'already' };
  return { kind: 'illegal', from: found.state };
}
```

The check and the write are one statement, so two workers cannot both move a payout out of `pending`. Whatever the transition triggers (the transfer, the email, the outbox event) runs only for the caller that got `applied`, or is itself idempotent. `already` is how a retried job recognises that an earlier attempt finished. Write an audit row and any outbox event in the same transaction as the transition.

A CHECK constraint on the column catches invalid state names, not invalid transitions. When several code paths write the column (which is itself worth fixing), a trigger that rejects transitions not in the table is a reasonable backstop.

## Stuck states

Every intermediate state (`processing`, `unknown`, `payment_pending`) needs something that notices when a row stays in it too long. A periodic query finds them and enqueues a reconcile job per row, deduplicated on the row id and version:

```sql
SELECT id, version FROM payouts
 WHERE state IN ('processing', 'unknown')
   AND updated_at < now() - make_interval(secs => $1::double precision)
 ORDER BY updated_at
 LIMIT $2;
```

The reconcile job asks the provider what happened, then transitions to `paid` or `failed`, or leaves the row and alerts once it passes the age limit. It never re-sends a transfer without a lookup or a provider idempotency key.

## Crash walk-through

Walk each flow like this before calling it done. For a payout job:

| Process dies | State left behind | What recovers it |
| --- | --- | --- |
| After claim, before `pending` to `processing` | job `running`, payout `pending` | Lease expires, job reclaimed, runs from the top |
| After `processing`, before the provider call | payout `processing` | Reclaim; the handler reads the payout, sees `processing`, and resumes by calling the provider with the same key |
| After the provider call, before recording | payout `processing`, transfer exists at provider | Reclaim, same key, provider returns the existing transfer, handler records `paid` |
| After `paid`, before `COMPLETE_JOB` | payout `paid`, job `running` | Reclaim, transition returns `already`, handler completes the job |
| Final attempt dies | job `running` with expired lease | Sweeper marks job `dead` and alerts; stuck-state query reconciles the payout |
