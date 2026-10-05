---
name: performance-benchmarking
description: Profiling, benchmarks, load tests and speedup claims (perf, py-spy, Node profilers, criterion, pyperf, Go, k6, wrk2, Lighthouse). Load it before optimizing slow code, chasing p99 latency, memory growth or browser INP and frame drops, writing or reviewing a benchmark or load test, or accepting a claim that a change is faster or handles more load.
license: MIT
metadata:
  author: Alex Tsanis
---

# Performance benchmarking

Most performance work goes wrong before any code changes. Either nobody profiled, so the wrong thing got optimized, or the number that justified the change was measuring a warm cache, a constant-folded loop or a stalled load generator. Treat every speedup claim, including your own, as a hypothesis: profile first, then do a controlled comparison. A change that is faster and wrong, or faster because it dropped validation or durability, gets rejected.

## Before changing anything

- The question and the one number that answers it: p99 latency at a stated arrival rate, throughput at a latency limit, peak RSS, CPU seconds per request, cost per job.
- The workload: request mix, data volume and key distribution like production, concurrency, and cache state (cold, warm, or the production hit rate).
- The correctness gate every variant must pass: tests, output diff against the baseline, invariants.
- The baseline: revision, exact command, machine, and the raw samples saved.
- A profile of the real workload that shows where the time goes. Without one, don't optimize.

## Failure catalogue

### Measurements that lie

**Optimizing without a profile.** Someone rewrites the JSON parser, the loop or the data structure they suspect. The profile would have shown 80% of wall time waiting on the database. Amdahl applies: a function that takes 2% of request time can't give more than 2% end to end, however much faster it gets. Take a CPU profile for CPU-bound work and a trace or off-CPU view when the process is mostly waiting. Then state one hypothesis that could be proven wrong and check it. Report the end-to-end effect along with the micro result.

**The optimizer deleted the benchmark.** When the result goes unused, or the input is a compile-time constant or loop-invariant, the compiler or JIT can fold, hoist or remove the work. The symptoms are sub-nanosecond timings, time that doesn't grow with input size, or a "10x" from a trivial change. Hide inputs and consume outputs:

```rust
// Before: SAMPLE is a constant and the result is dropped
b.iter(|| checksum(SAMPLE));

// After
use std::hint::black_box;
b.iter(|| black_box(checksum(black_box(SAMPLE))));
```

Rust: `std::hint::black_box` (criterion's own `black_box` is deprecated in favor of it; divan documents `divan::black_box`). JavaScript with mitata: `do_not_optimize(value)` and computed parameters. Go: `for b.Loop() { ... }` keeps arguments and results alive (benchmark on Go 1.26+, see microbenchmarks.md). Anywhere else, fold the results into a sink that gets read after the loop. As a sanity check, doubling the input size should roughly double the time for linear work.

**Benchmarking the cache.** The benchmark repeats one input, so everything after the first iteration is a memo hit, a branch the predictor has learned, and data sitting in L1. The "3x faster" was a hash lookup. Production inputs vary, and header blocks, cookies and ids are close to unique. Benchmark a corpus of realistic distinct inputs (sampled from production and scrubbed), measure the hit and miss paths separately, and weight them by the hit rate you actually measured in production.

**Warm when users are cold, or the reverse.** The first request after a deploy, a scale-out or an idle period pays for JIT compilation, lazy imports, connection and TLS setup, and a cold database buffer cache and OS page cache. Warmed-up benchmarks measure steady state, which is right for throughput. Cold start is a separate measurement in a fresh process: hyperfine with a cache-dropping `--prepare` command before each run, or a fresh container per sample.

**Setup inside the timer, or real cost outside it.** Cloning inputs, loading fixtures or building a client inside the timed loop inflates the number. The opposite also happens: Python `timeit` disables GC during timing by default, so allocation-heavy code looks cheaper than it runs (put `gc.enable()` in the setup string), and pytest-benchmark's `--benchmark-disable-gc` does the same. Use the harness's untimed setup: criterion `iter_batched`, divan `with_inputs`, pytest-benchmark `pedantic(setup=...)`, Go `b.Loop` (which resets the timer on first call).

**Debug builds and dev modes.** `cargo run` and `cargo test` build the dev profile. `cargo bench` uses the bench profile, which inherits from release. Node frameworks change behavior when `NODE_ENV` is not `production`. Coverage, tracers and debuggers attached to Python slow everything unevenly. Confirm the build mode before believing a number.

**Averages.** A mean hides the tail and hides bimodal distributions. A healthy-looking mean can sit on top of 1% of requests taking five seconds. For latency, report sample count, p50, p99, max and error rate, and look at the histogram. For microbenchmarks, report the harness's median and confidence interval. Never average percentiles across hosts or time windows: merge the histograms, then compute. Classic Prometheus histograms interpolate inside buckets, so a quantile is only as precise as its bucket. Put a bucket boundary at the SLO threshold.

**One run, one machine, one moment.** A single run is an anecdote. Numbers from another machine, a colleague's laptop or last month's CI are not a baseline. Run baseline and candidate on the same machine in the same session, several times each, in alternating order: all of A then all of B hands the second variant a hotter chip and whatever started in the background meanwhile. Then use a statistical comparison: criterion's change report, `benchstat` or `pyperf compare_to`. pytest-benchmark's `--benchmark-compare-fail` is a plain percentage threshold with no significance test, so set it above the A/A noise you measured. A difference inside the run-to-run spread counts as no difference.

**Laptop and shared-host noise.** Turbo boost, thermal throttling, running on battery, a browser in the background, burstable cloud instances running out of credits, noisy neighbors, shared CI runners. Any of these can swing results by more than the effect being measured. Use a quiet dedicated host with a fixed frequency governor and turbo off. On Linux, `python -m pyperf system tune` does most of this and warns when the machine is on battery, but it turns off turbo only on Intel (AMD steps in environment-and-ci.md). For CI gates, see the decision rules below.

**Wall-clock timers.** `Date.now()`, `time.time()` and `SystemTime` can jump when NTP adjusts the clock, and some have coarse resolution. Use monotonic clocks: `performance.now()` or `process.hrtime.bigint()`, `time.perf_counter()`, `std::time::Instant`.

### Load tests that lie

**Coordinated omission.** In a closed-loop generator, each virtual user waits for its response before sending the next request. When the server stalls, the generator stalls with it, so the stall gets recorded once instead of once per request that would have arrived. Percentiles come out far better than users experience. k6 with `vus` and `duration`, wrk, ab and most hand-written `for` loops with `await` all behave this way. Drive an open model at the target arrival rate:

```js
// Before: 500 VUs in a closed loop; throughput falls as latency rises
export const options = { vus: 500, duration: '10m' };

// After: fixed arrival rate from config; dropped_iterations > 0 means the generator could not keep up
export const options = {
  scenarios: {
    launch_rate: {
      executor: 'constant-arrival-rate',
      rate: Number(__ENV.TARGET_RPS),
      timeUnit: '1s',
      duration: __ENV.DURATION,
      preAllocatedVUs: Number(__ENV.PREALLOCATED_VUS),
      maxVUs: Number(__ENV.MAX_VUS),
    },
  },
  thresholds: {
    http_req_failed: [`rate<${__ENV.MAX_ERROR_RATE}`],
    http_req_duration: [`p(99)<${__ENV.P99_MS}`],
    dropped_iterations: ['count<1'],
  },
  summaryTrendStats: ['med', 'p(90)', 'p(99)', 'max', 'count'],
};
```

For HTTP benchmarking, wrk2 with `-R` measures latency from when each request should have been sent. Plain wrk does not. Details in [references/load-testing.md](references/load-testing.md).

**The load generator is the bottleneck.** Warning signs: the generator on the same host as the target, generator CPU pinned, a laptop on Wi-Fi, too few connections, file descriptor or ephemeral port limits. The result is a throughput plateau while the server sits idle, and latency that belongs to the generator. Run generators on separate hosts on a network path like the real clients'. Watch the generator's CPU during the run, treat k6 `dropped_iterations` as a failed run, and add generator hosts before you conclude the server is saturated.

**Fast failures counted as fast responses.** 429s, 500s, redirects to a login page and cached error pages all come back quickly and pull percentiles down. Every request needs a check on status (and on a body marker when the body matters), and the run needs an `http_req_failed` threshold. Report error rate next to latency. A run whose error rate is over budget has no valid latency result.

**Unrealistic data and access pattern.** Staging with 200 rows where production has millions: every query plan changes and everything fits in memory. Every virtual user fetching `/products/42` gives a 100% cache hit rate, and all-unique keys give 0%. Use production-scale data (anonymized) and a key distribution taken from access logs. Include the production write mix, because writes invalidate caches and take locks.

**No steady state.** A run that is too short never fills its pools, never triggers old-generation GC, never churns caches or lets the autoscaler react. wrk2 alone spends its first 10 seconds calibrating. Discard the ramp-up, measure a plateau long enough to cover several GC cycles and cache expiries, and do separate soak runs to find leaks.

**A number without its load.** "p99 is 80 ms" at an unstated rate, or "handles 5,000 rps" at an unstated latency, means nothing. Step the arrival rate up and record p99 and error rate at each step. Capacity is the highest rate that meets the SLO, less headroom.

### Where the time usually goes

Several of these are covered in depth by the skill that owns the code. Here each entry says what the problem looks like in a profile or a measurement, and points to the owning skill for the fix.

**N+1 queries and chatty calls.** The trace shows many identical short queries or HTTP calls inside one request, and latency grows with the row count rather than with load. Fetch in one query (`WHERE id = ANY($1)`, a join) or through the API's batch endpoint, and assert query counts in tests for list endpoints. ORM specifics are in database-engineering and python-engineering.

**Unbounded parallelism.** `Promise.all` or `asyncio.gather` over a list the user controls. It looks 10x faster on 50 items. On 5,000 the measurement shows pool wait, 429s and a memory spike, not CPU, and every other request queues behind it. Batch at the source first; otherwise bound concurrency to the downstream's limit (pool size, provider quota), not the core count. Patterns are in backend-architecture and typescript-engineering.

**Blocking the event loop.** p99 rises on every endpoint of the process, including ones that do no slow work, and event loop delay climbs while other cores sit idle. The synchronous call (`pbkdf2Sync`, `JSON.parse` of a large body, `requests` inside `async def`) is the wide frame in the CPU profile. nodejs-engineering and python-engineering list the usual culprits and the fixes.

**Allocation and GC pressure.** Allocating per item in a hot loop: string concatenation, `map().filter().map()` chains over large arrays, `format!` or `to_string()` inside loops, boxing, building intermediate lists nobody keeps. The profile shows GC or allocator frames near the top. Reuse buffers, preallocate with a known capacity, stream instead of materializing. Measure allocations as well as time: divan `AllocProfiler`, Go `-benchmem`, `node --heap-prof`, Python `tracemalloc`.

**Regex backtracking.** Nested or overlapping quantifiers such as `(\w+\s?)*`, `(a|aa)+` or `.*.*=` run in exponential or polynomial time on an almost-matching input. One request pins a core, which makes this a denial-of-service bug as well as a performance one. Python `re`, JavaScript, PCRE, Java and .NET (by default) all backtrack. Rust `regex`, Go `regexp` and RE2 guarantee linear time. To fix: cap input length before matching, rewrite the pattern so each character can match only one way, use atomic groups or possessive quantifiers (Python 3.11+), or switch to a linear-time engine. Test with a long run of matching characters followed by one that fails.

**Serialization hot spots.** JSON encode and decode dominate the flame graph. Causes include responses carrying fields nobody reads, serializing and parsing again between internal layers, logging whole objects, and `serde_json::from_reader` (serde_json documents that it's usually slower than reading the input into memory and calling `from_slice`, and that it does no buffering of its own, so a bare `File` is worse still). Send less (pagination, field selection) and parse once. Switch codecs (orjson or msgspec in Python, for example) only after the profile says so and output parity is checked.

**Missing and excessive indexes.** Missing: `EXPLAIN (ANALYZE, BUFFERS)` shows a sequential scan over a large table on a hot query, which `pg_stat_statements` will surface. Excessive: every index is maintained on every insert and on updates to its columns. In PostgreSQL, updating a column covered by a B-tree index rules out a HOT update. Unused indexes show `idx_scan = 0` in `pg_stat_user_indexes`, but check every replica and the time since stats were last reset before dropping one. Derive indexes from real query shapes and measure the write path as well as the read. See database-engineering.

**Unbounded caches.** RSS climbs under steady load, and diffed heap snapshots show one map or memo growing. Keyed by values a client controls, it grows until the process is OOM-killed, and an attacker can speed that up. Give every cache a size bound and, where data changes, a TTL, and measure the hit rate to show the cache earns its memory. The npm `lru-cache` package requires at least one of `max`, `maxSize` or `ttl` for this reason; python-engineering covers `lru_cache` on methods.

**Cache stampede.** Database load spikes at the moment a hot key expires, while the cache hit rate dips. Single-flight and TTL jitter fix it; see backend-architecture.

**Retry amplification.** Request counts to a dependency multiply as soon as it slows down: three layers of three attempts send up to 27. Retry at one layer, with backoff, jitter and a budget; see backend-architecture.

**CPU limits throttling containers.** p99 jumps while average CPU looks fine, because a process with more busy threads than its limit burns the period's quota early and is frozen for the rest. If `nr_throttled` and `throttled_usec` in the cgroup's `cpu.stat` rose during the run, the result includes throttling. Size pools to the limit or change the limit; infrastructure-ops has the details, including Go 1.25+ deriving `GOMAXPROCS` from the limit.

**Logging and tracing in the hot path.** Synchronous log writes, expensive formatting done even when the level is disabled, stack traces captured per request, high-cardinality metric labels. Check these whenever logging frames show up in the profile.

### Fixes that make it worse

**Speed bought with validation, durability or safety.** Examples: Pydantic `model_construct()` on request bodies (skips validation), Rust `from_utf8_unchecked` or `get_unchecked` on untrusted bytes (undefined behavior if the assumption is wrong), `fsync=off` (PostgreSQL can corrupt data on a crash), SQLite `synchronous=OFF`, authorization decisions cached without invalidation, dropping idempotency checks or TLS verification. Reject these. The one legitimate kind of trade-off is a documented bounded loss that the data owner accepts per operation. PostgreSQL `synchronous_commit = off` on a specific transaction is an example: it can lose the last few commits on a crash but doesn't corrupt anything.

**Cache key missing an input.** A response gets cached by URL while its content varies by user, tenant, locale or permission, so one user sees another's data. Or a memo is keyed on a subset of the arguments and returns the wrong result. The key must include every input that changes the output. Personalized responses stay out of shared caches (`Cache-Control: private` or `no-store`, `Vary` where it applies).

**Parallelism that changes semantics.** Side effects that used to happen in order now happen out of order, shared state gets mutated from several tasks, or work is left half done: `Promise.all` rejects on the first failure while the other promises keep running. Before parallelizing a loop, confirm its iterations are independent, and decide what a partial failure leaves behind.

**Precision traded for speed without saying so.** Float instead of decimal for money, `-ffast-math`-style reassociation, approximate algorithms, sampling. Allowed only when the correctness gate compares output within an explicit tolerance that the owner signed off on.

## Decision rules

Which tool answers which question:

| Question | Tool |
|---|---|
| Where does CPU time go? | Sampling profiler on the real workload (perf, samply, py-spy, `node --cpu-prof`, 0x) |
| Why is it slow while CPU is idle? | Tracing across services, off-CPU or idle-inclusive profiles, `EXPLAIN ANALYZE`, pool wait metrics |
| Why does memory grow? | Heap snapshots or allocation profiles taken over time under steady load, diffed |
| Is variant B faster than A? | Microbenchmark harness with a statistical comparison, confirmed end to end |
| Will the service meet its SLO at launch load? | Open-model load test at the target arrival rate on production-like data |
| Did this PR regress? | Benchmark gate comparing base and head in the same job |
| Did the UI get slower (INP, LCP, dropped frames)? | Field p75 from web-vitals with attribution decides; a DevTools trace with calibrated CPU throttling on a production build finds the cause |

- Microbenchmark only when the profile shows one function dominating and you can reproduce its real inputs. Always confirm the win on the end-to-end workload.
- Use an open model (arrival rate) when clients arrive independently of each other: public APIs, web traffic, webhooks. Use a closed model only when a fixed pool of clients each waits for its own response, such as a batch worker pool. Even then, report the latency distribution per client.
- For latency, report p50, p99, max, count and error rate at a stated rate. For microbenchmarks, report median and confidence interval. The minimum of repeated runs is a fair statistic only for deterministic CPU-bound code in isolation, never for anything that does I/O.
- Set a concurrency limit from the downstream's capacity (pool size, rate limit, lock contention), and prefer batching over concurrency.
- Add a cache only when the key includes every input to the output, the size is bounded, invalidation is defined, stampedes are handled, and a measured hit rate justifies the memory.
- For CI regression gates, run base and head in the same job on the same runner and compare relatively. Use instruction counts (for example under Valgrind's cachegrind) when shared runners are too noisy for timing. Set the failure threshold above the measured run-to-run noise, not at a round number.
- Stop optimizing when the target is met. Further gains that users cannot feel are complexity with no payoff.

## Review checklist

- Is there a profile of the real workload, and does the change target what it shows?
- Is the baseline from the same machine, same session, same build mode, with raw samples kept?
- Are there several runs per variant and a statistical comparison, and is the claimed difference larger than the noise?
- Do benchmark inputs vary like production inputs, hidden from the optimizer, with the output consumed?
- Is setup outside the timer and real cost (GC, drop, I/O) inside it?
- Does the load test use an arrival-rate model with zero dropped iterations and a generator that is not saturated?
- Does every request in the load test check its status, and is the error rate reported and under budget?
- Are p99 and max reported at a stated load, not only mean or p95?
- Is the test data production-scale, with a realistic key distribution and write mix?
- Is every new cache bounded, keyed on every input, invalidated, and justified by a measured hit rate?
- Is every new concurrency bounded by the downstream's limit, with partial failure handled?
- Does the change keep validation, authorization, durability and idempotency intact?
- Does the correctness gate pass on the optimized path, including edge cases the fast path special-cases?
- Is the end-to-end improvement recorded next to the command and environment that produced it?

## References

- [references/microbenchmarks.md](references/microbenchmarks.md): read before writing or reviewing a benchmark with criterion, divan, pyperf, pytest-benchmark, timeit, mitata, tinybench, Go `testing` or hyperfine.
- [references/load-testing.md](references/load-testing.md): read before writing or trusting a k6 or wrk2 test, or sizing capacity for a launch.
- [references/profiling.md](references/profiling.md): read when choosing and running a profiler (perf, samply, py-spy, Node's built-in profilers, 0x, clinic.js) or chasing memory growth.
- [references/environment-and-ci.md](references/environment-and-ci.md): read when setting up a benchmark machine, deciding whether a result is noise, or adding a performance gate to CI.
- [references/browser.md](references/browser.md): read when a page or interaction got slower (INP, LCP, CLS, janky animation), before claiming a frontend speedup, or when adding Lighthouse CI or a Playwright performance gate.

Related skills: database-engineering for query plans and index design, backend-architecture for retries, idempotency and queue backpressure, infrastructure-ops for container limits and autoscaling, security-engineering when a fast path touches validation or authorization.
