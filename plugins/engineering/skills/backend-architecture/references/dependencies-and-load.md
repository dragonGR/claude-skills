# Calling dependencies and surviving load

Read this when code calls another service, database or provider, retries anything, fans out work, or puts a cache in front of a store.

Examples are TypeScript on Node. The rules are the same in any runtime.

## Classify every outcome

Decide per dependency, in one function, what each observation means. The table is the default; a provider's documentation overrides it where it says more.

| Observation | Outcome | Retry with no idempotency key | Retry with the same key |
| --- | --- | --- | --- |
| DNS failure, connection refused, TLS handshake failure | Not sent | Yes | Yes |
| Timeout after the request was written, connection reset, socket closed on a reused keep-alive connection | Unknown | No | Yes, or reconcile first |
| 500, 502, 504, unparsable body | Unknown | No | Yes, unless the provider stores errors under the key (Stripe does for 500); then reconcile by lookup |
| 503 or 429 | Not processed only if the provider documents it | Only if documented | Yes, after `Retry-After` |
| 400, 401, 403, 404, 422 | Definitive failure | No | No, fix the request |
| 409 "same key in progress" | Another attempt running | No | Yes, after a delay |
| 2xx | Success | No | No |

Unknown is its own state in your data. It is resolved by a lookup or by a keyed retry, never by assuming failure. A keyed retry only helps when the provider has not stored the failure under that key (idempotency.md).

## Deadlines

Each inbound request gets a budget. Every outbound call gets the smaller of the time left and that dependency's own cap, and a call that cannot finish in the time left is not started.

```ts
export class Deadline {
  private constructor(private readonly expiresAt: number) {}

  static after(ms: number): Deadline {
    return new Deadline(performance.now() + ms);
  }

  remainingMs(): number {
    return Math.max(0, this.expiresAt - performance.now());
  }

  signal(capMs: number): AbortSignal {
    return AbortSignal.timeout(Math.min(this.remainingMs(), capMs));
  }
}

export class DeadlineExceededError extends Error {}

export async function callInventory(deadline: Deadline, sku: string, cfg: InventoryConfig): Promise<Response> {
  if (deadline.remainingMs() < cfg.minUsefulMs) throw new DeadlineExceededError('inventory: no budget left');
  return fetch(new URL(`/v1/stock/${encodeURIComponent(sku)}`, cfg.baseUrl), {
    signal: deadline.signal(cfg.timeoutMs),
  });
}
```

Use a monotonic clock (`performance.now()`) for elapsed time inside a process; wall-clock time can step. When passing a deadline to another service, send the remaining duration, not an absolute timestamp, and have the receiver cap it at its own maximum instead of trusting the caller.

The database needs its own limits: a statement timeout, a lock timeout, a connection timeout, and an idle-in-transaction timeout so a stuck client cannot hold locks forever. node-postgres exposes `statement_timeout`, `query_timeout`, `lock_timeout`, `idle_in_transaction_session_timeout` and `connectionTimeoutMillis`, all unset by default.

## Retry with backoff, jitter and a budget

```ts
import { setTimeout as delay } from 'node:timers/promises';

export type RetryPolicy = { maxAttempts: number; baseDelayMs: number; maxDelayMs: number };

export class RetryBudget {
  private tokens: number;

  constructor(private readonly depositPerRequest: number, private readonly maxTokens: number) {
    this.tokens = maxTokens;
  }

  recordRequest(): void {
    this.tokens = Math.min(this.maxTokens, this.tokens + this.depositPerRequest);
  }

  tryWithdraw(): boolean {
    if (this.tokens < 1) return false;
    this.tokens -= 1;
    return true;
  }
}

export async function withRetry<T>(
  op: (attempt: number) => Promise<T>,
  isRetryable: (err: unknown) => boolean,
  policy: RetryPolicy,
  budget: RetryBudget,
  deadline: Deadline,
): Promise<T> {
  budget.recordRequest();
  for (let attempt = 1; ; attempt += 1) {
    try {
      return await op(attempt);
    } catch (err) {
      const backoffCap = Math.min(policy.maxDelayMs, policy.baseDelayMs * 2 ** (attempt - 1));
      const wait = Math.random() * backoffCap;
      const giveUp =
        attempt >= policy.maxAttempts ||
        !isRetryable(err) ||
        wait >= deadline.remainingMs() ||
        !budget.tryWithdraw();
      if (giveUp) throw err;
      await delay(wait);
    }
  }
}
```

- Full jitter (a random wait between zero and the exponential cap) spreads retries out; a fixed or merely exponential delay makes every client retry in the same instant.
- The budget lets retries be a bounded fraction of traffic (`depositPerRequest` of 0.1 means about one retry per ten requests). When a dependency is down, retries stop instead of multiplying the load.
- `isRetryable` comes from the classification table. For side effects it returns true for unknown outcomes only when `op` sends the same idempotency key on every attempt.
- If the response carries `Retry-After`, wait at least that long or give up if it exceeds the deadline.
- Only one layer retries. If the HTTP client, mesh or SDK already retries, turn one of them off; stacked retries multiply.

## Circuit breaker

A breaker stops calling a dependency that is failing, so requests fail fast instead of piling up behind timeouts, and the dependency gets room to recover.

```ts
type BreakerState =
  | { kind: 'closed'; consecutiveFailures: number }
  | { kind: 'open'; retryAt: number }
  | { kind: 'half_open'; probeInFlight: boolean };

export class CircuitOpenError extends Error {}

export class CircuitBreaker {
  private state: BreakerState = { kind: 'closed', consecutiveFailures: 0 };

  constructor(private readonly failureThreshold: number, private readonly openMs: number) {}

  async run<T>(op: () => Promise<T>, indicatesUnhealthy: (err: unknown) => boolean): Promise<T> {
    if (this.state.kind === 'open') {
      if (performance.now() < this.state.retryAt) throw new CircuitOpenError();
      this.state = { kind: 'half_open', probeInFlight: false };
    }
    if (this.state.kind === 'half_open') {
      if (this.state.probeInFlight) throw new CircuitOpenError();
      this.state.probeInFlight = true;
    }
    try {
      const result = await op();
      this.state = { kind: 'closed', consecutiveFailures: 0 };
      return result;
    } catch (err) {
      this.onError(indicatesUnhealthy(err));
      throw err;
    }
  }

  private onError(unhealthy: boolean): void {
    if (this.state.kind === 'half_open') {
      this.state = unhealthy
        ? { kind: 'open', retryAt: performance.now() + this.openMs }
        : { kind: 'closed', consecutiveFailures: 0 };
      return;
    }
    if (this.state.kind === 'closed' && unhealthy) {
      const failures = this.state.consecutiveFailures + 1;
      this.state =
        failures >= this.failureThreshold
          ? { kind: 'open', retryAt: performance.now() + this.openMs }
          : { kind: 'closed', consecutiveFailures: failures };
    }
  }
}
```

Count only errors that say something about the dependency's health (timeouts, connection errors, 5xx). A 404 or a validation error means the dependency is answering. At high request rates, a failure ratio over a sliding window trips more reliably than a consecutive count. Keep one breaker per dependency and instance pool, and export its state as a metric; an open breaker is an incident signal. `CircuitOpenError` maps to 503 with `Retry-After` at your edge.

## Bounded concurrency and load shedding

`Promise.all(items.map(fn))` starts every call at once. Bound it:

```ts
export async function mapBounded<T, R>(
  items: readonly T[],
  limit: number,
  fn: (item: T) => Promise<R>,
): Promise<R[]> {
  if (!Number.isInteger(limit) || limit < 1) throw new RangeError(`limit must be a positive integer, got ${limit}`);
  const results = new Array<R>(items.length);
  const queue = items.entries();
  const worker = async (): Promise<void> => {
    for (const [index, item] of queue) results[index] = await fn(item);
  };
  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
  return results;
}
```

The workers share one iterator, so each item is taken exactly once. When one call rejects, `Promise.all` rejects but the other workers keep draining the iterator; pass an `AbortSignal` into `fn` if the remaining work should stop.

At the edge, reject work you cannot finish instead of queueing it until memory runs out:

```ts
export function shedLoad(maxInFlight: number, retryAfterSeconds: number): RequestHandler {
  let inFlight = 0;
  return (req, res, next) => {
    if (inFlight >= maxInFlight) {
      res.set('Retry-After', String(retryAfterSeconds));
      res.status(503).end();
      return;
    }
    inFlight += 1;
    res.once('close', () => {
      inFlight -= 1;
    });
    next();
  };
}
```

The same rule applies to every buffer: in-memory queues have a maximum length, consumer prefetch is sized to what the handler can finish within the message's visibility or ack timeout, and connection pools are sized against the database's connection limit divided across every replica.

## Caches

Cache-aside with stampede control:

```ts
export class SingleFlight<T> {
  private readonly inflight = new Map<string, Promise<T>>();

  run(key: string, load: () => Promise<T>): Promise<T> {
    const existing = this.inflight.get(key);
    if (existing) return existing;
    const pending = load().finally(() => this.inflight.delete(key));
    this.inflight.set(key, pending);
    return pending;
  }
}

export function jitteredTtlSeconds(baseSeconds: number, jitterFraction: number): number {
  return Math.round(baseSeconds * (1 - jitterFraction * Math.random()));
}
```

- Single-flight collapses concurrent misses within one process. Across many processes you still get one load per process; if that is too many, add a short recompute lock in the shared cache and serve the stale value to everyone who does not hold it. That lock only saves work, so it needs no fencing, but it must expire.
- Jittered TTLs stop keys written together (a deploy warming the cache) from expiring together.
- Stale-while-revalidate: store a soft expiry inside the value and a longer hard TTL on the key. Past the soft expiry, return the stale value and refresh in the background through single-flight.
- Invalidate after the transaction commits, by deleting the key rather than writing the new value. A reader that loaded the old row just before the commit can still write it back after your delete; the bounded TTL is what limits that window, so every key has one.
- Caching "not found" protects the store from lookups of missing ids, but a negative entry must expire quickly or be deleted when the row is created.
- Never read permissions, balances, stock or limits from a cache when the answer authorizes an action or moves value. Read those from the owner inside the transaction that acts on them.
