# API surface and process lifecycle

Read this when changing an API or event contract, adding a list endpoint, receiving webhooks, loading configuration, or handling shutdown and health probes.

Examples are TypeScript with Express, PostgreSQL and node-postgres. They assume Express 5, which forwards a rejected promise from an async handler to the error middleware; on Express 4 an async handler that throws produces an unhandled rejection and a hung request, so wrap handlers and call `next(err)`.

## Keyset pagination

```sql
CREATE INDEX invoices_tenant_created_idx ON invoices (tenant_id, created_at DESC, id DESC);

SELECT id, number, total_minor, created_at, created_at::text AS created_at_key
  FROM invoices
 WHERE tenant_id = $1
   AND (created_at, id) < ($2::timestamptz, $3::uuid)
 ORDER BY created_at DESC, id DESC
 LIMIT $4;
```

The first page runs the same query without the row comparison. Fetch `limit + 1` rows; if the extra row exists, there is a next page and the cursor is built from the last row you return.

The sort key must be unique, so `id` is always the tie-breaker. The cursor carries `created_at_key`, the database's own text form, because PostgreSQL stores microseconds and a JavaScript `Date` keeps milliseconds; a cursor built from `Date` sits slightly before the real value and silently skips or repeats rows at page boundaries.

```ts
const UUID_PATTERN = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

export type InvoiceCursor = { createdAt: string; id: string };

export function encodeCursor(cursor: InvoiceCursor): string {
  return Buffer.from(JSON.stringify([cursor.createdAt, cursor.id]), 'utf8').toString('base64url');
}

export function decodeCursor(raw: string, maxLength: number): InvoiceCursor {
  if (raw.length > maxLength) throw new BadRequestError('invalid cursor');
  let parsed: unknown;
  try {
    parsed = JSON.parse(Buffer.from(raw, 'base64url').toString('utf8'));
  } catch {
    throw new BadRequestError('invalid cursor');
  }
  if (
    !Array.isArray(parsed) ||
    parsed.length !== 2 ||
    typeof parsed[0] !== 'string' ||
    typeof parsed[1] !== 'string' ||
    Number.isNaN(Date.parse(parsed[0])) ||
    !UUID_PATTERN.test(parsed[1])
  ) {
    throw new BadRequestError('invalid cursor');
  }
  return { createdAt: parsed[0], id: parsed[1] };
}
```

The cursor is untrusted input. It only says where to resume; the tenant filter, the caller's permissions and the page size are applied from the request context on every page, never read back from the cursor. Bound `limit` with a maximum from configuration.

Offset pagination is acceptable for small, rarely changing lists (admin screens, settings) where a skipped or repeated row is harmless.

## Compatibility

Treat HTTP APIs, event payloads, queue messages and webhook payloads you send as contracts with consumers you do not deploy.

| Change | Safe for existing consumers |
| --- | --- |
| Add an optional response field | Yes, if consumers ignore unknown fields |
| Add an optional request field whose absence keeps the old behaviour | Yes |
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

This follows RFC 9457 problem details, with the correlation id as an extension member. When the response has already started streaming, the handler hands the error to Express's default handler, which closes the connection; calling `res.status` at that point would throw. `publicDetail` is written for the client; the error's message, stack, SQL, constraint names and upstream bodies go to the log only. Authentication failures return the same response whether the account exists or not.

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

```ts
import type { Server } from 'node:http';
import { setTimeout as delay } from 'node:timers/promises';

export function installShutdown(
  server: Server,
  deps: { markNotReady: () => void; stopWorkers: () => Promise<void>; pool: Pool },
  cfg: { drainDelayMs: number; closeTimeoutMs: number },
): void {
  let started = false;

  const shutdown = async (): Promise<void> => {
    deps.markNotReady();
    await delay(cfg.drainDelayMs);
    const force = setTimeout(() => {
      log.warn('shutdown deadline reached, closing remaining connections');
      server.closeAllConnections();
    }, cfg.closeTimeoutMs);
    force.unref();
    await Promise.all([
      new Promise<void>((resolve, reject) => server.close((err) => (err ? reject(err) : resolve()))),
      deps.stopWorkers(),
    ]);
    clearTimeout(force);
    await deps.pool.end();
  };

  for (const signal of ['SIGTERM', 'SIGINT'] as const) {
    process.once(signal, () => {
      if (started) return;
      started = true;
      log.info({ signal }, 'shutdown started');
      shutdown().then(
        () => process.exit(0),
        (err: unknown) => {
          log.error({ err }, 'shutdown failed');
          process.exit(1);
        },
      );
    });
  }
}
```

- Kubernetes removes the pod from Service endpoints at the same time as it starts graceful shutdown, and that removal takes time to reach every proxy and load balancer. Readiness goes false first and the process keeps serving through `drainDelayMs`, so requests routed during propagation still succeed. A `preStop` hook that waits achieves the same, and it runs before SIGTERM.
- `server.close()` stops accepting and waits for in-flight requests. From Node 19 it also closes idle keep-alive connections; on Node 18.2 and later but before 19, call `server.closeIdleConnections()` after it or idle sockets hold the close open. `closeAllConnections()` is the forced path at the deadline.
- `stopWorkers` aborts the workers' `shuttingDown` signal and waits for their loops to return (jobs-and-state-machines.md). A job still running at the deadline is abandoned, and its lease expiry hands it to another worker.
- `drainDelayMs + closeTimeoutMs` plus pool shutdown must fit inside `terminationGracePeriodSeconds` (default 30), and time spent in `preStop` counts against the same period. After that comes SIGKILL, which no handler sees.
- The signal has to reach the process. A shell-form Docker `CMD` or `ENTRYPOINT` runs under `/bin/sh -c`, which does not pass signals on; use exec form, or `exec` the process from a wrapper script. A process running as PID 1 with no SIGTERM handler ignores SIGTERM.

## Health probes

- Liveness: the process can make progress (the event loop answers). No database or downstream checks, since a shared dependency failing would restart every pod at once.
- Readiness: false during startup until config, pools and caches are ready, and false from the start of shutdown. Checking a hard dependency here is reasonable only if routing traffic to other pods would actually help; when every pod shares the dependency, all of them go unready together and the service returns nothing at all instead of useful errors.
- Startup: a separate startup probe covers slow boots, so liveness can stay strict.
