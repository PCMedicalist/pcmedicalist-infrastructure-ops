---
name: pcmedicalist-infrastructure-ops
description: Use when operating PCMedicalist deployment/release/observability practices — graceful shutdown, statelessness, capacity, load testing, profiling, GitOps, 12-factor, containers, lockfiles, branch protection.
version: 1.0.0
author: PCMedicalist
license: CC-BY-4.0 (PCMedicalist attribution required)
metadata:
  hermes:
    tags: [infrastructure, operations, devops, kubernetes, gitops, pcmedicalist]
    related_skills: [pcmedicalist-blue-green, pcmedicalist-canary, pcmedicalist-rollbacks, pcmedicalist-feature-flags, pcmedicalist-graceful-shutdown, pcmedicalist-connection-draining, pcmedicalist-health-checks, pcmedicalist-liveness-readiness, pcmedicalist-stateless-services, pcmedicalist-multi-region, pcmedicalist-slis, pcmedicalist-slos, pcmedicalist-error-budgets, pcmedicalist-incident-response, pcmedicalist-postmortems, pcmedicalist-chaos-engineering, pcmedicalist-autoscaling, pcmedicalist-h-scaling, pcmedicalist-vertical-scaling, pcmedicalist-cold-starts, pcmedicalist-capacity-planning, pcmedicalist-load-testing, pcmedicalist-memory-leaks, pcmedicalist-oom-killer, pcmedicalist-logging, pcmedicalist-log-levels, pcmedicalist-log-rotation, pcmedicalist-distributed-tracing, pcmedicalist-opentelemetry, pcmedicalist-correlation-ids, pcmedicalist-metrics-cardinality, pcmedicalist-p99-latency, pcmedicalist-alert-fatigue, pcmedicalist-error-tracking, pcmedicalist-profiling, pcmedicalist-flame-graphs, pcmedicalist-infra-as-code, pcmedicalist-terraform-state, pcmedicalist-gitops, pcmedicalist-environment-parity, pcmedicalist-staging-environments, pcmedicalist-12-factor-apps, pcmedicalist-multi-stage-builds, pcmedicalist-non-root-containers, pcmedicalist-image-scanning, pcmedicalist-resource-requests-limits, pcmedicalist-pod-disruption-budgets, pcmedicalist-semantic-versioning, pcmedicalist-lockfiles, pcmedicalist-reproducible-builds, pcmedicalist-branch-protection]
---

# PCMedicalist Infrastructure & Operations

## When to Use
- Operating or hardening a PCMedicalist deployment (baseLINE VPS, agent fleet, containerized services).
- Designing graceful shutdown, health checks, autoscaling, or statelessness into a service.
- Running load tests, capacity planning, or profiling a memory leak / OOM.
- Establishing release engineering: GitOps, 12-factor, multi-stage builds, non-root containers, image scanning, lockfiles, reproducible builds, branch protection.
- Setting up observability: structured logs, tracing, correlation IDs, metrics cardinality control, p99 alerts, error tracking, flame graphs.
- Any of the 22 topics below is in scope.

## Portfolio Reality
PCMedicalist runs baseLINE on a **single DreamHost VPS** (<DEPLOY_HOST>) via **PM2 + nginx**, with a **git-ignored `.env` (mode 600)** as local source of truth synced via rsync to the VPS. Deploys use GitHub Actions → rsync + PM2 reload. The 0x-agents fleet and RetroByte run locally. There is **no Kubernetes, no GitOps pipeline, no formal load test, and no APM** today — this skill defines the target practice as we scale, while respecting the current VPS/PM2 reality. Where a topic has its own skill, this one links rather than duplicates.

## Topics Covered (22)
1. Graceful Shutdown
2. Connection Draining
3. Stateless Services
4. Load Testing
5. Capacity Planning
6. Memory Leaks
7. OOM Killer
8. Structured Logging
9. Log Levels
10. Log Rotation
11. Distributed Tracing
12. OpenTelemetry
13. Correlation IDs
14. Metrics Cardinality
15. p99 Latency
16. Alert Fatigue
17. Error Tracking
18. Profiling / Flame Graphs
19. GitOps
20. 12-Factor Apps
21. Multi-Stage Builds / Non-Root Containers / Image Scanning
22. Resource Requests & Limits / Pod Disruption Budgets / Lockfiles / Reproducible Builds / Semantic Versioning / Branch Protection

Each topic: **Why → Pattern → Validate → Fix**.

---

## 1. Graceful Shutdown
**Why:** Killing a process mid-request drops in-flight work, corrupts connections, leaves PM2 in a bad state. Shutdown must stop accepting new work, finish in-flight, then exit.

**Pattern (Node/Express + PM2):**
```ts
async function shutdown(signal: string) {
  console.log(`Received ${signal}, draining...`);
  server.close(() => console.log('HTTP server closed'));   // stop accepting
  await prisma.$disconnect();                               // close DB pool
  await redis.quit();
  process.exit(0);
}
process.on('SIGTERM', () => shutdown('SIGTERM'));           // PM2 sends SIGTERM
process.on('SIGINT', () => shutdown('SIGINT'));
// PM2: in ecosystem.config: "shutdown_timeout": 30000 (wait up to 30s)
```
Set `shutdown_timeout` > your longest in-flight request. Never `process.exit()` immediately on signal.

**Validate:** During a deploy (PM2 reload), `ss -tnp` shows no half-open connections after shutdown; in-flight requests complete (not 502); DB pool closes cleanly (no `remaining connections` warnings).

**Fix:** If deploys drop requests, add the `SIGTERM` handler + `server.close()` + pool drains. Raise `shutdown_timeout`. Confirm PM2 isn't sending `SIGKILL` before drain.

---

## 2. Connection Draining
**Why:** At the load balancer / ingress, in-flight requests to a backend being removed must finish before the socket closes. Without draining, the LB sends RST and clients get errors during deploys.

**Pattern (nginx upstream + keepalive):**
```nginx
upstream baseline_app {
  server 127.0.0.1:3000;
  keepalive 32;
}
# on removal: nginx honors proxy_requests in flight; for K8s, use readiness=false to drain
```
For K8s: set `terminationGracePeriodSeconds` + readiness probe flips to false so the Service stops routing while pods drain. For PM2 single-host, the app's own graceful shutdown (topic 1) is the drain.

**Validate:** During a backend removal/deploy, active connections complete (no RST/502), new ones route to the healthy backend. `proxy`/`ingress` logs show requests finishing, then the backend leaving the pool.

**Fix:** If clients see 502s on deploy, ensure readiness flips false before termination and `terminationGracePeriodSeconds` covers drain. On single-host, rely on app-level graceful shutdown.

---

## 3. Stateless Services
**Why:** Stateful app processes can't scale horizontally or fail over cleanly — session affinity becomes a liability. Push state to Redis/DB, keep the process ephemeral.

**Pattern:**
```ts
// WRONG: session in process memory
const sessions = new Map();   // lost on restart, wrong instance on scale-out
// RIGHT: session in Redis (shared, survives restart, works across instances)
app.use(session({ store: new RedisStore({ client: redis }), secret: SESSION_SECRET, resave: false, saveUninitialized: false }));
```
Any mutable per-request state lives in Redis/Postgres, not the process. The process is replaceable at any moment (cattle, not pet).

**Validate:** Kill and restart the process → active users stay logged in (session in Redis). Run 3 instances behind a load balancer with no sticky sessions → all requests succeed regardless of which instance handles them.

**Fix:** If state is in-memory, externalize it (Redis for sessions/caches, DB for durable). Remove process-local mutable maps used across requests. See `pcmedicalist-stateless-services` if standalone.

---

## 4. Load Testing
**Why:** You don't know your breaking point until you test. "It works in dev" ≠ "handles 10k tips/min". Uncover bottlenecks before prod does.

**Pattern:**
```bash
# k6 example: ramp to 1000 VUs over 5m, hold 5m, spike to 2000
k6 run --vus 1000 --duration 5m --stage 5m:1000,5m:1000,2m:2000 loadtest.js
# watch: p99 latency, error rate, CPU/mem on VPS, DB connection count
```
Always test against a **staging** environment (topic 20) with prod-like data. Define the SLO (e.g. p99 < 500ms, error < 0.1%) and find where it breaks. Capture the breaking point + which resource saturates first.

**Validate:** The test reproduces a known prod-like mix; results show p99/error/cpu at each load step; the bottleneck resource is identified (DB pool? CPU? Redis? network?). A regression run post-fix shows improvement.

**Fix:** If you've never load-tested, start with k6/Artillery against staging. Set the SLO first. Fix the identified bottleneck (pool sizing, indexing, caching) and re-test. Make load tests part of the release gate for high-traffic changes.

---

## 5. Capacity Planning
**Why:** Right-size for peak + growth without overpaying. Under-provision = outages at peak; over-provision = wasted spend.

**Pattern:**
- Baseline on **peak** load, not average (peak drives the failure).
- Add headroom: target ≤ 60–70% utilization at peak (room for spikes + a node loss).
- Project growth: `required = peak * (1 + growth_rate)^horizon * safety_factor`.
- Track leading indicators (QPS, conn count, queue depth) toward limits.

**Validate:** At projected 6-month peak, the planned capacity shows ≤ 70% utilization in the load test (topic 4). A single-node/AZ loss still leaves ≥ 100% of current peak capacity (N+1). No resource trends toward 100% within the planning horizon.

**Fix:** If peak utilization > 80%, add capacity (scale up/out) or optimize. If > 100% projected, act before it happens. Document the capacity model + review quarterly.

---

## 6. Memory Leaks
**Why:** A slow leak grows RSS until OOM (topic 7) or GC thrash. Common in long-lived Node/PM2 processes: retained closures, unbounded caches, forgotten listeners, uncleared timers.

**Pattern (detection):**
```bash
# watch RSS over time
pm2 monit   # or: ps -o rss= -p $(pgrep -f baseline)
# heap snapshot diff with clinic.js / heapdump
npx clinic heapprofiler -- node dist/server.js
```
Fix patterns: bound caches (LRU with max), remove listeners on cleanup (`EventEmitter` off / `req.on('close')`), clear intervals, avoid storing large objects in module scope.

**Validate:** RSS plateaus under sustained load (no monotonic growth over hours). A heap-diff between two steady-state snapshots shows no retained-growing object class. After the fix, a 24h soak holds flat memory.

**Fix:** If RSS climbs forever, capture heap snapshots (pre/post leak), diff to find the retained class, fix the retention (bound the cache / remove the listener). Add a soak test to CI to catch regressions.

---

## 7. OOM Killer
**Why:** When the kernel runs out of memory it kills the highest-RSS process — often your app, taking down the service abruptly.

**Pattern:**
```bash
# cgroup memory limit (container): set requests/limits so OOM is predictable, not host-wide
# host-level: monitor `dmesg | grep -i 'killed process'` ; set vm.overcommit_memory appropriately
# Node: cap heap to leave room for the OS
node --max-old-space-size=1536 dist/server.js   # 1.5GB cap on a 2GB host
```
Prefer a **controlled** OOM (container limit with restart policy) over the host killing arbitrarily. Alert on OOM events. Set `--max-old-space-size` so Node throws `HeapOutOfMemory` (catchable) before the kernel OOM.

**Validate:** Under memory pressure, the constrained process restarts cleanly (restart policy) rather than taking down the host. `dmesg` shows no unexpected `killed process` for critical services. Alerts fire on OOM.

**Fix:** If the host OOM-killed your app, add container/process memory limits + restart policy. Reduce the leak (topic 6). Right-size the host. Alert on OOM so it's never a surprise.

---

## 8. Structured Logging
**Why:** `console.log("user " + id + " did " + action)` is unparseable at scale. Structured (JSON) logs are queryable, have levels, and feed the pipeline.

**Pattern:**
```ts
import pino from 'pino';
const log = pino({ level: process.env.LOG_LEVEL || 'info' });
log.info({ userId, action: 'tip.create', amountWei, correlationId }, 'tip created');
// NOT: console.log('tip created for ' + userId);
```
Every log line is a JSON object with a level, timestamp, and structured fields (no string concatenation). Ship to a central store (Loki/ELK) — not just stdout lost on restart.

**Validate:** Log lines are valid JSON with `level`, `time`, and relevant fields. You can query "all errors in last 5 min for userId=X" in the log backend. No secrets in log fields (scan for tokens/keys).

**Fix:** If logs are plain strings, adopt pino/winston with JSON output. Replace `console.log` with structured `log.<level>`. Ensure no PII/secrets are logged (redact). Ship to central storage with retention.

---

## 9. Log Levels
**Why:** Logging everything at `info` = noise; logging everything at `debug` in prod = cost + blind. Levels let you control verbosity per environment.

**Pattern:**
```
ERROR — something failed, needs attention
WARN  — degraded but functioning (fallback used, retry)
INFO  — significant lifecycle events (startup, tip confirmed)
DEBUG — detailed flow (only in staging/debug builds)
TRACE — finest-grained (never in prod)
```
Set `LOG_LEVEL=info` in prod, `debug` in staging. Never log `DEBUG`+ in prod by default (cardinality + cost). See `pcmedicalist-log-levels` if standalone.

**Validate:** Prod runs at `info` (no debug noise in the stream). Bumping to `debug` in staging reveals flow detail. ERROR/WARN are actionable (not INFO-level events mislabeled).

**Fix:** If prod is drowning in debug, raise the level. If you can't diagnose an issue because logs are too sparse, ensure ERROR/WARN/INFO capture enough. Standardize level usage across services.

---

## 10. Log Rotation
**Why:** Unbounded logs fill the disk → the service dies (no space). Rotation archives + deletes old logs on a schedule.

**Pattern:**
```bash
# logrotate for PM2/file logs
/var/www/baseline/logs/*.log {
  daily
  rotate 14
  compress
  missingok
  notifempty
  copytruncate    # PM2 keeps fd open; copytruncate avoids restart
}
# or rely on a log shipper (Fluent Bit) to ship + drop local
```
For containerized: ship to stdout + a collector; don't write to local disk long-term. Alert on disk > 80% (topic 7 ties in).

**Validate:** After rotation, old logs are compressed/archived and disk usage stays bounded over weeks. No `No space left on device` from logs. `logrotate -d` (dry run) shows the expected plan.

**Fix:** If disk fills from logs, add logrotate (with `copytruncate` for long-running fd holders) or a shipper. Set retention (14–30 days). Monitor disk %.

---

## 11. Distributed Tracing
**Why:** A request crossing 5 services (Telegram → agent → baseLINE API → Postgres → Bankr) is invisible in logs alone. Tracing shows the path + where time goes.

**Pattern (spans):**
```ts
// each service creates a span, propagates traceparent header
import { trace } from '@opentelemetry/api';
const span = trace.getTracer('baseline').startSpan('tip.create');
// child spans for DB call, Bankr call, each annotated with duration
span.end();
```
Use W3C `traceparent` propagation across HTTP/gRPC. Sample (head or tail) to control volume. See `pcmedicalist-distributed-tracing`, `pcmedicalist-opentelemetry`.

**Validate:** A single user action produces one trace ID spanning all services; the waterfall shows each hop's duration; the slow span is identifiable. Trace context propagates across service boundaries (not lost at the HTTP call).

**Fix:** If you can't follow a request across services, instrument with OTel + `traceparent` propagation. Sample to keep volume sane. Root-cause the slow span found in traces.

---

## 12. OpenTelemetry
**Why:** Vendor-neutral instrumentation — instrument once, export to any backend (Jaeger, Tempo, Honeycomb). Avoids lock-in + manual instrumentation duplication.

**Pattern:**
```ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
new NodeSDK({ traceExporter: new OTLPTraceExporter({ url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT }) }).start();
```
Auto-instrument (HTTP, DB, Express) + manual spans for business logic. Use the OTLP exporter to your backend of choice. See `pcmedicalist-opentelemetry`.

**Validate:** Spans flow to the backend without code changes per vendor. Switching backends = config change only (no re-instrumentation). Auto-instrument captures DB/HTTP without manual span code.

**Fix:** If instrumentation is vendor-specific, migrate to OTel SDK + OTLP. Enable auto-instrumentation. Standardize the exporter endpoint via env var.

---

## 13. Correlation IDs
**Why:** Without a request ID threaded through every log/trace, you can't piece together one user's journey across services.

**Pattern:**
```ts
// express middleware: generate or adopt incoming X-Request-ID / traceparent
app.use((req, res, next) => {
  req.correlationId = req.headers['x-request-id'] || crypto.randomUUID();
  res.setHeader('X-Request-ID', req.correlationId);
  log = log.child({ correlationId: req.correlationId });   // pino child logger
  next();
});
```
Propagate the ID to every downstream call (header) and into every log line + span. See `pcmedicalist-correlation-ids`.

**Validate:** A single request's logs across all services share the same `correlationId`. You can grep one ID and reconstruct the full journey. The ID is passed to downstream HTTP calls (not regenerated per hop).

**Fix:** If logs can't be correlated across services, add the middleware + propagate the header. Ensure every log line + span includes it. Use W3C `traceparent` as the canonical ID where tracing exists.

---

## 14. Metrics Cardinality
**Why:** High-cardinality metrics (a tag with unbounded values like `userId` or `requestId`) explode the time-series DB → cost blowup + query slowdown. The #1 observability footgun.

**Pattern:**
```ts
// WRONG — cardinality bomb
metrics.inc('request', { userId: req.user.id, path: req.path });
// RIGHT — bounded labels only
metrics.inc('request', { route: 'tip.create', status: 200 });   // route, not per-user
// per-user analytics → logs/traces, NOT metrics
```
Keep metric label values bounded (routes, status codes, region — not user IDs, UUIDs, timestamps). See `pcmedicalist-metrics-cardinality`.

**Validate:** The active series count is stable (not growing daily). No metric has a label with > ~100 distinct values in normal traffic. A dashboard query returns in seconds, not timeouts.

**Fix:** If series count grows unbounded, find the offending label (often `userId`/`ip`/`requestId`) and move it to logs/traces. Cap label cardinality. Alert on series-count growth.

---

## 15. p99 Latency
**Why:** Averages lie. p99 (99th percentile) is what your slowest 1% of users experience — the SLO that matters for "feels fast".

**Pattern:**
```ts
// record histogram, not just avg
const hist = metrics.histogram('http_duration_ms', { route });
hist.record(durationMs, { route: 'tip.create' });
// alert/dashboard on p99 (not mean)
```
Track p50/p95/p99 per route. Set SLO on p99 (e.g. p99 < 500ms). Investigate p99 regressions before p50 — tail latency is where users feel pain. See `pcmedicalist-p99-latency`.

**Validate:** The dashboard shows p99 (not just avg) per route; a deploy that improves mean but regresses p99 is caught. Alerts fire on p99 breach, not average. Traces (topic 11) explain the p99 spike.

**Fix:** If you only watch averages, add p99 histograms. Set p99 SLOs. When p99 regresses, use traces to find the slow tail (a specific dependency, lock, or GC pause).

---

## 16. Alert Fatigue
**Why:** Too many alerts (or noisy ones) → humans ignore them → real incidents missed. The boy-who-cried-wolf of ops.

**Pattern:**
- Alert on **symptoms** users feel (error rate, p99, saturation), not causes (CPU at 80% isn't an alert unless it breaches SLO).
- Use **multi-window burn-rate** for SLO-based alerts (avoids flapping).
- Dedupe + group; route to the right on-call with context.
- Every alert has a **runbook link** + a clear action.
```yaml
# only page on SLO burn, not every blip
alert: ErrorBudgetBurn
expr: burn_rate(slo) > 14.4   # 2% budget in 1h
for: 2m
```
See `pcmedicalist-alert-fatigue`, `pcmedicalist-slos`.

**Validate:** Alert volume is low enough that on-call trusts every page (e.g. < 2 pages/week false-positive). Each alert has a runbook + actionable cause. Symptom-based alerts catch user-impacting events. No alert fires for non-user-visible blips.

**Fix:** If on-call ignores alerts, audit them: delete/downgrade cause-based noise, keep symptom-based pages. Add burn-rate SLO alerts. Attach runbooks. Measure false-positive rate and drive it down.

---

## 17. Error Tracking
**Why:** Exceptions in prod you never see = silent failures. Centralized error tracking captures, groups, and notifies on regressions.

**Pattern:**
```ts
// Sentry / similar
import * as Sentry from '@sentry/node';
Sentry.init({ dsn: process.env.SENTRY_DSN, tracesSampleRate: 0.1 });
// automatic capture of unhandled rejections + thrown errors
app.use(Sentry.Handlers.errorHandler());
```
Group by error type + fingerprint; alert on new-error or spike. Include `correlationId` (topic 13) in the event for cross-link. See `pcmedicalist-error-tracking`.

**Validate:** A thrown exception in prod appears in the tracker grouped + deduplicated. A new error type triggers a notification. `correlationId` is attached so you can jump to logs/traces. No PII in error context.

**Fix:** If errors vanish into stdout, add Sentry/Rollbar. Ensure unhandled rejections + async errors are captured (not just try/catch). Group + alert on new/spiking errors. Scrub PII.

---

## 18. Profiling / Flame Graphs
**Why:** "The CPU is high" tells you nothing. A flame graph shows which function burns time — the actual optimization target.

**Pattern:**
```bash
# continuous profiling (Pyroscope) or on-demand
node --prof dist/server.js && node --prof-process isolate-*.log > profile.txt
# OR clinic.js / 0x for flame graphs
npx clinic flame -- node dist/server.js
```
Capture during representative load (topic 4). Read top-down: widest frame = most CPU. Fix the hot function (algorithm, caching, N+1). See `pcmedicalist-profiling`, `pcmedicalist-flame-graphs`.

**Validate:** A flame graph from a loaded run shows a clear hot path; after the fix, that frame shrinks and p99/CPU improves. The profile was taken under realistic load (not idle).

**Fix:** If "CPU is high" with no detail, capture a flame graph under load. Identify the widest frame, optimize it (cache, batch, fix N+1). Re-profile to confirm.

---

## 19. GitOps
**Why:** Manual `kubectl apply` / SSH deploys drift from source and aren't auditable. GitOps makes git the single source of truth; clusters reconcile automatically.

**Pattern (ArgoCD / Flux):**
```
git repo (manifests)  ->  ArgoCD watches  ->  cluster converges to declared state
# change = PR to manifests; merge = automatic, auditable, reversible
```
For our current VPS: a GitOps-lite = GitHub Actions reads the canonical repo + applies (rsync + PM2 reload), with the repo as source of truth. See `pcmedicalist-gitops`.

**Validate:** The cluster/repo state matches git after a PR merges (reconcile succeeds). A manual out-of-band change is detected + reverted (drift detection). Rollback = `git revert` + auto-apply. Audit log = git history.

**Fix:** If deploys are manual/SSH-only, move to GitOps (or GitOps-lite via Actions). Make git the source of truth. Enable drift detection. Rollbacks become git operations.

---

## 20. 12-Factor Apps
**Why:** A methodology for portable, scalable, maintainable services. Relevant to baseLINE as we containerize.

**Key factors (applied):**
1. **Codebase** — one repo per app, tracked in git.
2. **Dependencies** — explicit, locked (`package-lock.json`) — see topic 22.
3. **Config** — in env (our `.env`, mode 600), NOT in code. Twelve-factor: config = env vars, injected per environment.
4. **Backing services** — DB/Redis are attached resources (URL via env), swappable.
5. **Build/Release/Run** — strictly separated (CI builds, deploy releases).
6. **Processes** — stateless (topic 3).
7. **Port binding** — export via port; self-contained.
8. **Concurrency** — scale via processes.
9. **Disposability** — fast startup + graceful shutdown (topic 1).
10. **Dev/prod parity** — staging ~ prod (topic 20 below).
11. **Logs** — event stream to stdout (topic 8).
12. **Admin processes** — run as one-off (migrations) via release step.
See `pcmedicalist-12-factor-apps`.

**Validate:** Config is external (no secrets in code/repo). Processes are stateless (topic 3). Build/release/run are separate. Logs go to stdout. The app boots fast + shuts down gracefully.

**Fix:** If config is hardcoded, externalize to env. If state is in-process, fix (topic 3). If build/release/run are entangled, separate them (CI build → release artifact → run).

---

## 21. Multi-Stage Builds / Non-Root Containers / Image Scanning
**Why:** Bloated images slow deploys + widen attack surface; root containers let a compromise own the host; unscannd images ship CVEs.

**Pattern (Dockerfile):**
```dockerfile
# multi-stage: build deps separate from runtime
FROM node:20 AS build
WORKDIR /app && COPY . . && npm ci && npm run build
FROM node:20-slim AS runtime
WORKDIR /app && COPY --from=build /app/dist ./dist && npm ci --omit=dev
USER node:node            # non-root
EXPOSE 3000
CMD ["node","dist/server.js"]
```
Scan: `trivy image myapp:tag` in CI; fail on HIGH/CRITICAL. No `latest` tag (use digest/semver). See `pcmedicalist-multi-stage-builds`, `pcmedicalist-non-root-containers`, `pcmedicalist-image-scanning`.

**Validate:** Final image is slim (no build toolchain). Container runs as non-root (`whoami` ≠ root). `trivy` shows no HIGH/CRITICAL. Build is reproducible (same input → same digest).

**Fix:** If the image is one big stage + root, split multi-stage + add `USER node`. Add Trivy/Grype to CI failing on criticals. Pin base image by digest.

---

## 22. Resource Limits / PDBs / Lockfiles / Reproducible Builds / SemVer / Branch Protection
**Why:** These are the release-engineering guardrails that prevent "works on my machine" and accidental breakage.

- **Resource Requests & Limits** (K8s): `requests` guarantee, `limits` cap. Prevent noisy-neighbor + OOM storms. See `pcmedicalist-resource-requests-limits`.
- **Pod Disruption Budgets**: ensure ≥ N replicas stay up during voluntary disruptions (node drain, deploy). `minAvailable: 2`. See `pcmedicalist-pod-disruption-budgets`.
- **Lockfiles**: commit `package-lock.json` / `poetry.lock`. `npm ci` not `npm install` in CI. See `pcmedicalist-lockfiles`.
- **Reproducible Builds**: same input → bit-identical output (pinned deps, deterministic order, no timestamps in artifacts). See `pcmedicalist-reproducible-builds`.
- **Semantic Versioning**: `MAJOR.MINOR.PATCH` — breaking/feature/fix. APIs version accordingly. See `pcmedicalist-semantic-versioning`.
- **Branch Protection**: require PR review + CI green before merge to `main`; no force-push; no direct commits. Branch protection is the gate that makes all the above stick. See `pcmedicalist-branch-protection`.

**Pattern (GitHub branch protection via API/CLI):**
```bash
gh api repos/PCMedicalist/pcmedicalist-baseline/branches/main/protection \
  --method PUT --input - <<'JSON'
{ "required_status_checks": {"strict": true, "contexts": ["build","test"]},
  "enforce_admins": true, "required_pull_request_reviews": {"required_approving_review_count": 1},
  "restrictions": null }
JSON
```

**Validate:** `npm ci` on a clean checkout reproduces the exact dependency tree (lockfile committed, no drift). Build is reproducible (same digest). `main` rejects direct pushes + unreviewed PRs (branch protection active). A voluntary disruption keeps ≥ minAvailable pods.

**Fix:** If lockfiles aren't committed, commit them + switch CI to `npm ci`. If `main` allows direct push, enable branch protection. If builds aren't reproducible, remove timestamps/non-determinism. If no PDB, add one matching your replica count.

---

## Validation Checklist (22 topics)
- [ ] SIGTERM → graceful shutdown + pool drains (1)
- [ ] LB/ingress drains in-flight on removal (2)
- [ ] State externalized (Redis/DB), process ephemeral (3)
- [ ] Load test against staging with defined SLO (4)
- [ ] Capacity = peak × growth × safety, ≤70% at peak (5)
- [ ] RSS plateaus (no leak); heap-diff clean (6)
- [ ] Memory limits + restart policy; no host OOM (7)
- [ ] JSON structured logs, no secrets (8)
- [ ] LOG_LEVEL=info prod, debug staging (9)
- [ ] Log rotation bounded; disk monitored (10)
- [ ] Spans across services, traceparent propagated (11)
- [ ] OTel SDK + OTLP exporter (12)
- [ ] correlationId on every log/span/downstream call (13)
- [ ] Metric labels bounded (no userId/ip) (14)
- [ ] p99 (not mean) tracked + SLO'd (15)
- [ ] Symptom-based alerts + runbooks, low false-positive (16)
- [ ] Centralized error tracking, grouped, PII-scubbed (17)
- [ ] Flame graph identifies hot function under load (18)
- [ ] Git is source of truth; drift detected (19)
- [ ] 12-factor: config external, stateless, logs to stdout (20)
- [ ] Multi-stage + non-root + image scan clean (21)
- [ ] Lockfiles/limits/PDB/SemVer/branch-protection in force (22)

## Fix Recipes
- **"Deploys drop requests"** → #1, #2: graceful shutdown + drain.
- **"Memory grows forever"** → #6, #7: find leak, bound caches, set limits.
- **"Can't debug cross-service"** → #11–13: tracing + correlation IDs.
- **"Alerts ignored"** → #16: symptom-based + runbooks.
- **"Works locally, breaks in prod"** → #20, #22: 12-factor + lockfiles + branch protection.
- **"Bloated/root/vulnerable image"** → #21: multi-stage + non-root + Trivy.
- **"Unreviewed push to main"** → #22: enable branch protection.

## Cross-References
- `pcmedicalist-blue-green` / `pcmedicalist-canary` / `pcmedicalist-rollbacks` — safe deploys
- `pcmedicalist-feature-flags` — progressive rollout
- `pcmedicalist-health-checks` / `pcmedicalist-liveness-readiness` — #2 drain signal
- `pcmedicalist-stateless-services` — #3
- `pcmedicalist-multi-region` — #5 capacity across regions
- `pcmedicalist-slis` / `pcmedicalist-slos` / `pcmedicalist-error-budgets` — #15, #16 SLOs
- `pcmedicalist-incident-response` / `pcmedicalist-postmortems` / `pcmedicalist-chaos-engineering` — ops maturity
- `pcmedicalist-autoscaling` / `pcmedicalist-h-scaling` / `pcmedicalist-vertical-scaling` / `pcmedicalist-cold-starts` — #5 scaling
- `pcmedicalist-load-testing` — #4
- `pcmedicalist-memory-leaks` / `pcmedicalist-oom-killer` — #6, #7
- `pcmedicalist-logging` / `pcmedicalist-log-levels` / `pcmedicalist-log-rotation` — #8–10
- `pcmedicalist-distributed-tracing` / `pcmedicalist-opentelemetry` / `pcmedicalist-correlation-ids` — #11–13
- `pcmedicalist-metrics-cardinality` / `pcmedicalist-p99-latency` / `pcmedicalist-alert-fatigue` / `pcmedicalist-error-tracking` — #14–17
- `pcmedicalist-profiling` / `pcmedicalist-flame-graphs` — #18
- `pcmedicalist-infra-as-code` / `pcmedicalist-terraform-state` / `pcmedicalist-gitops` — #19
- `pcmedicalist-environment-parity` / `pcmedicalist-staging-environments` — #20 dev/prod parity
- `pcmedicalist-12-factor-apps` — #20
- `pcmedicalist-multi-stage-builds` / `pcmedicalist-non-root-containers` / `pcmedicalist-image-scanning` — #21
- `pcmedicalist-resource-requests-limits` / `pcmedicalist-pod-disruption-budgets` — #22
- `pcmedicalist-semantic-versioning` / `pcmedicalist-lockfiles` / `pcmedicalist-reproducible-builds` / `pcmedicalist-branch-protection` — #22
