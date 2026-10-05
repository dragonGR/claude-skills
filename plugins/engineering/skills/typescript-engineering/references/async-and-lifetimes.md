# Async code and resource lifetimes

Read this when writing or reviewing promise-heavy code: fan-out over lists, outbound HTTP, background work, streams or timers.

The question to ask of every promise, timer, listener and stream is who owns it: who observes its failure, who stops it, and what happens to it when the request, the invocation or the process ends. Code without an answer leaks, loses errors or does work after the caller has reported failure.

## Choosing a concurrency shape

| Situation | Shape |
|---|---|
| Small fixed set of independent reads, all needed | `Promise.all` |
| Siblings should stop when one fails | `Promise.all` plus a shared `AbortController` aborted in `catch` |
| Each outcome is reported or compensated separately | `Promise.allSettled`, then inspect every result |
| Input sized by data (rows, users, files) | Bounded workers, below |
| Order matters, or each step depends on the previous | `for...of` with `await` |
| Work must survive a crash or deploy | Durable queue or outbox, not an in-process promise |

### Stopping siblings on first failure

`Promise.all` rejects as soon as one input rejects and does nothing to the others. They only stop if they honor a signal you abort.

```ts
async function bestQuote(req: QuoteRequest, providers: readonly QuoteProvider[]): Promise<Quote[]> {
  const controller = new AbortController();
  const signal = AbortSignal.any([controller.signal, AbortSignal.timeout(config.quoteTimeoutMs)]);
  try {
    return await Promise.all(providers.map((p) => p.quote(req, signal)));
  } catch (err: unknown) {
    controller.abort(err);
    throw err;
  }
}
```

### Compensating after partial failure

Compensate only once every operation has settled; otherwise the cleanup races a create that lands a second later.

```ts
const results = await Promise.allSettled(steps.map((step) => step.create(spec, idempotencyKey)));
const reasons = results.flatMap((r) => (r.status === "rejected" ? [r.reason] : []));
if (reasons.length > 0) {
  const created = results.flatMap((r) => (r.status === "fulfilled" ? [r.value] : []));
  await compensate(created, spec, idempotencyKey);
  throw new AggregateError(reasons, "provisioning failed");
}
```

A rejection caused by a timeout may still have succeeded remotely. `compensate` therefore looks resources up by the idempotency key rather than trusting the `created` list alone.

### Bounded concurrency

`Promise.all(rows.map(enrich))` over 20,000 rows opens 20,000 requests at once, exhausts sockets and the connection pool, and trips the remote side's rate limits. Use a fixed number of workers pulling from one shared iterator. If the project already depends on `p-limit` or similar, use that instead.

```ts
export async function mapSettledBounded<T, R>(
  items: Iterable<T>,
  limit: number,
  fn: (item: T) => Promise<R>,
): Promise<PromiseSettledResult<R>[]> {
  if (!Number.isInteger(limit) || limit < 1) {
    throw new RangeError(`limit must be a positive integer, got ${limit}`);
  }
  const results: PromiseSettledResult<R>[] = [];
  const entries = Array.from(items).entries();

  async function worker(): Promise<void> {
    // All workers share one iterator, so each entry is taken exactly once.
    for (const [index, item] of entries) {
      try {
        results[index] = { status: "fulfilled", value: await fn(item) };
      } catch (reason: unknown) {
        results[index] = { status: "rejected", reason };
      }
    }
  }

  await Promise.all(Array.from({ length: limit }, worker));
  return results;
}
```

The limit comes from configuration and reflects the smallest downstream capacity: the DB pool size, the partner's rate limit, the file descriptor budget.

## Outbound HTTP

Node's `fetch` has no overall deadline. undici's defaults allow 300 seconds waiting for headers and 300 seconds between body chunks, so a stalled upstream holds the request, its pool slot and whatever the caller holds for minutes.

```ts
export async function fetchJson<S extends z.ZodType>(
  url: URL,
  schema: S,
  init: RequestInit & { timeoutMs: number },
): Promise<z.output<S>> {
  const { timeoutMs, signal, ...rest } = init;
  const timeout = AbortSignal.timeout(timeoutMs);
  const res = await fetch(url, {
    ...rest,
    // The signal stays attached while the body is read, so the deadline covers res.json() too.
    signal: signal ? AbortSignal.any([signal, timeout]) : timeout,
  });
  if (!res.ok) {
    await res.body?.cancel();
    throw new UpstreamError(res.status, url.host);
  }
  return schema.parse(await res.json());
}
```

Points a reviewer checks:

- `fetch` resolves for 4xx and 5xx. Code that reads `await res.json()` without `res.ok` parses an error page as data.
- Every body is consumed or cancelled. undici otherwise leaves connection release to the garbage collector, and the pool can run dry.
- A timeout rejects with a `DOMException` named `TimeoutError`; an abort from the caller's signal is an `AbortError`. Map them differently if the caller cares.
- A timed-out or reset write is an unknown outcome. Retry only idempotent requests or requests carrying an idempotency key the server honors, with capped attempts and jittered backoff, and never retry on 4xx other than 408 and 429. The protocol side lives in backend-architecture.
- The URL is built with `new URL(path, base)` from a fixed path template, with user values in path segments passed through `encodeURIComponent` and query values set through `URLSearchParams`. Never pass user input as the whole `path`: `new URL("https://evil.example", base)` ignores `base` (SSRF is covered in security-engineering).

## Background work in a Node server

A promise started in a handler and not awaited is either lost on failure or kills the process through an unhandled rejection. Give best-effort background work an owner that logs failures and is drained at shutdown. Anything that must happen (a charge, an email the user was promised, a ledger entry) goes through a durable queue or outbox instead.

```ts
export class BackgroundTasks {
  readonly #pending = new Set<Promise<void>>();
  readonly #log: Logger;
  #closed = false;

  constructor(log: Logger) {
    this.#log = log;
  }

  run(name: string, task: () => Promise<void>): void {
    if (this.#closed) throw new Error(`background task "${name}" started after shutdown`);
    const tracked: Promise<void> = Promise.resolve()
      .then(task)
      .catch((err: unknown) => {
        this.#log.error({ err, task: name }, "background task failed");
      })
      .finally(() => this.#pending.delete(tracked));
    this.#pending.add(tracked);
  }

  async drain(graceMs: number): Promise<void> {
    this.#closed = true;
    const settled = Promise.allSettled(this.#pending).then(() => "done" as const);
    const expired = new Promise<"expired">((resolve) => {
      setTimeout(resolve, graceMs, "expired").unref();
    });
    if ((await Promise.race([settled, expired])) === "expired") {
      this.#log.warn({ pending: this.#pending.size }, "exiting with background tasks unfinished");
    }
  }
}
```

On Workers the equivalent owner is `ctx.waitUntil` (`references/runtimes.md`).

## Graceful shutdown

`BackgroundTasks.drain` runs in the shutdown sequence after the HTTP server has closed, alongside stopping queue consumers. The sequence itself (readiness, drain delay, a deadline under the grace period) is in backend-architecture's api-and-lifecycle reference, and the Node server code, including the keep-alive connections that hold `server.close()` open, is in nodejs-engineering. Register the signal listener as a synchronous function that starts the async shutdown and handles both outcomes, because a rejected `async` listener is an unhandled rejection.

After `uncaughtException`, log synchronously and exit non-zero without draining: the process state is undefined, and Node's documentation says resuming is not safe. An unhandled rejection already ends the process under Node's default `throw` mode. The fix is the floating promise that caused it; a process-wide handler that only logs turns crashes into silent corruption. Crash policy is owned by nodejs-engineering.

## Cancellation plumbing

- Accept an `AbortSignal` in every function that does I/O or long loops, pass it to `fetch`, driver calls that accept one, `setTimeout` from `node:timers/promises`, and `pipeline`.
- In loops, call `signal.throwIfAborted()` between units of work.
- Combine signals with `AbortSignal.any([...])` instead of adding listeners to a long-lived parent signal per request. A listener added to the server's shutdown signal on every request and never removed is a leak.
- When you do add a listener, pass `{ once: true }` and remove it in `finally` if the operation completes first.

## Timers

- Every `setInterval` has a matching `clearInterval` in the owner's close path. An interval created at module load in a library keeps test runners and CLIs from exiting; `unref()` it.
- `setTimeout` delays above 2147483647 ms are replaced with 1 ms. Re-arm in steps for long waits, and for anything measured in days prefer a persistent scheduler, since the process will restart before it fires.

```ts
const MAX_TIMER_DELAY_MS = 2_147_483_647;

export function setTimeoutAt(fireAt: number, fn: () => void, signal: AbortSignal): void {
  // NaN (for example from Date.parse on bad input) would become a 1 ms delay and re-arm forever.
  if (!Number.isFinite(fireAt)) throw new RangeError(`fireAt must be a finite epoch time, got ${fireAt}`);
  if (signal.aborted) return;
  let timer: ReturnType<typeof setTimeout> | undefined;
  const cancel = (): void => clearTimeout(timer);
  const arm = (): void => {
    const remaining = fireAt - Date.now();
    if (remaining <= 0) {
      signal.removeEventListener("abort", cancel);
      fn();
      return;
    }
    timer = setTimeout(arm, Math.min(remaining, MAX_TIMER_DELAY_MS));
  };
  signal.addEventListener("abort", cancel, { once: true });
  arm();
}
```

- Measure durations with `performance.now()`, not `Date.now()`, which follows wall-clock corrections.

## Streams

- Join streams with `pipeline` from `node:stream/promises`. It forwards errors, destroys every stream on failure or client disconnect, and accepts `{ signal }`. `.pipe()` leaves the destination open when the source errors.
- An async generator works as a pipeline source, which is the simplest way to stream rows with backpressure handled for you:

```ts
await pipeline(
  async function* () {
    for await (const row of db.exportRows(accountId, { batchSize: config.exportBatchSize })) {
      yield `${JSON.stringify(row)}\n`;
    }
  },
  zlib.createGzip(),
  res,
  { signal },
);
```

- When writing by hand, stop when `write()` returns `false` and wait for `'drain'` (`await once(stream, "drain")`, which rejects if the stream emits `'error'`). Node otherwise buffers every chunk in memory.
- `await res.text()`, `arrayBuffer()` or `JSON.parse` on an upload buffers the whole body. Enforce a size limit before reading, and stream anything that can be large.

## Event emitters

- An `'error'` event with no listener throws and exits the process. Any emitter you create or receive (sockets, child processes, streams outside `pipeline`, custom emitters) needs an error listener.
- `MaxListenersExceededWarning` means listeners are being added per operation and not removed. Find the missing `off`; raising `setMaxListeners` hides the leak.
- `events.once(emitter, name, { signal })` and `events.on(emitter, name, { signal })` give promise and async-iterator forms that clean up when the signal aborts.
