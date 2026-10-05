# Profiling

Read this when choosing a profiler, running one against a real workload, reading a flame graph, or chasing memory growth.

## Decide what kind of slow it is

Before picking a tool, look at CPU usage while the slow thing runs.

- The process is near 100% of a core (or of its cgroup limit): on-CPU work. Use a sampling CPU profiler.
- CPU is low but requests are slow: the process is waiting on a database, a lock, a pool, the network, disk or a throttled cgroup. A CPU profile will show idle frames or nothing useful. Use tracing across services, database statistics, pool wait metrics, idle-inclusive sampling (py-spy `--idle`), and cgroup `cpu.stat` for throttling.
- Memory climbs under steady load: take heap snapshots or allocation profiles at intervals and diff them.
- Latency is fine on average with bad spikes: correlate the spikes with GC logs, cgroup throttling, cron jobs, cache expiry, autovacuum and deploys before profiling code.

Profile the real workload (production, or a load test with production-shaped traffic) for long enough to cover the slow path. A profile of a unit test shows the unit test.

## Linux perf

```bash
perf record -F "$HZ" -g -p "$PID" -- sleep "$DURATION_S"  # sample a running process
perf record -F "$HZ" --call-graph dwarf -- ./target/release/app "$@"   # when frame pointers are missing
perf report
perf stat -e cycles,instructions,cache-misses,branch-misses -- ./target/release/app "$@"
```

- Stacks need frame pointers or DWARF unwinding. For Rust, set `debug = true` (or `"line-tables-only"`) in `[profile.release]` or `[profile.bench]` for symbols, and build with `RUSTFLAGS="-C force-frame-pointers=yes"` for cheap `-g` stacks. Debug info doesn't change the optimization level.
- JIT runtimes need a symbol map. Node: `node --perf-basic-prof app.js`. Python 3.12+ on Linux: `python -X perf app.py` or `PYTHONPERFSUPPORT=1`, which makes Python functions show up in perf stacks.
- `perf stat` counts answer "is it memory bound": low instructions per cycle together with many cache misses points at data layout and access pattern, not at instruction count.
- Access depends on `kernel.perf_event_paranoid`. Changing it is a host-wide security decision, so ask the machine owner. Don't lower it on shared production hosts on your own initiative.

## samply

```bash
samply record -- ./target/release/app "$@"
samply record -p "$PID"                         # attach to a running process
samply record --save-only -o "$PROFILE_OUT" -- ./target/release/app "$@"   # write the file, skip the local server
```

samply samples at 1000 Hz by default (`-r` changes it) and opens the result in the Firefox Profiler, served locally. On Linux it uses perf events, so the same `perf_event_paranoid` rules apply, and it records on-CPU samples only. On macOS and Windows it also captures off-CPU samples. It's a good default for native code (Rust, C, C++) because the call tree, flame graph and source view are all in one UI.

## py-spy (Python)

```bash
py-spy record -o profile.svg --pid "$PID"                     # flame graph from a live process
py-spy record --format speedscope -o profile.json -- python -m app
py-spy record --idle --subprocesses -o profile.svg --pid "$PID"   # include waiting threads; follow worker processes
py-spy dump --pid "$PID"                                      # what is every thread doing right now
py-spy top --pid "$PID"
```

- It samples from outside the process with no code changes and low overhead, so it's safe to use against a production worker for a bounded time.
- `--subprocesses` is needed for gunicorn, uvicorn with workers, and multiprocessing, or you profile the idle parent process.
- `--idle` includes threads that aren't running. Use it when the question is "why is it waiting". `--gil` shows only samples holding the GIL. `--native` adds C extension frames.
- `dump` on a hung process is often faster than any profile.
- py-spy reads CPython's internal structures, so every new Python minor version needs a py-spy release that knows its layout. 0.4.2 is the first that supports 3.14. When py-spy cannot read a process, compare the two versions before anything else.
- Attaching needs ptrace: root or a relaxed `ptrace_scope` on Linux, `--cap-add SYS_PTRACE` in Docker (the `SYS_PTRACE` capability in Kubernetes), and sudo on macOS. Ask before changing security settings on a shared host.

Python memory: `tracemalloc` shows where live allocations came from.

```python
import tracemalloc

tracemalloc.start(TRACEBACK_FRAMES)
before = tracemalloc.take_snapshot()
run_workload()
after = tracemalloc.take_snapshot()
for stat in after.compare_to(before, "traceback")[:TOP_N]:
    print(stat)
    for line in stat.traceback.format():
        print(line)
```

## Node.js

```bash
node --cpu-prof --cpu-prof-dir=./profiles server.js          # written only on a normal exit; open in Chrome DevTools
node --heap-prof --heap-prof-dir=./profiles server.js        # sampling heap profile, same exit rule
node --heapsnapshot-signal=SIGUSR2 server.js                 # `kill -USR2 <pid>` writes a heap snapshot
node --heapsnapshot-near-heap-limit=3 server.js              # snapshots before an OOM
0x -o server.js                                              # flame graph; Node 20+
0x -P 'autocannon localhost:$PORT' server.js                 # start load once the server listens
```

- `--cpu-prof` and `--heap-prof` are built in and stable in current Node LTS lines. They are the first choice because they need nothing installed.
- Both write their file only when the process exits normally: the event loop empties or something calls `process.exit()`. A server stopped with Ctrl+C or SIGTERM and no handler for that signal dies without writing anything (on Node 26.10 it exits with 130 or 143 and the profile directory stays empty), so a ten-minute run under load is lost. Before a long run, check the server handles both signals by closing the listener and idle connections so the loop drains, the same shutdown nodejs-engineering describes:

  ```js
  for (const signal of ['SIGINT', 'SIGTERM']) {
    process.once(signal, () => {
      server.close();
      server.closeIdleConnections();
    });
  }
  ```

  Stop the load generator before sending the signal, so open connections go idle. A fallback timer that calls `process.exit()` when the drain hangs still gets the profile written. 0x catches SIGINT and SIGTERM itself and writes the flame graph, so it needs none of this.
- Clinic.js (`clinic doctor`, `flame`, `bubbleprof`, `heapprofiler`) is no longer actively maintained (the last release, 13.0.0, is from June 2023), and its README warns that results may be inaccurate on current Node. Use it for a hint if it runs. Get the evidence from the built-in profilers or 0x.
- Leak hunting: take a snapshot after warm-up, run steady load, take a second, run more load, take a third. Objects allocated between the first and second snapshots that are still retained in the third are the suspects. Sort by retained size and follow the retainer path to the module-level `Map`, the listener that never gets removed, or the closure holding a request.
- A heap snapshot blocks the event loop and needs about twice the current heap size in memory, so take one only on an instance out of rotation. Snapshots contain whatever was in memory: tokens, session data, personal data. Treat the files as secrets, keep them off shared drives and delete them when done.

## Databases

- `pg_stat_statements` ordered by `total_exec_time` shows which queries cost the most in aggregate. A 2 ms query run 40,000 times per minute beats the 3 s report nobody runs.
- `EXPLAIN (ANALYZE, BUFFERS)` on production-sized data. Plans on a small staging table say nothing about production. `ANALYZE` executes the statement, so wrap `INSERT`, `UPDATE` and `DELETE` in `BEGIN; ... ROLLBACK;`, and don't run it against production writes at all without the owner's approval.
- Compare estimated rows with actual rows. A large mismatch means stale statistics or correlated columns, and the fix is `ANALYZE` or extended statistics, not a new index.
- See database-engineering for index design and locking.

## Reading a flame graph

- Width is time (samples). Left-to-right order is alphabetical or arbitrary, not chronological.
- Look for the widest frames near the top that are your code (self time), then walk down to see who calls them.
- A wide `epoll_wait`, `futex`, `select` or idle frame means waiting. Switch to off-CPU or tracing tools.
- A flat profile with no dominant frame usually means allocation and abstraction overhead spread everywhere. Check the allocation profile and GC time before micro-tuning individual functions.
- Wide GC, `malloc` or `free` frames: allocation pressure. Find the allocating call sites with an allocation profiler, not the CPU profile.
- Big regex, JSON or logging frames are the usual cheap wins. See the catalogue in SKILL.md.

Keep the before and after profiles next to the benchmark results. The before profile justifies the change, and the after profile shows the hot spot moved where you expected.
