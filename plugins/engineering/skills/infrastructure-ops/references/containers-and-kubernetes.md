# Containers and Kubernetes

Read this when writing or reviewing a Dockerfile, an entrypoint script, a Deployment, probe or resource settings, a pod `securityContext`, or the way migrations run.

## Dockerfile

Before: every common problem in one file.

```dockerfile
FROM node:latest
WORKDIR /app
ARG NPM_TOKEN
COPY . .
RUN echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc && npm install && rm .npmrc
RUN npm run build
CMD npm run start:prod
```

- `node:latest` changes under you and differs between build hosts.
- `NPM_TOKEN` is visible in `docker history`, and `.npmrc` is present in the layer written by that `RUN` even though it is deleted afterwards. `COPY . .` also brings in `.env`, `.git` and anything else in the context.
- `npm install` can rewrite the lockfile when it disagrees with `package.json`, where `npm ci` fails instead. Dev dependencies, sources and the build toolchain all ship to production.
- Runs as root.
- Shell-form `CMD` through `npm`: SIGTERM never reaches Node, every stop ends in SIGKILL.

After:

```dockerfile
# syntax=docker/dockerfile:1

# name:tag@sha256:digest, supplied by the build and kept current by the update bot.
# An empty value fails the build rather than falling back to a floating tag.
ARG NODE_IMAGE

FROM ${NODE_IMAGE} AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
COPY . .
RUN npm run build

FROM ${NODE_IMAGE} AS prod-deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci --omit=dev

FROM ${NODE_IMAGE}
ENV NODE_ENV=production
WORKDIR /app
COPY --from=prod-deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json ./
# The official node image's "node" user. Numeric so Kubernetes runAsNonRoot can verify it.
USER 1000:1000
CMD ["node", "dist/server.js"]
```

Build with `docker build --secret id=npmrc,src="$NPMRC_PATH" --build-arg NODE_IMAGE="$NODE_IMAGE" .`. Pair it with a `.dockerignore` that excludes at least `.git`, `.env*`, `node_modules`, local build output, key files and Terraform state.

Files copied in stay owned by root and are read-only to UID 1000, which is what you want: the app should not be able to rewrite its own code.

The app itself must handle SIGTERM (drain, close, exit). If it spawns child processes, add an init as PID 1: install `tini` in the image and use `ENTRYPOINT ["tini", "--"]`, or rely on the runtime's init option where the platform provides one.

## Entrypoint scripts

Before:

```sh
#!/bin/sh
npx prisma migrate deploy
node dist/server.js
```

Two defects. Every replica runs the migration on every start (see "Migrations" below), and `node` runs as a child of `sh`, which does not forward SIGTERM.

After, when a wrapper is needed at all (for example to assemble config from files):

```sh
#!/bin/sh
set -eu
exec node dist/server.js "$@"
```

`exec` replaces the shell with Node, so Node becomes PID 1 and receives signals directly.

## Deployment manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-api
spec:
  replicas: {{ .Values.replicas }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 25%
  selector:
    matchLabels: { app: orders-api }
  template:
    metadata:
      labels: { app: orders-api }
    spec:
      terminationGracePeriodSeconds: {{ .Values.terminationGracePeriodSeconds }}
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        seccompProfile: { type: RuntimeDefault }
      containers:
        - name: api
          image: "{{ .Values.image.repository }}@{{ .Values.image.digest }}"
          ports:
            - { name: http, containerPort: {{ .Values.port }} }
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities: { drop: ["ALL"] }
          volumeMounts:
            - { name: tmp, mountPath: /tmp }
          startupProbe:
            httpGet: { path: /livez, port: http }
            periodSeconds: 5
            failureThreshold: 30
          livenessProbe:
            httpGet: { path: /livez, port: http }
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet: { path: /readyz, port: http }
            periodSeconds: 5
            failureThreshold: 2
          resources:
            requests:
              cpu: {{ .Values.resources.cpuRequest }}
              memory: {{ .Values.resources.memory }}
            limits:
              memory: {{ .Values.resources.memory }}
      volumes:
        - { name: tmp, emptyDir: {} }
```

Points a reviewer should check:

- The image is referenced by digest, so `imagePullPolicy` does not matter for correctness: the digest cannot point at different content.
- `maxUnavailable: 0` keeps full capacity during the rollout; old pods go away only as new ones pass readiness.
- The startup probe allows `periodSeconds * failureThreshold` for boot, then liveness takes over with a short threshold.
- No CPU limit, memory limit equal to request. If policy requires a CPU limit, set it well above the request and watch throttling.
- `terminationGracePeriodSeconds` must exceed the app's drain delay plus its close timeout plus pool shutdown.

## Probe endpoints

```ts
app.get('/livez', (_req, res) => {
  res.status(workerHeartbeat.isStale() ? 503 : 200).end();
});

app.get('/readyz', (_req, res) => {
  res.status(lifecycle.acceptingTraffic() ? 200 : 503).end();
});
```

`/livez` touches nothing outside the process. `workerHeartbeat` is updated by the background loop each iteration; a loop stuck for longer than its configured bound is a real reason to restart. `/readyz` turns false at the start of shutdown and until startup work (config, pool warm-up, cache load) is done. Add a dependency check to readiness only when this pod can fail independently of its siblings (a per-pod connection pool that can wedge, a sidecar), not for a database every pod shares.

## CPU throttling check

When p99 latency spikes but CPU averages look fine, compare throttled periods with total periods per container (cAdvisor metrics):

```promql
sum by (namespace, pod, container) (rate(container_cpu_cfs_throttled_periods_total{container!=""}[5m]))
/
sum by (namespace, pod, container) (rate(container_cpu_cfs_periods_total{container!=""}[5m]))
```

A ratio that is regularly well above zero on a latency-sensitive service means the limit is shaping latency. Raise or remove the limit, or reduce parallelism inside the process (thread pools, `GOMAXPROCS` on Go before 1.25, worker counts) to match it.

## Migrations as a Job

Run the migration once per release, before the Deployment changes:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: orders-migrate-{{ .Values.release.id }}
spec:
  backoffLimit: 0
  activeDeadlineSeconds: {{ .Values.migration.deadlineSeconds }}
  template:
    spec:
      restartPolicy: Never
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile: { type: RuntimeDefault }
      containers:
        - name: migrate
          image: "{{ .Values.image.repository }}@{{ .Values.image.digest }}"
          command: ["node", "dist/migrate.js"]
```

The pipeline applies the Job, waits for `condition=complete` with a timeout, stops the release on failure or timeout, and only then updates the Deployment. `backoffLimit: 0` because a migration that failed halfway needs a person or an idempotent rerun, not an automatic retry loop. Name the Job per release: a Job's pod template cannot be changed after creation, so reapplying one fixed name with a new image fails. The migration itself sets `lock_timeout` and follows the expand-contract rules in database-engineering.

If the migration tool keeps a lock row (Liquibase `DATABASECHANGELOGLOCK`, similar tables in other tools), a killed run can leave it held. Know the tool's release command before the first incident, and alert when a migration Job exceeds its deadline instead of letting new pods wait on the lock.
