# Load testing

Read this before writing a k6 or wrk2 test, before trusting a load test result, or when sizing capacity for a launch.

## Pick the model before the tool

Clients of a public service arrive independently of each other. A slow response doesn't stop the next user from clicking, so the test has to keep sending at the target rate however the server is doing. That is an open model. A closed model (N virtual users, each waiting for its response before sending again) quietly lowers the offered load when the server slows down, which is coordinated omission. The stall gets sampled once where it should have been sampled for every request that would have arrived during it.

- Open model (arrival rate): web and API traffic, webhooks, anything with many independent clients. The default.
- Closed model (fixed VUs): only when production really is a fixed pool of workers that each wait for their own response, such as a batch consumer with N workers. Report throughput along with latency, because the throughput is the thing that degrades.

## k6

### Template

```js
import http from 'k6/http';
import { check } from 'k6';
import { SharedArray } from 'k6/data';

const BASE_URL = __ENV.BASE_URL;
// Product ids sampled from production access logs, so hot and cold keys appear at their real frequency.
const productIds = new SharedArray('product ids', () => JSON.parse(open(__ENV.KEYS_FILE)));

export const options = {
  discardResponseBodies: true,
  scenarios: {
    browse: {
      executor: 'ramping-arrival-rate',
      startRate: Number(__ENV.START_RPS),
      timeUnit: '1s',
      preAllocatedVUs: Number(__ENV.PREALLOCATED_VUS),
      maxVUs: Number(__ENV.MAX_VUS),
      stages: JSON.parse(__ENV.STAGES), // e.g. steps up to and past the target rate, each held long enough to plateau
    },
  },
  thresholds: {
    http_req_failed: [`rate<${__ENV.MAX_ERROR_RATE}`],
    'http_req_duration{name:product}': [{ threshold: `p(99)<${__ENV.P99_MS}`, abortOnFail: true, delayAbortEval: '30s' }],
    dropped_iterations: ['count<1'],
    checks: [`rate>${__ENV.MIN_CHECK_PASS_RATE}`],
  },
  summaryTrendStats: ['med', 'p(90)', 'p(99)', 'p(99.9)', 'max', 'count'],
};

export default function () {
  const id = productIds[Math.floor(Math.random() * productIds.length)];
  const res = http.get(`${BASE_URL}/api/products/${id}`, { tags: { name: 'product' } });
  check(res, { 'status is 200': (r) => r.status === 200 });
}
```

### Notes

- `constant-arrival-rate` and `ramping-arrival-rate` are k6's open-model executors. `vus` with `duration`, `constant-vus`, `ramping-vus` and `per-vu-iterations` are all closed.
- `dropped_iterations` counts iterations k6 could not start because it ran out of VUs (or time). Any non-zero value means the offered load was lower than configured. Raise `maxVUs` or add generator capacity, then rerun. Never report latency from that run.
- `maxVUs` defaults to `preAllocatedVUs`. Allocating VUs mid-test costs generator CPU, so preallocate close to `target_rate × expected_p99_seconds` with headroom.
- Don't put `sleep()` in an arrival-rate scenario to simulate think time. The executor already sets the pacing, and sleep only ties up VUs.
- The default end-of-test summary shows `avg, min, med, max, p(90), p(95)`, with no p99. Set `summaryTrendStats` (or `--summary-trend-stats`) to show the percentiles your SLO is written in.
- `http_req_duration` is sending + waiting + receiving. It leaves out DNS, TCP connect and TLS handshake. When clients don't reuse connections, look at `http_req_connecting` and `http_req_tls_handshaking` too. Since k6 1.0 the end-of-test summary is compact by default and leaves those metrics out; `--summary-mode=full` shows them (k6 2.x removed the `legacy` mode and `--no-summary`).
- `http_req_failed` counts responses outside the expected statuses (200 to 399 unless changed with `http.setResponseCallback`). k6 follows redirects by default, so a redirect to a login page ends in a 200 and passes. So check the final status or a body marker on the endpoints that matter, and turn on `responseType: 'text'` for just those requests when `discardResponseBodies` is set.
- Tag requests with a stable `name` when URLs contain ids. Otherwise every URL becomes its own time series and thresholds per endpoint stop working.
- `discardResponseBodies: true` is what k6 recommends for load generation. It cuts generator memory and GC, which makes results more reliable.
- A failed threshold makes k6 exit non-zero, which is what a CI step needs.

## wrk2

```bash
wrk -t"$THREADS" -c"$CONNECTIONS" -d"$DURATION" -R"$TARGET_RPS" --latency -s paths.lua "$BASE_URL"
```

- `-R` is the target total rate, and it is required: the code exits with "Throughput MUST be specified" without it, although the README still says it defaults to 1000. wrk2's last commit is from September 2019.
- wrk2 measures latency from when each request should have been sent at the configured rate, which corrects for coordinated omission. `-U` prints the uncorrected histogram as well. The gap between the two shows how much a closed-loop tool would have hidden.
- The first 10 seconds are calibration, so runs shorter than 10 to 20 seconds say little. Run for minutes.
- Connections have to cover the rate: each connection has at most one request in flight, so `connections ≥ rate × latency` or the tool itself becomes the limit. Keep threads at or below the generator's cores.
- A single URL hits one cache entry. Use a Lua script that picks paths from a realistic key list.
- Plain `wrk` (without the 2) is closed-loop and does not correct. Don't compare its tail latencies with an SLO.

## Is the generator healthy

A run only counts if all of these hold:

- The generator runs on its own hosts, not on the machine under test, and its CPU stays well under saturation for the whole run. Watch it with the same monitoring as the servers.
- The achieved rate matches the configured rate (`dropped_iterations` is zero in k6, and the wrk2 summary rate matches `-R`).
- Nothing on the path limits the test before the service does: the load balancer's connection limits, NAT or conntrack tables, ephemeral ports, `ulimit -n` on the generator, WAF or rate limiting (allowlist the generator or the test measures the WAF).
- The network path looks like the real clients'. Same region for a regional service. A laptop on Wi-Fi is never the generator.

## Designing the run

1. Production-like environment: same instance types, replica counts, pool sizes, autoscaling limits, CPU limits and config flags. Write down every difference.
2. Production-scale data (anonymized) and key distributions taken from access logs. Include the write mix, because writes invalidate caches and take locks.
3. Third-party dependencies: use the provider's sandbox or a stub, and say which in the report. Never load test a payment provider, email service or partner API without their written agreement, and never point a load test at production without the owner's go-ahead and a kill switch.
4. Warm up, then step: hold each arrival-rate step long enough to reach steady state (pools full, several GC cycles, cache churn), then step up. Continue past the target rate to find the knee.
5. Record per step: offered rate, achieved rate, p50, p99, max, error rate, and server CPU, memory, pool wait and database load.
6. Soak separately. Hold a realistic rate for hours to catch leaks, pool exhaustion, log volume and disk growth.

## Reporting

```text
step  offered rps  achieved rps  p50 ms  p99 ms  max ms  errors  api cpu  db cpu  pool wait p99
1     500          500           22      61      140     0.00%   31%      18%     0 ms
2     1000         1000          24      75      210     0.00%   55%      34%     1 ms
3     1500         1500          29      130     480     0.02%   78%      52%     9 ms
4     2000         1994          61      890     3100    0.40%   64%      71%     240 ms
```

The reading: capacity against a p99 < 300 ms SLO sits between steps 3 and 4. At step 4, pool wait jumps while API CPU falls back, so the database connection pool is the first limit, not CPU. Put the environment, commit, data set and exact command under the table.
