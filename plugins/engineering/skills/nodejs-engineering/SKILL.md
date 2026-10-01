---
name: nodejs-engineering
description: Node.js in production covering crashes and shutdown, server timeouts behind load balancers, memory limits in containers, the libuv thread pool, request context, child processes, hashing, inspector debugging, and upgrading Node and npm dependencies. Load it before writing or reviewing Node servers, workers or CLIs, before an upgrade, and when a Node process leaks, hangs, crashes or returns 502s.
license: MIT
metadata:
  author: Alex Tsanis
---

# Node.js engineering

Most Node incidents are not language bugs. They are runtime behavior nobody configured: a keep-alive timeout shorter than the load balancer's, a heap limit larger than the container, four thread-pool slots shared by DNS and file reads, a process that keeps serving after its state is corrupt, an upgrade merged because the tests were green. This skill covers the runtime, how to run it and how to upgrade it. Type design, validation, async correctness and numbers belong to the typescript-engineering skill; security review belongs to security-engineering; Ethereum hashing and contracts belong to solidity-engineering.

Check the Node version first (`node --version`, `engines`, `.nvmrc`, the Docker base image). Defaults below are for current releases; several have changed between majors.

## Failure catalogue

### Crashes and process lifecycle

**Continuing after `uncaughtException`.** A handler that logs and carries on leaves the process running with half-finished state: a transaction never committed, a lock never released, a counter never decremented. Node's own docs say the process is in an undefined state at that point. Log with full context, flush, and exit non-zero so the supervisor restarts a clean process.

**Silencing unhandled rejections.** Since Node 15 the default mode is `throw`, so an unhandled rejection ends the process. Adding `process.on('unhandledRejection', console.error)` turns that crash into silent data loss. Fix the floating promise. Keep the default, or log and exit.

**Shutdown that drops work or never finishes.** On SIGTERM the handler calls `process.exit()` at once and cuts off in-flight requests, or it calls `server.close()` and waits forever, because `close()` stops new connections but idle keep-alive sockets stay open. A correct sequence: mark the instance not ready, keep serving through a short drain delay so the load balancer stops routing, `server.close()`, `server.closeIdleConnections()` (Node 18.2+), wait for in-flight requests, stop consumers and timers, close pools, then exit. Put a hard deadline under the orchestrator's grace period that calls `server.closeAllConnections()` and exits non-zero.

**`process.exit()` cutting off output.** Writes to stdout and stderr can be asynchronous when they are pipes, which is the normal case in containers. `process.exit()` right after a log line can lose the line that explained the failure. Set `process.exitCode` and let the event loop drain, or exit only after the logger flushes.

**Signals that never arrive.** `npm start` or a shell-form container command puts a wrapper in front of Node, and SIGTERM stops there. Start `node` directly, or use an init process. The infrastructure-ops skill covers container entrypoints.

### HTTP servers behind a proxy

**Keep-alive timeout shorter than the load balancer's idle timeout.** The load balancer reuses a connection the Node server has just closed and returns a 502 or the client sees `ECONNRESET`. It happens at low traffic, a few times an hour, and is hard to reproduce. `server.keepAliveTimeout` defaults to 5 seconds in current releases, while common load balancers keep idle connections for 60 seconds or more. Set it above the proxy's idle timeout, from configuration, and verify the effective value in the running process. Node 22.19 and 24.6 added `server.keepAliveTimeoutBuffer`, and Node's main branch raises the keep-alive default to 65 seconds, so check your version before relying on any default.

**Request timeouts left at defaults.** `server.requestTimeout` is 300 seconds and `server.headersTimeout` is at most 60 seconds. A client that trickles a body ties up a socket for five minutes. Set request and header timeouts for your workload, cap body size in the parser, and put deadlines on every outbound call.

**Trusting forwarded headers.** `X-Forwarded-For` and `X-Forwarded-Proto` are client-writable unless your proxy overwrites them. Trust exactly the number of proxy hops you run.

### Memory

**Heap limit that does not match the container.** The process is killed by the kernel with no JavaScript error, or it spends its last minutes in garbage collection. Check the limit the process actually got (`v8.getHeapStatistics().heap_size_limit`) inside the container. Set `--max-old-space-size` (MiB) or `--max-old-space-size-percentage` explicitly, and leave headroom under the container limit for memory outside the V8 heap: Buffers, native modules, thread stacks and code.

**Leaks that look like caches.** A module-level `Map` keyed by user or request id, memoization without eviction, a listener added per request to a long-lived emitter, a closure in a retry queue holding the whole request. Bound every cache by size and age. Confirm a leak with heap snapshots taken before and after a repeated workload, not with RSS alone.

**Reading `heapUsed` and ignoring the rest.** Buffers show up in `external` and `arrayBuffers` from `process.memoryUsage()`, not in `heapUsed`, and memory allocated by native modules may not show up anywhere except RSS. A heap that looks flat with RSS climbing usually means Buffers or a native module.

**Uninitialized Buffers.** `Buffer.allocUnsafe(n)` and `Buffer.allocUnsafeSlow(n)` return memory that may contain earlier data from the process, including secrets. Use `Buffer.alloc(n)` unless every byte is written before the Buffer leaves the function.

### Event loop and thread pool

**Four threads for everything.** The libuv pool defaults to 4 threads and serves all asynchronous `fs` calls, async `crypto` (`pbkdf2`, `scrypt`, `randomBytes`, `generateKeyPair`), `dns.lookup()` and `zlib`. A burst of password hashing or slow DNS stalls unrelated file reads and compression. Set `UV_THREADPOOL_SIZE` in the environment before the process starts; setting it from code is not guaranteed to work because the pool may already exist.

**DNS through the thread pool.** `http`, `fetch` and most clients resolve hostnames with `dns.lookup()`, which uses the pool and the OS resolver with no caching inside Node. Reuse connections with keep-alive agents so lookups happen once per connection, not once per request.

**CPU work on the main thread.** Image processing, large JSON, compression of big payloads and hashing loops block every request. Move them to `worker_threads` with a bounded pool, or out of the request path.

**No event loop measurements.** Latency spikes get blamed on the database. Export event loop delay from `perf_hooks.monitorEventLoopDelay()` (p50, p99, max) next to request latency. A p99 delay of hundreds of milliseconds means the process itself is the bottleneck.

### Request context

**Request data in module scope.** A module variable holding the current user or tenant is shared by every concurrent request. Carry it with `AsyncLocalStorage`, entered once per request in the outermost middleware, and pass explicit arguments where the call path is short. Check that context survives your database driver and queue client, since some callback-based libraries lose it.

### Child processes

**Missing `'error'` handler.** `spawn('missing-binary')` does not throw: it emits `'error'` with `ENOENT`, and an emitter with no `'error'` listener crashes the process. Handle `'error'`, `'exit'` and non-zero codes, and treat a signal-terminated child as a failure.

**Output limits and pipes.** `execFile` and `exec` buffer output and fail when it exceeds `maxBuffer`. A `spawn` whose stdout nobody reads can block the child once the pipe fills. Stream output, or set limits deliberately.

**Orphaned children.** Killing the parent does not kill a detached child or its descendants. Track children, kill them on shutdown, and put a timeout on every one.

### Crypto and hashing

**Hashing or signing `JSON.stringify` output.** Key order and number formatting are not canonical, so two services hash the same object differently. Sign a canonical encoding (a fixed field order, RFC 8785 JSON canonicalization, or EIP-712 typed data for wallet signatures), never ad hoc JSON.

**Encoding mismatches.** `Buffer.from(str)` assumes UTF-8; hex strings need `'hex'`, and `'base64'` and `'base64url'` are different alphabets. Hashing `'0xabc...'` as text instead of as bytes is a common source of mismatched signatures. Decide the byte representation once, at the boundary.

### Debugging and diagnostics

**Inspector exposed to the network.** `--inspect=0.0.0.0` or a published port 9229 gives anyone who reaches it full code execution in the process. Keep the default `127.0.0.1` binding and reach production processes through an SSH tunnel or `kubectl port-forward`. On Linux and macOS, SIGUSR1 opens the inspector on a running process, so anyone allowed to signal it can open a debugger.

**Debugging the emitted JavaScript.** Breakpoints land in the wrong place because the code runs from a build. Run with `--enable-source-maps` for readable stack traces, and debug through Chrome DevTools or an IDE that follows source maps.

**No evidence after a crash.** A process that dies from heap exhaustion leaves nothing. Set `--heapsnapshot-near-heap-limit` and `--report-on-fatalerror` with a diagnostic directory on a volume you can read, and alert on the files appearing. Snapshots are as large as the heap and contain user data and secrets: treat them as sensitive.

Commands, recipes and a leak-hunting procedure: [references/debugging-and-diagnostics.md](references/debugging-and-diagnostics.md).

### Upgrades

**Upgrading everything at once.** One pull request bumps Node, the framework, the ORM and forty libraries. Something breaks, and nobody can tell which change did it or roll back one part. Upgrade one major version of one thing at a time, behind passing tests, and deploy it before starting the next.

**Trusting semver.** Minor and patch releases change behavior: a default, a timeout, an error type, a transitive dependency. `0.x` packages promise nothing. Read the changelog and the migration guide for every version you skip, and treat "it compiles" as the start of testing, not the end.

**Node majors treated like a library bump.** A Node major changes the bundled OpenSSL, V8, npm, default timeouts and deprecated APIs, and native modules must be rebuilt. Read the release notes for every major you cross, update the version in every place that installs Node (`engines`, `.nvmrc`, CI, Docker base image, serverless runtime), and run the full suite plus a staging deploy.

**Silencing conflicts instead of resolving them.** `--legacy-peer-deps`, `--force` or an override that pins a transitive version makes the install pass and leaves two incompatible copies at runtime, or a library running against a peer it was never tested with. Resolve the conflict, or document the override with the reason and a date to remove it.

**Adopting releases the day they ship.** Compromised packages are usually caught within days. Give new versions a waiting period before automated updates pick them up (Renovate `minimumReleaseAge`, Dependabot `cooldown`), except for security fixes.

Planning, commands, codemods, automation and rollback: [references/dependency-upgrades.md](references/dependency-upgrades.md).

## Decision rules

- **Version:** run an active LTS release (even-numbered majors) in production, pin it in `engines`, `.nvmrc` and the base image, and test the next LTS in CI before the current one reaches end of life.
- **Crash or recover:** recover from expected, scoped errors inside the request that caused them. Exit on anything that reaches the process level.
- **Worker threads or a separate service:** a pool of `worker_threads` for CPU work measured in tens or hundreds of milliseconds per task; a separate service or queue when work takes seconds, needs isolation or must survive a deploy.
- **Cluster or replicas:** in containers, run one Node process per container and scale with replicas. Use `cluster` only on hosts you manage directly.
- **`NODE_ENV`:** set it to `production` in production because libraries change behavior on it, and never use it for application feature flags.
- **Permission model:** for scripts that process untrusted files, consider `--permission` (stable since 22.13 and 23.5) to restrict filesystem, child process and worker access.

## Review checklist

- Does the process exit non-zero on uncaught exceptions and unhandled rejections, after logging?
- Does SIGTERM drain: not ready, drain delay, close the listener, close idle connections, finish in-flight work, close pools, hard deadline under the grace period?
- Is `keepAliveTimeout` above the load balancer's idle timeout, set from configuration and checked in the running process?
- Are request, header and body limits set for the workload, and does every outbound call have a deadline?
- Is the heap limit set explicitly, with headroom under the container limit for off-heap memory?
- Is every cache bounded, and is no request data stored in module scope?
- Is `UV_THREADPOOL_SIZE` set in the environment where the workload hashes, compresses or reads files heavily?
- Is CPU-heavy work off the main thread, and is event loop delay exported as a metric?
- Do child processes have `'error'` handlers, timeouts, output limits and cleanup on shutdown?
- Is every override, forced install or skipped peer dependency documented with a reason and removal date?
- Are signatures and hashes computed over a canonical encoding?
- Is the inspector bound to localhost only, and are crash diagnostics written somewhere private?
- Is the Node version an active LTS, pinned in every place that installs Node?
- Does each upgrade change one major version of one thing, with its changelog read and a rollback ready?

## References

- [references/debugging-and-diagnostics.md](references/debugging-and-diagnostics.md): read when you need to attach a debugger, step through code, debug tests or TypeScript, profile CPU, find a memory leak, diagnose a hang or capture evidence from a crash.
- [references/dependency-upgrades.md](references/dependency-upgrades.md): read before upgrading Node, a framework or a major dependency, when reviewing an upgrade pull request, or when configuring Renovate or Dependabot.
