# Deploys, cutovers, alerting and recovery

Read this when planning a release or rollback, a DNS or certificate cutover, or when writing alert rules, heartbeats, log handling or a restore drill.

## Release compatibility review

During a rolling deploy, and again during a rollback, version N-1 and version N run at the same time against the same state. Before a release, answer each of these for the diff:

| Surface | Question |
|---|---|
| Database schema | Can N-1 read and write every table after N's migration? (Renames and drops wait for a later release.) |
| Queue and event payloads | Can N-1 consumers process messages N produces, and can N consume what N-1 still produces? New fields optional, no new required fields, unknown types parked rather than dropped. |
| Caches and sessions | Can N-1 read cache entries, cookies and session blobs N writes? If not, version the cache key or the format. |
| Public API | Do clients already in the wild (mobile apps, SDKs, partner integrations) still work? Removal needs a deprecation window measured in client upgrade cycles. |
| Config and flags | Does N require config that N-1's environment lacks, or the reverse? Missing config must fail the deploy, not start with a default. |
| Scheduled jobs | If N changes a job, can N-1's copy run concurrently during the rollout without double work? |

A "no" anywhere means the change is split into expand and contract releases: ship readers that accept both shapes, then ship writers of the new shape, then remove the old shape once nothing produces or needs it.

## Rollback runbook requirements

A rollback path exists when all of these are true:

- The currently running artifact is identified by digest and recorded per environment, so "previous" is a lookup, not a guess.
- One documented command redeploys it (`kubectl rollout undo deployment/<name>`, the previous ECS task definition revision, the previous release in the deploy tool) and completes in minutes.
- The release's migrations were additive, so the previous version still runs against the current schema.
- Config and secrets changed in the release are either backward compatible or versioned alongside the artifact.
- Someone has run the rollback in staging for this kind of change.

When one of these cannot hold (a one-way data transformation, an external contract change), write it in the release plan with the roll-forward fix and the person on point.

## DNS and certificate cutovers

1. At least one full current-TTL period before the change, lower the record's TTL to a short value. Changing the TTL only takes effect once caches holding the old record expire.
2. Make the new target serve correctly for the same hostname before switching: certificate covers the name, health checks pass, allowlists include it.
3. Switch the record. Watch traffic arrive at the new target and fall at the old one.
4. Keep the old target serving (or proxying to the new one) until its own logs show no meaningful traffic. Some resolvers, runtimes and long-lived connections ignore or outlive the TTL.
5. Raise the TTL back.

Certificates follow the same overlap rule: the new certificate is installed and verified before the old one expires or is revoked, and any client that pins a certificate or intermediate is found before the change, not after.

## Alert rules

Symptom-based paging from an SLO with multiwindow burn rates. For a 99.9% availability SLO the error budget is 0.001; 14.4 is the burn rate that spends 2% of a 30-day budget in one hour.

```yaml
groups:
  - name: api-slo
    rules:
      - record: job:http_request_errors:ratio_rate1h
        expr: |
          sum by (job) (rate(http_requests_total{code=~"5.."}[1h]))
          / sum by (job) (rate(http_requests_total[1h]))
      - record: job:http_request_errors:ratio_rate5m
        expr: |
          sum by (job) (rate(http_requests_total{code=~"5.."}[5m]))
          / sum by (job) (rate(http_requests_total[5m]))
      - alert: ErrorBudgetFastBurn
        expr: |
          job:http_request_errors:ratio_rate1h{job="api"} > (14.4 * 0.001)
          and
          job:http_request_errors:ratio_rate5m{job="api"} > (14.4 * 0.001)
        labels:
          severity: page
        annotations:
          summary: "{{ $labels.job }} is spending its 30-day error budget over 14x too fast"
          runbook_url: https://runbooks.example.com/api/error-budget
```

The long window proves the problem is significant, the short window (1/12 of the long one) makes the alert stop soon after recovery. Add the 6x over 6h with 30m pair as a second page and 1x over 3 days with 6h as a ticket. Latency SLOs work the same way with a ratio of requests slower than the threshold, which needs a histogram with a bucket at the threshold.

Heartbeats for work that fails by not happening. The job records the time of its last success (a gauge pushed at the end of a successful run, or exported by the scheduler):

```yaml
      - alert: BackupStale
        expr: time() - backup_last_success_timestamp_seconds{job="db-backup"} > 26 * 3600
        labels:
          severity: page
      - alert: BackupHeartbeatMissing
        expr: absent(backup_last_success_timestamp_seconds{job="db-backup"})
        for: 1h
        labels:
          severity: page
```

The `absent()` rule catches the case where the job has never reported, or the metric vanished after a rename, which a threshold rule silently evaluates as "no data, no alert". Set the staleness bound to the schedule interval plus the job's normal duration plus slack.

Other silent failures and the signal to alert on:

- Certificates: days until expiry measured from outside, for example the blackbox exporter's `probe_ssl_earliest_cert_expiry - time() < 14 * 86400`. Probe every hostname clients use, including those behind CDNs and on internal services.
- Queues: age of the oldest message, not depth (SQS publishes `ApproximateAgeOfOldestMessage`; for a database-backed queue export `max(now() - created_at)` over pending rows). Depth is normal while a single poisoned message blocks a partition.
- Operational balances (gas wallets, SMS or email credits, payout float): alert on runway, `predict_linear(ops_wallet_balance[6h], 3 * 86400) < 0`, plus an absolute floor from config.
- Cron jobs and CronJobs: the heartbeat pattern above.
- Replication and CDC: lag in seconds, and a heartbeat row written on the primary and read on the replica.

## Logs

Allowlist what goes in, rather than trying to strip what should not:

```ts
function requestLogFields(req: Request, route: string) {
  return {
    method: req.method,
    route,
    requestId: req.headers.get('x-request-id'),
    userAgent: req.headers.get('user-agent'),
  };
}
```

- Log the route template (`/users/:id/orders`), not the raw URL: raw URLs carry ids, emails and sometimes tokens in the query string.
- Never log `Authorization`, `Cookie`, `Set-Cookie`, API keys, request bodies of auth, payment or profile endpoints, or full upstream responses.
- Error serializers should drop config objects and connection strings. Check what your database and HTTP client errors actually contain before logging them whole; some carry the connection URL or the request headers.
- Keep logger-level redaction of known sensitive paths as a second layer, not the only one.
- If a secret is found in logs, rotate it. Deleting the log line does not reach copies in the log pipeline, backups and third-party processors.

## Metric labels

Allowed label values come from small, fixed sets: route template, method, status class, region, queue name. Anything unbounded (user id, email, tenant id at large tenant counts, raw path, request id, error message text) goes into logs or trace attributes. When reviewing a new metric, multiply the possible values of each label; if the product is not comfortably small, the label is wrong. A single high-cardinality label added in a hot path can take down the metrics backend for every team.

## Backup restore drill

Run on a schedule, automated, and alerting on failure:

1. Pick the latest backup (and a point-in-time target, if PITR is offered) from the backup account, not from production.
2. Restore into isolated scratch infrastructure with no route to production and no production credentials.
3. Verify data: expected tables present, row counts within a band of production's, the newest row's timestamp close to the backup time, a few known records readable.
4. Record elapsed time from start to verified, and compare it with the recovery time objective. Compare the newest recovered row with the recovery point objective.
5. Tear down the scratch environment and record the result where the backup alert can see it.

Backups belong in a separate account or project with write-once or deletion-restricted storage, so a compromised production account, a bad `terraform destroy` or a ransomware actor cannot delete them. Keep the decryption keys recoverable independently of the production account.
