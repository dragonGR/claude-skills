# Debugging and diagnostics

Read this when you need to stop a Node process at a line, see why a value is wrong, debug a test or TypeScript, profile CPU, find a memory leak, explain a hang, or keep evidence from a crash.

Pick the lightest tool that answers the question. A log line answers "what was the value here" in a minute. A debugger answers "how did it get here" and "what else is in scope". A profile answers "where does the time go". A heap snapshot answers "what is holding the memory". Guessing from source code answers none of them reliably.

## Safety first

The inspector protocol gives whoever connects full code execution inside the process, with its credentials and network access.

- Keep the default binding, `127.0.0.1:9229`. Never pass `--inspect=0.0.0.0` and never publish the port from a container.
- For a remote or production process, open the port locally on that host and reach it through `ssh -L 9229:127.0.0.1:9229 host` or `kubectl port-forward pod/<name> 9229`.
- A paused process serves no traffic. On a production instance, take it out of the load balancer first, or use probe mode, profiles and snapshots, which do not stop at breakpoints.
- Heap snapshots and diagnostic reports contain request data, tokens and keys. Write them to a private directory, move them with the same care as a database dump, and delete them when done.

## Starting with the inspector

```sh
node --inspect server.js              # listen, keep running
node --inspect-brk server.js          # listen and pause before the first line
node --inspect=127.0.0.1:0 server.js  # pick a free port, printed on startup
node --inspect-wait server.js         # listen and wait for a debugger before running
```

Use `--inspect-brk` or `--inspect-wait` when the bug happens at startup: with plain `--inspect`, the code you care about often runs before you attach.

For several inspectable processes, list targets with `curl -s http://127.0.0.1:9229/json/list` and attach by the `webSocketDebuggerUrl`.

`NODE_OPTIONS='--inspect'` propagates to child processes, and each child needs its own port. Prefer passing the flag only to the process you care about.

## Attaching to a process that is already running

On Linux and macOS, SIGUSR1 opens the inspector on a running process:

```sh
kill -USR1 <pid>
node inspect -p <pid>
```

The process prints the WebSocket URL to stderr. Inside a container, run both commands in the same container or network namespace.

## The `node inspect` command line

```sh
node inspect script.js
node inspect -p <pid>
node inspect 127.0.0.1:9229
```

| Command | Effect |
| --- | --- |
| `c`, `cont` | continue |
| `n`, `next` | step over |
| `s`, `step` | step into |
| `o`, `out` | step out |
| `pause` | pause running code; use it to find where a hang is |
| `sb(line)` | breakpoint on a line of the current file |
| `sb('file.js', line)` | breakpoint in a file |
| `sb('file.js', line, 'cond')` | conditional breakpoint |
| `sb('fn()')` | break on the first statement of a function |
| `cb('file.js', line)` | clear a breakpoint |
| `breakpoints` | list breakpoints |
| `bt` | backtrace |
| `list(5)` | source with five lines of context |
| `watch('expr')`, `watchers` | evaluate expressions on every pause |
| `exec expr`, `p expr` | evaluate once in the paused frame |
| `repl` | open a REPL in the paused scope (Ctrl+C returns to `debug>`) |
| `profile`, `profileEnd`, `profiles[0].save()` | record and save a CPU profile |
| `takeHeapSnapshot()` | write a heap snapshot |
| `restart`, `kill`, `.exit` | restart, stop the script, leave |

If you leave `node inspect` while the target is paused, the target stays paused. Continue or kill it before you quit.

## Probe mode (Node 24.16+ or 26.1+)

Probe mode starts the script, evaluates expressions every time execution reaches a location, prints the values and exits. There is no interactive session, which makes it the right tool from scripts, CI and AI agents.

```sh
node inspect --probe app.js:42 --expr 'order.total' app.js
node inspect --probe app.js:42 --expr 'order.total' --cond 'order.total < 0' --max-hit 5 app.js
node inspect --json --probe app.js:42 --expr 'order' --preview -- app.js
```

`--cond` (24.20+ or 26.6+) records a hit only when the condition is truthy; `--max-hit` (24.19+ or 26.4+) stops after that many hits. Node 22 has no probe mode. Probe mode launches a new process from the entry script; it does not attach to a running one.

## Chrome DevTools and IDEs

Open `chrome://inspect`, add `127.0.0.1:9229` under Configure, and click Inspect, or use the "Open dedicated DevTools for Node" link. DevTools follows source maps, so you can set breakpoints in the original TypeScript. VS Code and WebStorm attach to the same port.

## TypeScript

Breakpoints and stack traces refer to the emitted JavaScript unless source maps are in play.

- Build with source maps (`"sourceMap": true`, or the bundler's option) and run with `--enable-source-maps` so stack traces point at `.ts` lines. Source maps cost some memory and startup time; measure before enabling them in a hot production path.
- To debug TypeScript run through tsx: `node --inspect-brk --import tsx src/main.ts`.
- `node inspect` on its own shows emitted JavaScript. Use DevTools or an IDE when you need to step through the original source.

## Debugging tests

Run a single test file in a single worker, paused before the first line, then attach:

```sh
node --inspect-brk ./node_modules/vitest/vitest.mjs run --no-file-parallelism src/orders.test.ts
node --inspect-brk ./node_modules/jest/bin/jest.js --runInBand src/orders.test.ts
node --inspect-brk --test --test-isolation=none src/orders.test.js
node --inspect-brk --test --experimental-test-isolation=none src/orders.test.js   # Node 22
```

`--test-isolation=none` runs the test files in the runner's own process so the debugger sees them. Node 23.6 renamed it from `--experimental-test-isolation`, and Node 22 still only accepts the old name.

A pool of workers each with its own inspector is not worth fighting. If the failure only shows under parallel runs, that is a shared-state or ordering bug: look for module-level state, shared database rows and real timers before reaching for the debugger.

## Finding a hang

1. Start with `--inspect`, reproduce until the process stops making progress, attach, and run `pause`, then `bt`. A busy loop shows its stack. If nothing is on the stack, the process is waiting.
2. A process that waits forever is usually blocked on a promise that never settles: a request with no timeout, a lock never released, a queue consumer with no messages, a stream nobody reads. Find the last log line, then look for the awaited call without a deadline.
3. A process that will not exit after its work is done has something keeping the event loop alive: an open server, a pool, an interval, a socket. Close resources on shutdown instead of calling `process.exit()` to hide it.
4. Measure event loop delay with `perf_hooks.monitorEventLoopDelay()`. High delay with low CPU on other cores points at synchronous work on the main thread.

## CPU profiles

```sh
node --cpu-prof --cpu-prof-dir=./profiles server.js
```

The profile is written when the process exits normally. Drive the workload, stop the process with SIGINT or SIGTERM handled by a clean shutdown, and open the `.cpuprofile` in Chrome DevTools (Performance panel, or the Profiler in the Node DevTools window). For a running process, attach and use `profile` and `profileEnd` in `node inspect`, or record from DevTools. Profile in an environment that resembles production; a profile from a laptop with a warm cache and no load misleads.

## Finding a memory leak

1. Confirm there is a leak. Graph `heapUsed`, `external` and RSS over hours under steady load. A sawtooth that returns to the same floor is normal garbage collection. A rising floor is a leak or an unbounded cache.
2. Take three heap snapshots: after warm-up, after a fixed amount of the suspect workload, and after the same amount again. Use `takeHeapSnapshot()` in `node inspect`, DevTools, `v8.writeHeapSnapshot()` behind an admin-only trigger, or start with `--heapsnapshot-signal=SIGUSR2` and send the signal.
3. Load them in DevTools (Memory panel). Compare the second and third with the Comparison view and sort by size delta. Objects that grow by the same amount each round are the leak.
4. Open the retainers of one leaked object and follow the path to a GC root. The usual answers: a module-level `Map` or array, a listener on a long-lived emitter, a timer or interval, a closure in a retry or cache structure, or a request object referenced from a log or metrics buffer.
5. Fix the owner (bound it, evict, remove the listener, clear the timer) and repeat the measurement to show the floor no longer rises.

Taking a snapshot pauses the process, and Node's documentation says it needs memory about twice the size of the heap. A process close to its limit can be killed by the snapshot itself. Take snapshots on an instance that is out of rotation, with memory to spare.

`--heap-prof` writes a sampling heap profile on exit and is cheaper than snapshots when the question is "which code allocates the most".

## Keeping evidence from crashes

```sh
node --heapsnapshot-near-heap-limit=1 \
     --report-on-fatalerror \
     --diagnostic-dir=/var/diagnostics/node \
     server.js
```

- `--heapsnapshot-near-heap-limit=n` writes up to `n` snapshots as the heap approaches its limit, which shows what filled it.
- `--report-on-fatalerror` writes a diagnostic report (JSON with JavaScript and native stacks, heap statistics, resource usage, environment and loaded libraries) on fatal errors such as out-of-memory. `process.report.writeReport()` writes one on demand, and `--report-uncaught-exception` covers uncaught exceptions.
- `--diagnostic-dir` sets where these files go. Point it at a mounted volume so the files survive the container restart, keep it private, and alert when new files appear.

Reports include environment variables. Check what your deployment puts in the environment before collecting them, and never attach them to public issues.

## Scripting the inspector

When you need the same breakpoint and state capture many times, drive the Chrome DevTools Protocol from a script with `chrome-remote-interface` (install it outside the project if you do not want a new dependency):

```js
import CDP from 'chrome-remote-interface';

const INSPECTOR_PORT = Number(process.env.INSPECTOR_PORT ?? 9229);
const TARGET_URL_PATTERN = process.env.TARGET_URL_PATTERN;
const TARGET_LINE = Number(process.env.TARGET_LINE);
const EXPRESSION = process.env.EXPRESSION;

const client = await CDP({ port: INSPECTOR_PORT });
const { Debugger, Runtime } = client;

Debugger.paused(async ({ callFrames }) => {
  const [frame] = callFrames;
  const { result } = await Debugger.evaluateOnCallFrame({
    callFrameId: frame.callFrameId,
    expression: EXPRESSION,
    returnByValue: true,
  });
  console.log(`${frame.url}:${frame.location.lineNumber + 1}`, result.value ?? result.description);
  await Debugger.resume();
});

await Runtime.enable();
await Debugger.enable();
await Debugger.setBreakpointByUrl({ urlRegex: TARGET_URL_PATTERN, lineNumber: TARGET_LINE - 1 });
await Runtime.runIfWaitingForDebugger();
```

CDP line numbers are zero-based, which is why the script subtracts one. On Node 24.16+ or 26.1+, probe mode usually does the same job without a script.
