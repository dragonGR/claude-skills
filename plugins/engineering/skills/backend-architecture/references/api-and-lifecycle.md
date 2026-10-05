# API surface and process lifecycle

Read this when changing an API or event contract, receiving webhooks, loading configuration, or handling shutdown.

Examples are TypeScript with Express, PostgreSQL and node-postgres. They assume Express 5, which forwards a rejected promise from an async handler to the error middleware; on Express 4 an async handler that throws produces an unhandled rejection and a hung request, so wrap handlers and call `next(err)`.

## List endpoints

Page with keyset pagination on a unique, immutable sort key, behind an opaque cursor that is validated as untrusted input and never carries the tenant or filters. The SQL, index and cursor encoding are in database-engineering's `patterns.md`.

## Compatibility

Treat HTTP APIs, event payloads, queue messages and webhook payloads you send as contracts with consumers you do not deploy.

| Change | Safe for existing consumers |
| --- | --- |
| Add an optional response field | Yes, if consumers ignore unknown fields |
| Add an optional request field whose absence keeps the old behavior | Yes |
| Add a required request field | No |
| Remove or rename a field | No |
| Change a type, unit or format (number to string, cents to decimal, date format) | No |
| Add an enum value | Breaks consumers that switch exhaustively; announce it and require a default branch |
| Tighten validation on existing input | No |
| Change a default, sort order or page size | No |
| Change the status code or error code for an existing condition | No |
| New event type on an existing topic | Only if consumers skip unknown types |

For anything marked no: add the new shape alongside the old, move the known consumers, measure that the old shape has no traffic, then remove it. For events, old-shape messages remain in queues and in dead-letter queues through the deploy, so consumers accept both shapes until those have drained.

As a consumer, parse strictly what you use, ignore fields you do not use, and route unknown enum values and event types to an explicit branch that logs and skips or parks them.

## Errors at the edge

```ts
app.use((err: unknown, _req: Request, res: Response, next: NextFunction) => {
  if (res.headersSent) return next(err);
  const correlationId = String(res.locals.correlationId);
  if (err instanceof AppError) {
    res.status(err.status).type('application/problem+json').json({
      type: err.typeUri,
      title: err.title,
      status: err.status,
      detail: err.publicDetail,
      correlationId,
    });
    return;
  }
  log.error({ err, correlationId }, 'unhandled error');
  res.status(500).type('application/problem+json').json({
    type: 'about:blank',
    title: 'Internal Server Error',
    status: 500,
    correlationId,
  });
});
```

This follows RFC 9457 problem details, with the correlation id as an extension member. When the response has already started streaming, the handler hands the error to Express's default handler, which closes the connection; writing a new response at that point would throw `ERR_HTTP_HEADERS_SENT`. `publicDetail` is written for the client; the error's message, stack, SQL, constraint names and upstream bodies go to the log only. Authentication failures return the same response whether the account exists or not.

## Receiving webhooks

```ts
app.post(
  '/webhooks/payments',
  express.raw({ type: 'application/json', limit: config.webhookMaxBytes }),
  async (req, res) => {
    if (!Buffer.isBuffer(req.body)) throw new BadRequestError('expected a JSON body');
    const event = verifyPaymentWebhook(req.body, req.get(config.webhookSignatureHeader), config.webhookSecret);
    await pool.query(
      `INSERT INTO inbound_events (source, event_id, event_type, payload)
       VALUES ('payments', $1, $2, $3)
       ON CONFLICT (source, event_id) DO NOTHING`,
      [event.id, event.type, JSON.stringify(event)],
    );
    res.sendStatus(204);
  },
);
```

- `verifyPaymentWebhook` wraps the provider SDK's own verification function and throws on a missing or invalid signature. It must see the exact bytes received. Mount this route before any global `express.json()`, or the raw body is gone and the check cannot work.
- If the provider signs a timestamp, reject events outside its tolerance to limit replay; the provider SDK usually does this.
- The insert is the dedupe. Providers redeliver on timeouts and non-2xx responses, sometimes long after the first delivery.
- A job processes `inbound_events` through the usual lease claim. When the event is about an object's state (a payment succeeded, a subscription changed), fetch the object from the provider's API and apply that state through a guarded transition, because events arrive out of order and a replayed old event must not roll state back.

## Configuration at startup

```ts
export class ConfigError extends Error {}

const APP_ENVS = ['development', 'staging', 'production'] as const;
const LOG_LEVELS = ['debug', 'info', 'warn', 'error'] as const;

function required(env: NodeJS.ProcessEnv, name: string): string {
  const value = env[name]?.trim();
  if (!value) throw new ConfigError(`${name} is required`);
  return value;
}

function oneOf<T extends string>(env: NodeJS.ProcessEnv, name: string, allowed: readonly T[], fallback?: T): T {
  const raw = env[name]?.trim();
  if (!raw) {
    if (fallback === undefined) throw new ConfigError(`${name} is required`);
    return fallback;
  }
  const match = allowed.find((candidate) => candidate === raw);
  if (match === undefined) throw new ConfigError(`${name} must be one of ${allowed.join(', ')}`);
  return match;
}

function httpsUrl(env: NodeJS.ProcessEnv, name: string): URL {
  const raw = required(env, name);
  let url: URL;
  try {
    url = new URL(raw);
  } catch {
    throw new ConfigError(`${name} is not a valid URL`);
  }
  if (url.protocol !== 'https:') throw new ConfigError(`${name} must use https`);
  return url;
}

function intInRange(env: NodeJS.ProcessEnv, name: string, min: number, max: number, fallback?: number): number {
  const raw = env[name]?.trim();
  if (!raw) {
    if (fallback === undefined) throw new ConfigError(`${name} is required`);
    return fallback;
  }
  const value = /^-?\d+$/.test(raw) ? Number(raw) : Number.NaN;
  if (!Number.isSafeInteger(value) || value < min || value > max) {
    throw new ConfigError(`${name} must be an integer between ${min} and ${max}`);
  }
  return value;
}

export function loadConfig(env: NodeJS.ProcessEnv) {
  return {
    appEnv: oneOf(env, 'APP_ENV', APP_ENVS),
    payoutApiUrl: httpsUrl(env, 'PAYOUT_API_URL'),
    payoutApiKey: required(env, 'PAYOUT_API_KEY'),
    databaseUrl: required(env, 'DATABASE_URL'),
    httpPort: intInRange(env, 'PORT', 1, 65535),
    dbPoolMax: intInRange(env, 'DB_POOL_MAX', 1, 200, 10),
    logLevel: oneOf(env, 'LOG_LEVEL', LOG_LEVELS, 'info'),
  } as const;
}
```

Call `loadConfig(process.env)` before opening ports or pools and let a `ConfigError` exit the process with a non-zero code; a crash-looping deploy is visible, a service quietly running against a sandbox is not. Destinations, credentials, environment identity and security switches have no fallback. Tunables where every in-range value is safe (log level, pool size) may have one. Where credentials encode their mode (Stripe secret keys start with `sk_live_` or `sk_test_`), check the mode against `appEnv` at startup. Never log the config object.

## Graceful shutdown

The sequence is the same in every runtime:

1. On SIGTERM, fail readiness and keep serving. Kubernetes starts removing the pod from Service endpoints at the same moment it sends SIGTERM, and the change takes seconds to reach every proxy and load balancer, so new requests keep arriving.
2. Wait a drain delay long enough for that propagation. A `preStop` sleep does the same job; it runs before SIGTERM and its time counts against the grace period.
3. Stop accepting new connections.
4. Let in-flight requests finish. Close idle keep-alive connections at once, and close each busy connection as soon as its response has been sent, with `Connection: close` on responses that start after shutdown began. A busy connection left alone stays open after its response until the server's keep-alive timeout, which behind a load balancer is often longer than the whole grace period.
5. Stop workers and consumers: no new claims, and the current job finishes or is abandoned to its lease expiry.
6. Close pools, flush logs, exit 0.
7. A deadline a few seconds inside the grace period (`terminationGracePeriodSeconds`, default 30, minus any `preStop` time) exits non-zero with whatever is still open. SIGKILL follows the grace period and no handler sees it.

In Node, step 4 needs code of its own, and the obvious versions cut responses off. `server.close()` and `server.closeIdleConnections()` both treat a connection whose response has ended but is still being written to a slow client as idle and destroy it; on Node 26.10 a slow client reading a 64 MiB response got under 20 MB and a reset with either call. A connection left alone after its response instead waits out `keepAliveTimeout`, which on a server tuned for a load balancer is longer than the grace period. The working pattern sends `Connection: close` on responses that start after shutdown began, ends each socket on the response's `'finish'` event, and calls `close()` only once no response is mid-write. The tested routine is in nodejs-engineering.

Workers stop through the `shuttingDown` signal in jobs-and-state-machines.md; a job still running at the deadline is abandoned, and its lease expiry hands it to another worker. Signal delivery to PID 1, exec-form `CMD` and probe configuration are in infrastructure-ops.
