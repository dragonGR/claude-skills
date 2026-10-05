---
name: backend-architecture
description: Backend flows that must survive retries, races and crashes: idempotency keys, unknown outcomes, timeouts, outbox and queue consumers, job leases, state machines, caching, webhooks, API changes, graceful shutdown. Load it before writing or reviewing code that moves money or state across a network, queue or job, and when data goes missing or doubles.
license: MIT
metadata:
  author: Alex Tsanis
---

# Backend architecture

The job is keeping durable state correct when requests repeat, race, time out and die halfway, and being able to say afterwards what happened. Most incidents in this area come from a short list of patterns that look harmless in a diff: a commit followed by a publish, a retry around a charge, a `SELECT` followed by an `UPDATE`. Learn to see those first. Microservices, CQRS, event sourcing and sagas answer specific problems, so name the problem before reaching for one.

## Ground rules

- Every piece of durable state has one owner that enforces its invariants. Everything else goes through the owner's interface. Two services writing one table is one service with a network in the middle.
- A call that leaves the process has three outcomes: succeeded, failed, unknown. Timeouts, connection resets, 500/502/504 and a crash after sending are unknown.
- Exactly-once delivery does not exist. What you can build is at-least-once delivery plus effects that are idempotent where they are applied. Kafka transactions and SQS FIFO deduplication stop at the broker; they do not cover your database write or the provider call.
- Identity, ownership, prices and amounts come from the owner's state, never from the request body. security-engineering covers authorization in depth.
- Before calling something a bug, trace who can reach it and check for a later guard: a unique constraint, a conditional update, a version check, a provider-side idempotency key, a reconciler. Report it only if it survives that second look.

## Failure catalogue

Each entry: what it looks like in code, why it breaks, what to do instead. Templates live in the reference files.

### Side effects and unknown outcomes

**Dual write.** `await orders.save(order); await bus.publish('order.created', order)`, or a DB write followed by an HTTP call to a service that has to stay in step, or the reverse order. A crash, deploy or broker timeout between the two leaves the database and the rest of the world disagreeing, silently and permanently: orders nobody fulfils, emails about rows that rolled back. Wrapping both in a transaction changes nothing because the publish is not part of it. Write the event into an `outbox` table in the same transaction as the state change and let a relay publish it afterwards.

**Blind retry of a non-idempotent side effect.** A retry loop, an HTTP client retry option, a mesh retry policy or a queue redelivery around `createCharge`, `transfer`, `sendSms`. The first attempt timed out after the provider did the work; the retry does it again. Send an idempotency key the provider honors, generated and stored before the first attempt and identical on every attempt. Without a key, retry only when the error proves the request never reached the server (connection refused, DNS failure). Provider keys expire: Stripe may prune keys once they are at least 24 hours old and rejects a reused key whose parameters differ, so reconciliation of an unknown outcome has to happen inside the retention window or look the operation up by your own reference.

**Idempotency key stored after the side effect.** `SELECT` the key, not found, do the work, `INSERT` the key. Two concurrent duplicates (double click, a client retrying after its own timeout) both pass the `SELECT` and both do the work, and a crash after the work leaves no record at all. Insert the key first with state `in_progress` under a unique constraint and commit it; whoever loses the insert gets the stored response or a 409. Store the final response in the same transaction as the local effect.

**Idempotency key not bound to caller and request.** The primary key is the client's key alone, and a replay returns the stored response without comparing requests. Caller B presenting caller A's key receives A's response body; a client bug reusing a key for a different payment is told the payment succeeded. Key the table on `(scope, key)` where scope is the authenticated principal or tenant, store a hash of the canonical validated request, and answer 422 when a known key arrives with a different fingerprint.

**Timeout recorded as failure.** `catch (err) { await markFailed(id) }` around a transfer, after which a user or job retries "failed" items. Use a distinct state for unknown outcomes. A reconciler asks the provider (by idempotency key or your reference) what happened and moves the row to succeeded or failed. Only a definitive answer is terminal.

**Crash between steps.** A handler charges, then reserves stock, then sends an email, with progress held only in memory. Kill it between any two lines and nothing knows where to resume. Persist progress as a state row before each external step, make each step idempotent, and run a sweeper that finds rows stuck in an intermediate state past a deadline and resumes or compensates them. If you cannot say what happens on `kill -9` at each line, the flow is not finished.

### Concurrent writes and state machines

Lost updates, check-then-insert races, network calls made inside a database transaction and unstable pagination are covered, with fixes, in database-engineering. In backend code each invariant lives in one conditional statement or constraint owned by one service, and no transaction stays open across a network call.

**Unguarded state transition.** `UPDATE orders SET status = 'shipped' WHERE id = $1`, or `if (order.status === 'paid')` in application code followed by a separate save. This ships cancelled orders, refunds twice, resurrects deleted items and lets two workers both win. Keep an allowed-transitions table in code, write `WHERE id = $1 AND status = ANY($2)`, and treat zero rows as a conflict (re-read to tell "already done" from "illegal"). Side effects tied to a transition run only in the request that won it.

### Jobs, locks and time

**Job that runs twice.** An in-process cron on every replica, a polling worker scaled to three pods, or workers that `SELECT * FROM jobs WHERE status = 'pending'` and start working without claiming. Every replica does the work. Claim atomically (`UPDATE ... WHERE id = (SELECT ... FOR UPDATE SKIP LOCKED) RETURNING`) with a lease. For schedules, let every replica enqueue with `ON CONFLICT DO NOTHING` on a key for the schedule slot, so exactly one job exists per slot.

**Lock with expiry and no fencing.** Redis `SET lock owner NX PX ttl`, a lease row, an etcd lease. The holder stalls (GC pause, CPU throttling, a slow call) past the TTL, a second worker takes the lock, and the first wakes up and keeps writing. A `finally { redis.del(lock) }` then deletes the second holder's lock. Each acquisition must produce a monotonically increasing fencing token, and every write the holder makes must be conditional on it (`WHERE lease_token = $t`), so a stale holder's writes hit zero rows. Release by compare-and-delete on your own value. When the protected resource cannot check a token, such as a third-party API, the lock only saves duplicate effort; correctness has to come from the downstream idempotency key.

**Ordering or expiry by wall clock.** `if (event.occurredAt < row.updatedAt) return`, `ORDER BY created_at` to pick the latest event, lease expiry compared with `Date.now()` from several hosts. Clocks skew, step backwards and tie at millisecond resolution, and a producer's timestamp is input from someone else. Order by a per-entity version assigned by the owner. Compute leases on one clock, the database's. In PostgreSQL `now()` is the transaction start time, so inside a long transaction use `clock_timestamp()`.

### Messaging

**Consumer without dedupe.** The handler assumes one delivery. Redelivery after a consumer crash, a partition rebalance, a visibility timeout expiring under a slow handler, or an outbox relay republishing after its own crash runs the effect again. Insert the message id into a `processed_messages` table with a unique constraint in the same transaction as the effect and skip when it conflicts. A dedupe check in Redis followed by a database write is another dual write.

**Ack before commit.** `msg.ack(); await handle(msg)`, or auto-ack. A crash after the ack loses the message. Ack after the transaction commits and accept the duplicates that follow from that.

**Out-of-order events.** `OrderCancelled` handled before `OrderCreated` (different partitions, retries, a replay from the dead-letter queue), or two `PriceChanged` events applied so the older price wins. Events carry the aggregate version. Snapshot-style projections apply an event only when its version is higher than the stored one. Delta events (points added, stock decremented) need strict sequence: apply only `version = current + 1`, and park or redeliver when there is a gap. An event for an entity you have never seen is parked or upserted, never dropped.

**Outbox relay with a high-water-mark cursor.** `SELECT * FROM outbox WHERE id > $cursor ORDER BY id`. Sequence values are assigned at insert, not at commit, so a transaction holding id 101 can commit after 102 has been published and the cursor has moved on. Event 101 is never sent. Select unpublished rows and mark each one published, or use logical decoding (CDC), which follows commit order.

**Poison message loop.** A message that always throws is retried forever, blocking its partition or burning a worker. Cap attempts, then move it to a dead-letter queue with the error, alert on it, and keep a tested replay path.

### Dependencies under load

**Missing timeouts.** HTTP clients, database drivers, pool checkout and lock waits often wait forever by default; node-postgres ships with `connectionTimeoutMillis`, `statement_timeout` and `query_timeout` all unset. One slow dependency parks every request and your service goes down with it. Give every outbound call a deadline taken from the caller's remaining budget, and give pool checkout its own timeout.

**Retry storm.** Client, gateway, service and driver each make up to three attempts, so one user request becomes up to 81 calls exactly when the dependency is overloaded; fixed delays make the retries arrive together. Retry at one layer only, cap attempts, back off exponentially with full jitter, honor `Retry-After`, keep a retry budget (retries as a bounded fraction of traffic), and do not retry a 4xx other than 408 and 429 unless the API documents it as retryable, such as an idempotency "request in progress" 409.

**No backpressure or circuit breaker.** `Promise.all(items.map(callApi))` over an unbounded list, an in-memory queue that grows until the process is OOM-killed, a consumer prefetch of thousands. When the dependency slows, latency and memory grow without limit. Bound concurrency, bound queues and reject when full (503 with `Retry-After`), open a breaker after repeated failures and probe before closing it, and shed load before the process falls over.

**Cache stampede.** A hot key expires and every concurrent request recomputes it against the database at once, or a cache warmed at deploy with one TTL expires all at once. Collapse concurrent misses into one load per key, jitter TTLs, and serve stale while revalidating where staleness is acceptable.

**Cache treated as the source of truth.** Permissions, balances, stock or entitlements decided from a cache invalidated on a best-effort basis; a cache written before the transaction commits and holding a value that rolled back; `DEL key` before the database write, so a concurrent reader repopulates the old value. Decisions that move money or grant access read the owner's store. Invalidate after commit (through the outbox when it must not be missed) and give every entry a bounded TTL.

### Edges and contracts

**Webhook processed inline.** The handler does all the business work, calls other services and returns 200 after twenty seconds; the provider times out and redelivers, so the work runs twice, and a provider backlog becomes your outage. A global JSON body parser also destroys the raw bytes, so the signature check gets removed "temporarily". Verify the signature over the raw body, insert the event keyed by the provider's event id with `ON CONFLICT DO NOTHING`, return 2xx, and process asynchronously. Applying the event in the same short transaction as the dedupe insert is fine when the effect is a local write; anything that calls another service goes through the async path. Treat the payload as a hint and refetch the object from the provider when its current state matters, since events arrive late, out of order and replayed.

**Breaking change shipped as a refactor.** Renaming or removing a field, `id` changing from number to string, tighter validation on input that used to pass, a changed default, a new enum value that old clients switch over exhaustively, different error codes, a field whose meaning changes. The same applies to event payloads, where messages in the old shape are still sitting in the queue during the deploy. Keep changes additive within a version; otherwise expand, migrate the known consumers, then contract. Readers ignore unknown fields and handle unknown enum values.

**Internal errors leaking.** `res.status(500).json({ error: err.message, stack: err.stack })`, database errors naming tables and constraints, upstream response bodies passed through, different messages for "no such user" and "wrong password". Map errors at the edge to a stable machine-readable shape (RFC 9457 problem details), log the detail with a correlation id, and return only the id.

### Process lifecycle

**Missing config falls back to a default.** `process.env.PAYOUT_API_URL ?? 'https://sandbox...'`, `DATABASE_URL || 'postgres://localhost/dev'`, `Number(process.env.LIMIT)` quietly becoming `NaN`, a security switch whose absence means off. In production a missing variable sends money to a sandbox while marking it paid, writes to the wrong database or disables a check. Parse all configuration once at startup and refuse to start when a required value is missing or malformed. Defaults are acceptable only for tunables where any value in range is safe (log level, pool size); never for destinations, credentials, environment identity or security switches.

**Shutdown that drops in-flight work or never finishes.** The SIGTERM handler exits at once and cuts off requests and jobs, or it waits on connections that never close until SIGKILL arrives. Load balancers keep routing to the instance for a few seconds after SIGTERM, and keep-alive connections that were busy when the listener closed stay open after their response. The sequence: fail readiness, keep serving through a short drain delay, stop accepting, finish in-flight requests and close each connection when its response is done, stop workers so their current job finishes or is abandoned to its lease, close pools, then exit, with a hard deadline inside the grace period that closes whatever is left. SIGKILL and OOM kills skip every handler, so long work must be resumable from a lease and idempotent steps anyway. Code in [references/api-and-lifecycle.md](references/api-and-lifecycle.md); signal delivery, grace periods and probes are in infrastructure-ops.

## Decision rules

- **Concurrency control.** One row, invariant expressible in `WHERE`: conditional `UPDATE` and check the row count. Decision needs other rows: `SELECT ... FOR UPDATE` in a consistent lock order, or SERIALIZABLE with a retry on SQLSTATE `40001`. A human editing for minutes: version column and 409 on mismatch. Across services: no distributed transaction; move the invariant into one owner, or run a saga with explicit compensations plus a reconciler.
- **Publishing events.** State and event in the same database: outbox. High volume or strict commit order: CDC from the write-ahead log. Losing the event is genuinely fine (metrics, cache warming): publish directly, and say so where it happens.
- **Idempotency keys.** Any POST that creates something or moves value takes a client key. Jobs and consumers derive the key from business identity (payout id, invoice id plus period), never per attempt. The key sent downstream is created and stored before the first call.
- **Where to retry.** At the single layer that knows whether the operation is idempotent, usually closest to the dependency. Every other layer fails fast.
- **Synchronous or accepted.** Answer synchronously when the work is local, fast and the caller needs the result. Return 202 with a status resource when the work calls slow or unreliable dependencies, or when the caller retries on its own clock (webhooks, batch clients).
- **Locks.** When the data lives in one database, claim rows there instead of taking a distributed lock. A distributed lock without fencing is an efficiency tool, never the only correctness guard.
- **Caching.** Cache what is expensive and tolerates staleness, and write down the tolerated staleness. Never cache the input to an authorization or money decision.
- **Service boundaries.** Split when there is an independent owner, scaling profile or release cadence. Shared tables mean it is still one service.

## Review checklist

Answer each with yes or no before calling a backend change done.

- Does every outbound call have a timeout, and is a timeout handled as an unknown outcome?
- Is every retried side effect protected by an idempotency key stored before the first attempt and reused on every attempt?
- Are client idempotency keys inserted before the work, scoped to the caller, and bound to a request fingerprint?
- Does every state change that others must hear about write to the outbox in the same transaction?
- Is every network call made with no database transaction or row lock held?
- Is every read-modify-write a single conditional statement, version-checked, or locked?
- Is every status change a conditional update from an allowed prior state, with zero rows handled?
- Can every job and consumer run twice, concurrently, without a double effect? Are lease holders fenced?
- Do consumers ack only after commit and dedupe in the same transaction as the effect?
- Does ordering come from owner-assigned versions rather than timestamps?
- Does every intermediate state have a sweeper or reconciler with a deadline and an alert?
- Are retries capped, jittered and done at one layer?
- Are concurrency and queues bounded, with a defined response when full?
- Are money and access decisions read from the owner store rather than a cache?
- Do webhooks verify the signature over raw bytes, dedupe by event id and return before processing?
- Do list endpoints page by a unique, stable key with a bounded limit?
- Is the API or event change additive, or is there an expand-and-contract plan with named consumers?
- Do error responses omit stack traces, SQL and upstream bodies?
- Does the process refuse to start with missing or malformed required configuration?
- On SIGTERM, does in-flight work finish or stay resumable within the grace period?
- For each line of the flow, do you know what happens on `kill -9` there and what notices?

## References

- [idempotency.md](references/idempotency.md): read when building or reviewing an endpoint or job that creates, charges, transfers or calls a provider that must not run twice. Key table, handler, fingerprinting, takeover, reconciliation.
- [outbox-and-consumers.md](references/outbox-and-consumers.md): read when a state change must produce an event or a consumer applies events. Outbox table, relay, dedupe inbox, ordering by version, dead letters.
- [jobs-and-state-machines.md](references/jobs-and-state-machines.md): read when writing background jobs, schedulers, locks or status fields. Lease claim with fencing token, heartbeats, guarded transitions, stuck-state recovery.
- [dependencies-and-load.md](references/dependencies-and-load.md): read when calling other services or putting a cache in front of something. Outcome classification, deadlines, retry with jitter and budget, breaker, bounded concurrency, stampede control.
- [api-and-lifecycle.md](references/api-and-lifecycle.md): read when changing an API surface, receiving webhooks, loading config or handling shutdown. Compatibility table, problem details, startup config, the shutdown sequence.

Related skills: database-engineering for schema, isolation levels and migrations; security-engineering for authorization and threat modeling; infrastructure-ops for deployment, probes and observability plumbing; test-engineering for concurrency and failure-injection tests.
