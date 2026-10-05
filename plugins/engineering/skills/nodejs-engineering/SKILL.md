---
name: nodejs-engineering
description: The Node.js process in production: crash policy, SIGTERM shutdown code, keep-alive and request timeouts behind load balancers, heap limits in containers, the libuv thread pool, child processes, inspector debugging, crash diagnostics, Node version upgrades. Load it for Node servers, workers and CLIs, and when one leaks, hangs, crashes or returns 502s.
license: MIT
metadata:
  author: Alex Tsanis
---

# Node.js engineering

Most Node incidents are not language bugs. They are runtime behavior nobody configured: a keep-alive timeout shorter than the load balancer's, a heap limit larger than the container, four thread-pool slots shared by DNS and file reads, a process that keeps serving after its state is corrupt, a shutdown that waits on sockets nobody closes. This skill covers the process as a deployed thing: how it crashes, stops, uses memory and threads, how to debug it and how to move it to a new Node major. Code-level correctness (types, validation, async patterns, numbers), npm dependencies and supply chain, and non-Node runtimes belong to typescript-engineering; the language-neutral shutdown sequence to backend-architecture; container entrypoints and probes to infrastructure-ops; security review to security-engineering.

Check the Node version first (`node --version`, `engines`, `.nvmrc`, the Docker base image). Defaults below are for current releases; several have changed between majors.

## Failure catalogue

### Crashes and process lifecycle

**Continuing after `uncaughtException`.** A handler that logs and carries on leaves the process running with half-finished state: a transaction never committed, a lock never released, a counter never decremented. Node's own docs say the process is in an undefined state at that point. Do synchronous cleanup only, log with full context, and exit non-zero without draining, so the supervisor restarts a clean process.

**Silencing unhandled rejections.** Since Node 15 the default mode is `throw`, so an unhandled rejection ends the process. Adding `process.on('unhandledRejection', console.error)` turns that crash into silent data loss. Fix the floating promise. Keep the default, or log and exit.

**Shutdown that drops work or never finishes.** On SIGTERM the handler calls `process.exit()` at once and cuts off in-flight requests, or it waits on `server.close()` until the orchestrator's SIGKILL. Since Node 19, `close()` destroys the connections it considers idle at that moment. A connection still serving a request gets its response with `Connection: keep-alive` and then stays open for `keepAliveTimeout` plus `keepAliveTimeoutBuffer`, and the close callback waits for it. With the keep-alive timeout raised above the load balancer's (below), that is over a minute, past Kubernetes' default 30-second grace period. Calling `closeIdleConnections()` again does not fix it: Node counts a connection as idle once the request has been read and `end()` has been called on the response, even while that response is still being written to a slow client, so both `close()` and a sweep cut such a response off. End each connection when its response finishes, and call `close()` only when no response is mid-flush:

```ts
import type { Server, ServerResponse } from "node:http";
import { setTimeout as delay } from "node:timers/promises";

export function installShutdown(
  server: Server,
  deps: {
    log: Logger;
    markNotReady(): void;
    stopConsumers(): Promise<void>;
    closePools(): Promise<void>;
    flushLogs(): Promise<void>;
  },
  cfg: { drainDelayMs: number; deadlineMs: number },
): void {
  let stopping = false;
  let closing = false;
  const active = new Set<ServerResponse>();

  // Prepended so it runs before the app's handler has sent headers.
  server.prependListener("request", (req, res) => {
    active.add(res);
    if (stopping) res.setHeader("Connection", "close");
    res.once("finish", () => {
      if (closing) req.socket.end();
    });
    res.once("close", () => active.delete(res));
  });

  const closeServer = async (): Promise<void> => {
    closing = true;
    // close() destroys a connection whose response has ended but is still being written.
    for (;;) {
      const flushing = [...active].filter((res) => res.writableEnded && !res.writableFinished);
      if (flushing.length === 0) break;
      await Promise.all(flushing.map((res) => new Promise((resolve) => res.once("close", resolve))));
    }
    await new Promise<void>((resolve, reject) => {
      server.close((err) => (err ? reject(err) : resolve()));
    });
  };

  const shutdown = async (): Promise<void> => {
    deps.markNotReady();
    await delay(cfg.drainDelayMs);
    await Promise.all([closeServer(), deps.stopConsumers()]);
    await deps.closePools();
  };

  const exit = (): void => {
    void deps.flushLogs().then(
      () => process.exit(),
      () => process.exit(),
    );
  };

  const onSignal = (signal: NodeJS.Signals): void => {
    if (stopping) return;
    stopping = true;
    deps.log.info({ signal }, "shutdown started");
    const deadline = setTimeout(() => {
      deps.log.error({}, "shutdown deadline reached, closing remaining connections");
      process.exitCode = 1;
      server.closeAllConnections();
      exit();
    }, cfg.deadlineMs);
    deadline.unref();
    shutdown().then(exit, (err: unknown) => {
      deps.log.error({ err }, "shutdown failed");
      process.exitCode = 1;
      exit();
    });
  };

  process.once("SIGTERM", onSignal);
  process.once("SIGINT", onSignal);
}
```

What each part is for, checked on Node 26.10 with `keepAliveTimeout: 8000`:

- `'finish'` fires once the whole response has been handed to the operating system, so `socket.end()` there closes the connection without cutting the response short. Without it, a request in flight when `close()` ran held the close callback for 9 seconds after its response; with it, the process exited as the response completed.
- The `flushing` wait protects responses that are mid-write when `close()` runs. A slow client reading a 64 MiB response was reset after 11 to 20 MB in three runs where `server.close()` ran mid-write, and received the whole body with the wait in place.
- `Connection: close` on responses that start after SIGTERM tells the client and the load balancer not to reuse the connection. The listener has to be prepended: an app that answers synchronously has already sent its headers by the time a listener added with `on` runs, and `setHeader` then throws `ERR_HTTP_HEADERS_SENT`.
- The signal listener is synchronous because a rejected `async` listener is an unhandled rejection.
- At the deadline, `closeAllConnections()` lets the server close so consumers and pools still shut down, and the exit code is 1. `deadlineMs` counts from SIGTERM and has to end a few seconds before the grace period does, or SIGKILL arrives first and no handler sees it.

The language-neutral sequence and why the drain delay exists are in backend-architecture's api-and-lifecycle reference; probe configuration and signal delivery to PID 1 are in infrastructure-ops.

**`process.exit()` cutting off output.** Writes to stdout and stderr are asynchronous when they are pipes on POSIX, which is the normal case in containers. `process.exit()` right after a log line can lose the line that explained the failure. Set `process.exitCode` and let the event loop drain, or exit only after the logger flushes, as `exit()` above does.

**Signals that never arrive.** A shell-form container command (`CMD npm start` as a plain string) runs under `/bin/sh -c`, which may not pass SIGTERM on, and a PID 1 process with no SIGTERM handler ignores the signal. npm 12 forwards SIGTERM to a script's process, but every wrapper is one more place for signals and exit codes to go wrong. Use the exec form and start `node` directly, or use an init process. The infrastructure-ops skill covers container entrypoints.

### HTTP servers behind a proxy

**Keep-alive timeout shorter than the load balancer's idle timeout.** The load balancer reuses a connection the Node server has just closed and returns a 502, or the client sees `ECONNRESET`. It happens at low traffic, a few times an hour, and is hard to reproduce. `server.keepAliveTimeout` is 5 seconds on Node 26.10 and earlier, while common load balancers keep idle connections for 60 seconds or more. Set it above the proxy's idle timeout, from configuration, and check the value in the running process. `server.keepAliveTimeoutBuffer` (Node 22.19 and 24.6 and later, default 1 second) keeps the socket open that much longer than the timeout advertised to clients. An unreleased change on Node's main branch raises the default to 65 seconds; set the value explicitly either way, and make sure shutdown copes with it (above).

**Request timeouts left at defaults.** `server.requestTimeout` is 300 seconds and `server.headersTimeout` is at most 60 seconds. A client that trickles a body ties up a socket for five minutes. Set request and header timeouts for your workload, cap body size in the parser, and put deadlines on every outbound call.

### Memory

**Heap limit that does not match the container.** The process is killed by the kernel with no JavaScript error, or it spends its last minutes in garbage collection. Without flags, Node sizes the heap from the cgroup memory limit: on Node 26.10, a 512 MiB container got a 268 MiB heap and an 8 GiB container about 2 GiB, with a cap near 4 GiB on large hosts. The failures come from a hard-coded `--max-old-space-size` copied from a bigger machine, from growth outside the heap (Buffers, native modules, thread stacks, code), or from a large container where the default cap leaves memory unused. Prefer `--max-old-space-size-percentage`, which also reads the cgroup limit, over a literal MiB value, leave headroom for off-heap memory, and check `v8.getHeapStatistics().heap_size_limit` inside the container.

**Leaks that look like caches.** A module-level `Map` keyed by user or request id, memoization without eviction, a listener added per request to a long-lived emitter, a closure in a retry queue holding the whole request. Bound every cache by size and age. Confirm a leak with heap snapshots taken before and after a repeated workload, not with RSS alone.

**Reading `heapUsed` and ignoring the rest.** Buffers show up in `external` and `arrayBuffers` from `process.memoryUsage()`, not in `heapUsed`, and memory allocated by native modules may not show up anywhere except RSS. A heap that looks flat with RSS climbing usually means Buffers or a native module.

**Uninitialized Buffers.** `Buffer.allocUnsafe(n)` and `Buffer.allocUnsafeSlow(n)` return memory that may contain earlier data from the process, including secrets. Use `Buffer.alloc(n)` unless every byte is written before the Buffer leaves the function.

### Event loop and thread pool

**Four threads for everything.** The libuv pool defaults to 4 threads and serves all asynchronous `fs` calls, async `crypto` (`pbkdf2`, `scrypt`, `randomBytes`, `generateKeyPair`), `dns.lookup()` and `zlib`. A burst of password hashing or slow DNS stalls unrelated file reads and compression. Set `UV_THREADPOOL_SIZE` in the environment before the process starts; setting it from code is not guaranteed to work because the pool may already exist.

**DNS through the thread pool.** `http`, `fetch` and most clients resolve hostnames with `dns.lookup()`, which uses the pool and the OS resolver with no caching inside Node. Reuse connections with keep-alive agents so lookups happen once per connection, not once per request.

**CPU work on the main thread.** Image processing, large JSON, compression of big payloads and hashing loops block every request. Move them to `worker_threads` with a bounded pool, or out of the request path.

**No event loop measurements.** Latency spikes get blamed on the database. Export event loop delay from `perf_hooks.monitorEventLoopDelay()` (p50, p99, max) next to request latency. A p99 delay of hundreds of milliseconds means the process itself is the bottleneck.

### Request context

**Request data in module scope.** A module variable holding the current user or tenant is shared by every concurrent request. Pass identity, tenant and permissions as explicit arguments: anything that authorizes or filters data must not depend on ambient context. `AsyncLocalStorage` can lose its store (some callback-based drivers and queue clients drop it) and can hand it to work that outlives the request, such as an interval, a cached promise or a listener registered during the request. Code reading `als.getStore()?.tenantId` then gets `undefined` or another request's tenant, and an `undefined` filter can mean no filter. Use `AsyncLocalStorage`, entered once per request in the outermost middleware, for request ids, log fields and trace context, where a lost store degrades logs instead of leaking data.

### Child processes

**Missing `'error'` handler.** `spawn('missing-binary')` does not throw: it emits `'error'` with `ENOENT`, and an emitter with no `'error'` listener crashes the process. Handle `'error'`, `'exit'` and non-zero codes, and treat a signal-terminated child as a failure.

**Output limits and pipes.** `execFile` and `exec` buffer output and fail when it exceeds `maxBuffer`. A `spawn` whose stdout nobody reads can block the child once the pipe fills. Stream output, or set limits deliberately.

**Orphaned children.** Killing the parent does not kill a detached child or its descendants. Track children, kill them on shutdown, and put a timeout on every one.

### Debugging and diagnostics

**Inspector exposed to the network.** `--inspect=0.0.0.0` or a published port 9229 gives anyone who reaches it full code execution in the process. Keep the default `127.0.0.1` binding and reach production processes through an SSH tunnel or `kubectl port-forward`. On Linux and macOS, SIGUSR1 opens the inspector on a running process, so anyone allowed to signal it can open a debugger.

**Debugging the emitted JavaScript.** Breakpoints land in the wrong place because the code runs from a build. Run with `--enable-source-maps` for readable stack traces, and debug through Chrome DevTools or an IDE that follows source maps.

**No evidence after a crash.** A process that dies from heap exhaustion leaves nothing. Set `--heapsnapshot-near-heap-limit` and `--report-on-fatalerror` with a diagnostic directory on a volume you can read, and alert on the files appearing. Snapshots are as large as the heap and contain user data and secrets: treat them as sensitive.

Commands, recipes and a leak-hunting procedure: [references/debugging-and-diagnostics.md](references/debugging-and-diagnostics.md).

### Node upgrades

**Node majors treated like a library bump.** A Node major changes V8, OpenSSL, the bundled npm, HTTP defaults and the module ABI, and removes APIs and flags deprecated earlier. Native modules must be rebuilt. Read the release notes for every major you cross, update the version in every place that installs Node (`engines`, `.nvmrc`, CI, Docker base image, serverless runtime) in one change, and run the full suite plus a staging deploy.

**A new npm arriving with the toolchain.** A newer Node or CI image can bring a new npm major. npm 12 skips dependency install scripts that `allowScripts` does not list, so a native package installs cleanly and fails on first use. Policy and commands are in typescript-engineering's tooling-and-supply-chain reference.

Release schedule, what breaks between majors, and the upgrade procedure: [references/node-upgrades.md](references/node-upgrades.md).

## Decision rules

- **Version:** run a release in its LTS phase (Active or Maintenance) and move before its end-of-life date. Through Node 26 only even-numbered majors become LTS; from Node 27 there is one major a year and each becomes LTS after six months as Current. Pin it in `engines`, `.nvmrc` and the base image, and run the next LTS in CI before switching.
- **Crash or recover:** recover from expected, scoped errors inside the request that caused them. Exit on anything that reaches the process level.
- **Worker threads or a separate service:** a pool of `worker_threads` for CPU work measured in tens or hundreds of milliseconds per task; a separate service or queue when work takes seconds, needs isolation or must survive a deploy.
- **Cluster or replicas:** in containers, run one Node process per container and scale with replicas. Use `cluster` only on hosts you manage directly.
- **`NODE_ENV`:** set it to `production` in production because libraries change behavior on it, and never use it for application feature flags.
- **Permission model:** for trusted scripts that process untrusted files, run with `--permission` (stable from 22.13) and explicit `--allow-fs-read`/`--allow-fs-write` paths. From Node 25 it also blocks network access unless `--allow-net` is given. It catches mistakes in trusted code; Node's docs say malicious code can bypass it, so it is not a sandbox for untrusted code or dependencies.

## Review checklist

- Does the process exit non-zero on uncaught exceptions and unhandled rejections, after logging, without draining after `uncaughtException`?
- Does SIGTERM drain: not ready, drain delay, connections ended as their responses finish, `server.close()` only when no response is mid-flush, consumers stopped, pools closed, and a deadline inside the grace period that closes all connections?
- Is `keepAliveTimeout` above the load balancer's idle timeout, set from configuration and checked in the running process?
- Are request, header and body limits set for the workload, and does every outbound call have a deadline?
- Is the heap limit either the cgroup-derived default or a percentage, with headroom for off-heap memory, and checked inside the container?
- Is every cache bounded, is no request data stored in module scope, and are identity and tenant passed explicitly rather than read from `AsyncLocalStorage`?
- Is `UV_THREADPOOL_SIZE` set in the environment where the workload hashes, compresses or reads files heavily?
- Is CPU-heavy work off the main thread, and is event loop delay exported as a metric?
- Do child processes have `'error'` handlers, timeouts, output limits and cleanup on shutdown?
- Is the inspector bound to localhost only, and are crash diagnostics written somewhere private?
- Is the Node version in its LTS phase, pinned in every place that installs Node, and is the next upgrade planned before end of life?

## References

- [references/debugging-and-diagnostics.md](references/debugging-and-diagnostics.md): read when you need to attach a debugger, step through code, debug tests or TypeScript, profile CPU, find a memory leak, diagnose a hang or capture evidence from a crash.
- [references/node-upgrades.md](references/node-upgrades.md): read before moving to a new Node major, when reviewing that pull request, or when choosing which Node release to run.

Related skills: typescript-engineering for async correctness, validation, dependency upgrades and supply chain, Workers and Bun; backend-architecture for the shutdown sequence, retries and idempotency; infrastructure-ops for container entrypoints, probes and Kubernetes; security-engineering for authorization and threat modeling; performance-benchmarking before optimizing.
