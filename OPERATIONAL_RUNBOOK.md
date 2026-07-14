# Operational Runbook - iowire.com Ecosystem

**Purpose**: Day-to-day operations, maintenance, and troubleshooting guide
**Audience**: System administrators, DevOps engineers, on-call staff
**Last Updated**: 2026-05-04

---

## 📋 Table of Contents

1. [System Overview](#system-overview)
2. [Common Operations](#common-operations)
3. [Monitoring & Health Checks](#monitoring--health-checks)
4. [Alert Runbooks](#alert-runbooks)
5. [Troubleshooting](#troubleshooting)
6. [Backup & Recovery](#backup--recovery)
7. [Incident Response](#incident-response)
8. [Maintenance Procedures](#maintenance-procedures)
9. [Emergency Contacts](#emergency-contacts)

---

## System Overview

### Architecture

**Infrastructure**: Single EC2 instance (us-east-1) — `t3.medium` <!-- TODO(R4): bump to t3.large per Sprint 107 R4 (status: Infra owner, not done as of 2026-05-04) -->
- **IP**: `<DB-EIP>` (Elastic IP — `dig +short standard.iowire.com` is the source of truth)
- **OS**: Ubuntu
- **SSH Key**: `~/code/tb_code/cert/standard-poc-key.pem`
- **SSH User**: `ubuntu`
- **Canonical host**: `standard.iowire.com` (A record points to Elastic IP)
- **Code Path**: `/opt/<app>`

**Services** (Docker Compose — canonical compose file is `docker-compose.yml`; pinned image versions in parens):
- `caddy` — reverse proxy, auto-TLS, CSP headers (`caddy:2.9.1-alpine`)
- `db` — PostgreSQL 15 (`postgres:15-alpine`)
- `pgbouncer` — connection pooling, transaction-mode (`edoburu/pgbouncer:v1.25.1-p0`)
- `redis` — sessions, cache, Celery broker, failed-login tracking (`redis:7-alpine`)
- `app` — STANDARD Flask app, `gunicorn -k gevent -w 1 -b 0.0.0.0:5000 --worker-connections 1000` (image built locally; see [Gunicorn Worker Count](#gunicorn-worker-count))
- `celery-worker` — async tasks (concurrency 2, max-tasks-per-child 100). Canonical task list: `boom-queue/celery_app.py:178-212` (`include` arg).
- `celery-beat` — scheduled tasks: nightly Connect reconciliation (`06:00 UTC`), nightly Stripe transfer reconciliation, sandbox balance topup every 4h at `:15` (`tasks.sandbox_topup.maintain_sandbox_balance`, no-op outside sandbox mode).
- `tusd` — resumable file uploads (`tusproject/tusd:v2`)
- `livekit` — WebRTC SFU, host-networked (`livekit/livekit-server:v1.7`); ingress (`v1.5`) and egress (`v1.8`) sidecars.
- `alloy` — Grafana Agent / log shipping (`grafana/alloy:v1.15.0`)

**Active Subdomains**:
- `iowire.com` — Main marketing site
- `standard.iowire.com` — STANDARD music platform (the only Flask app)
- `admin.iowire.com` — Redirects to `/admin` on STANDARD

**Archived (DO NOT reference in active commands)**:
- `portal.iowire.com` — Client portal, archived
- `metrics.iowire.com` — Analytics dashboard, archived
- The `portal`, `metrics`, and `admin` Docker services were retired with those subdomains. The single `app` container now serves all `/admin` routes via the STANDARD Flask app.

### Access Information

**SSH Access**:
```bash
ssh -i ~/code/tb_code/cert/standard-poc-key.pem ubuntu@standard.iowire.com
```

**Database Access**:

> **Security note:** Master DB password is fetched via SSM in a subshell;
> never paste into a DSN string (lands in shell history + `/proc/<pid>/cmdline`,
> visible to `ps`). The `( ... )` subshell scopes `PGPASSWORD` to the single
> command — it does NOT leak into the parent shell or subsequent invocations.

# Prod DB is the dedicated **RDS** `standard-prod-db` (no local `<db-container>`
# container). The app connects as the NON-ROTATING owner role `standard_iowire_owner`
# through pgbouncer; use that role for app-consistent access, NOT the auto-rotating
# `postgres` master. The RDS is reachable ONLY from the prod EC2 box (SG-scoped), so
# run these ON the box after `ssh ubuntu@standard.iowire.com` (confirm: `dig +short standard.iowire.com` = <DB-EIP>).

```bash
# Easiest: reuse the app container's existing owner DATABASE_URL (through pgbouncer)
docker compose -f docker-compose.yml -f docker-compose.production.yml \
  exec app sh -lc 'psql "$DATABASE_URL"'

# Direct to RDS as the owner role. PGPASSWORD lives only inside the subshell parens.
( PGPASSWORD=$(AWS_PROFILE=standard aws ssm get-parameter \
    --name /<env>/RDS_OWNER_PASSWORD \
    --with-decryption \
    --query Parameter.Value \
    --output text) \
  psql "postgresql://standard_iowire_owner@standard-prod-db.c09gyk8iqz13.us-east-1.rds.amazonaws.com:5432/standard_iowire?sslmode=require" )
```

**Service Logs** (`docker compose -f docker-compose.yml ...` from `/opt/<app>`):
```bash
cd /opt/<app>

# All services, follow
docker compose -f docker-compose.yml logs -f --tail=100

# Specific service
docker compose -f docker-compose.yml logs -f app
docker compose -f docker-compose.yml logs -f celery-worker
docker compose -f docker-compose.yml logs -f caddy
docker compose -f docker-compose.yml logs -f db
```

> Use `docker compose` (space) — `docker-compose` (hyphen) still resolves but is the deprecated v1 binary; the prod host has v2 only.

---

## Common Operations

### Daily Tasks

#### 1. Check System Health

```bash
# SSH into server
ssh -i ~/code/tb_code/cert/standard-poc-key.pem ubuntu@standard.iowire.com

# Check all services are running
cd /opt/<app>
docker compose -f docker-compose.yml ps

# Expected output: All services "Up (healthy)"
# If any show "Exited" or "Unhealthy", investigate

# Check health endpoint (Caddy serves the app's /health on 80/443; localhost 5000 is the app's
# direct port, host-bound to loopback only so LiveKit can hit it)
curl http://localhost:5000/health
# Expected: JSON with {"status": "ok", ...}

# Check all live subdomains (admin redirects to standard's /admin)
for domain in iowire.com standard.iowire.com admin.iowire.com; do
  echo "Testing $domain..."
  curl -I https://$domain | head -1
done
# Expected: All return "HTTP/2 200" or "HTTP/2 302"

# Rate-limit smoke probe (B1 — Sprint 107 W1). Confirms the rate-limiter is
# wired AND the test secret rotated correctly. Run as part of post-deploy.
RATE_LIMIT_TEST_SECRET="$(aws ssm get-parameter --name /<env>/RATE_LIMIT_TEST_SECRET \
  --with-decryption --query Parameter.Value --output text)" \
  bash /opt/<app>/scripts/post_deploy_smoke.sh
# Expected: tests/e2e/test_00_rate_limit_smoke.py passes (HTTP 429 after N requests)
```

#### 2. Review Recent Logs

```bash
# Check for errors in last hour
docker compose -f docker-compose.yml logs --since 1h 2>&1 | grep -i error

# Check for failed authentication attempts
docker compose -f docker-compose.yml logs --since 1h | grep -i "authentication failed\|invalid credentials"

# Check database connections
docker compose -f docker-compose.yml logs db --since 1h | grep -i "connection"

# Cross-reference with Sentry (preferred for application errors)
(source /opt/<app>/.env.ssm && python3 /opt/<app>/scripts/sentry_issues.py)
# Lists unresolved Sentry issues for STANDARD prod. Treat as authoritative for
# application-layer errors — log greps catch infrastructure stuff Sentry misses.
```

#### 3. Monitor Resource Usage

```bash
# CPU and memory
docker stats --no-stream

# Disk usage
df -h

# Docker disk usage
docker system df
```

### Weekly Tasks

#### 1. Review Audit Logs

```bash
# Connect to database
docker exec -it <db-container> psql -U postgres -d standard_iowire

# Check failed login attempts
SELECT 
  event_type, 
  COUNT(*) as count,
  event_data->>'email' as email
FROM audit_logs
WHERE success = false
  AND event_type LIKE '%login%'
  AND created_at > NOW() - INTERVAL '7 days'
GROUP BY event_type, event_data->>'email'
ORDER BY count DESC;

# Check unusual activity
SELECT 
  event_type,
  COUNT(*) as count,
  date_trunc('day', created_at) as day
FROM audit_logs
WHERE created_at > NOW() - INTERVAL '7 days'
GROUP BY event_type, day
ORDER BY day DESC, count DESC;
```

#### 2. Check SSL Certificates

```bash
# Check certificate expiration
echo | openssl s_client -servername standard.iowire.com -connect standard.iowire.com:443 2>/dev/null | openssl x509 -noout -dates

# Caddy auto-renews certificates, but verify they're valid
curl -vI https://standard.iowire.com 2>&1 | grep -i "expire\|issuer"
```

#### 3. Database Maintenance

```bash
# Clean up expired sessions
docker exec -it <db-container> psql -U postgres -d standard_iowire -c "SELECT cleanup_expired_sessions();"

# Check database size
docker exec -it <db-container> psql -U postgres -d standard_iowire -c "SELECT pg_size_pretty(pg_database_size('standard_iowire'));"

# Vacuum and analyze
docker exec -it <db-container> psql -U postgres -d standard_iowire -c "VACUUM ANALYZE;"
```

### Monthly Tasks

#### 1. Review Security

```bash
# Check for security updates (Ubuntu — apt, not yum)
ssh -i ~/code/tb_code/cert/standard-poc-key.pem ubuntu@standard.iowire.com
sudo apt-get update && sudo apt list --upgradable 2>/dev/null

# Update Docker images
cd /opt/<app>
docker compose -f docker-compose.yml pull
docker compose -f docker-compose.yml up -d

# Check for compromised credentials
# Review audit logs for suspicious patterns
```

#### 2. Verify Backups

```bash
# Check last backup (S3 is authoritative; local copy at /opt/<app>/backups for fast restore)
aws s3 ls s3://standard-iowire-backups/daily/ --human-readable | tail -10
ls -lht /opt/<app>/backups/ | head -10

# Test restore via the scripted drill (does NOT touch prod)
AWS_PROFILE=standard python3 /opt/<app>/scripts/backup/restore_drill.py --latest --verbose
```

#### 3. Performance Review

```bash
# Check slow queries (if pg_stat_statements enabled)
docker exec -it <db-container> psql -U postgres -d standard_iowire << 'EOF'
SELECT 
  query,
  calls,
  total_time / 1000 as total_seconds,
  mean_time / 1000 as mean_seconds
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;
EOF

# Review application error rates
docker compose -f docker-compose.yml logs --since 30d 2>&1 | grep -i error | wc -l
```

---

## Monitoring & Health Checks

### Health Check Endpoints

| Service | Endpoint | Expected Response |
|---------|----------|-------------------|
| Caddy (Prometheus metrics) | `http://localhost:8080/metrics` | text/plain 200 |
| App (`/health`) | `http://localhost:5000/health` | JSON `{"status":"ok",...}` (200) |
| External | `https://standard.iowire.com/health` | JSON `{"status":"ok",...}` (200) |
| LiveKit ingress | `http://localhost:8080/` (livekit-ingress) | 200 |

> Ports 5001 / 4444 / 5002 (former portal/metrics/admin services) are NOT in use. Any reference to them is stale.

### Automated Monitoring

**Uptime Monitor**: `scripts/uptime_monitor.sh`
- Runs every 2 minutes via cron
- Checks internal health via `docker exec` (app container `/health` endpoint)
- Checks external HTTPS endpoint (`https://standard.iowire.com/health`)
- Monitors container status, disk space, memory usage
- Alerts via Slack webhook + email after 3 consecutive failures
- Recovery notifications when service comes back up

**Metrics Collector**: `scripts/metrics_collector.sh`
- Runs every 5 minutes via cron
- Scrapes Prometheus `/metrics` endpoint via `docker exec`
- Collects container CPU/memory, Redis keys/memory, PgBouncer connections
- Logs JSONL to `/var/log/standard-metrics.jsonl`
- Threshold-based alerting for high error rates, latency, and memory usage

**Cron Setup** (on production server):
```bash
# Add to crontab: crontab -e
*/2 * * * * /opt/<app>/scripts/uptime_monitor.sh >> /var/log/uptime_monitor.log 2>&1
*/5 * * * * /opt/<app>/scripts/metrics_collector.sh >> /var/log/metrics_collector.log 2>&1
```

### Key Metrics to Monitor

**System Metrics**:
- CPU usage < 70% average
- Memory usage < 80%
- Disk usage < 80%
- Load average < 2.0 (for 2 vCPU)

**Application Metrics**:
- Response time < 500ms (95th percentile)
- Error rate < 1%
- Active users (via sessions table)
- Failed login attempts < 10/hour

**Database Metrics**:
- Connection count < 80 (max pool size: 100)
- Query time < 100ms (average)
- Database size growth rate
- Active sessions count

---

## Alert Runbooks

Procedures invoked when a Grafana Cloud alert fires in `#alerts-prod`. Each
section is the target of the `runbook_url` annotation in
[`grafana/alerts/rules.yml`](../../grafana/alerts/rules.yml). Per the
monitoring spec's noise budget, no runbook = no alert.

#### Alert S3: Disk Space Low

**Severity:** P2 · **Trigger:** `(node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.15` for 5+ min on `/host/root`.

**Why this matters:** Single-EC2 stack — when disk fills, PostgreSQL stops accepting writes (silent ENOSPC), Loki WAL flushes fail, file uploads 5xx. No exception is raised in Python, so Sentry will NOT fire. This is a complete outage with no automated recovery.

**Triage (in order):**

1. SSH to prod: `ssh -o IdentitiesOnly=yes -i ~/code/tb_code/cert/standard-poc-key.pem ubuntu@standard.iowire.com`
2. Confirm disk usage: `df -h /` — note `% used` per partition.
3. Identify largest consumers: `sudo du -sh /var/log/* /var/lib/docker/* /opt/<app>/* 2>/dev/null | sort -h | tail -10`
4. Apply the matching remediation:
   - **Docker logs (most common):** `sudo docker system prune -f --volumes` (frees image+log layers).
   - **Postgres data growth:** check `SELECT pg_size_pretty(pg_database_size('iowire'));` against historical baseline. If >2× baseline, investigate orphan/temp tables. NEVER `DROP DATABASE`.
   - **App logs (`/var/log`):** rotate logs immediately: `sudo logrotate -f /etc/logrotate.conf`. Verify cron-driven rotation is wired (Sprint 107 B2 closed this gap; check `/etc/logrotate.d/standard-backup`).
   - **Loki WAL backup:** `docker compose -f /opt/<app>/docker-compose.yml logs alloy --tail=200` for stuck flushes.
5. Confirm recovery: `df -h /` shows >25% free, alert auto-resolves within 5 min.

**Escalate if:** disk fills again within 24h after cleanup (suggests runaway write loop) — page Alex with `[STANDARD-P1]` in Slack DM.

---

#### Alert A2: High 5xx Rate

**Severity:** P1 · **Trigger:** `rate(http_errors_total[5m]) / rate(http_requests_total[5m]) > 0.05` for 5+ min.

**Why this matters:** ≥5% of customer-facing requests are erroring. Sentry will surface specific exceptions; this alert fires on the rate so you know the magnitude before reading 50+ Sentry events.

**Triage (in order):**

1. Cross-reference Sentry: <https://sentry.io/organizations/iowire/issues/?statsPeriod=15m&query=is%3Aunresolved> — group by exception type. The most common error in the last 15 min is the root cause candidate.
2. Check container health: `docker compose -f /opt/<app>/docker-compose.yml ps` — any unhealthy/restarting? Recent OOM kill? `docker compose ... logs app --tail=200 | grep -iE "error|killed|oom"`.
3. Check upstream dependencies in this order:
   - **Database:** prod DB is dedicated RDS (`standard-prod-db`), reached through pgbouncer — reuse the app container's owner `DATABASE_URL` (see [Database Access](#access-information) above): `docker compose -f /opt/<app>/docker-compose.yml -f /opt/<app>/docker-compose.production.yml exec app sh -lc 'psql "$DATABASE_URL" -c "SELECT count(*) FROM pg_stat_activity;"'` (should be <80% of pool). Absolute `-f` paths so the command works from any directory post-SSH.
   - **Redis:** `docker compose ... exec redis redis-cli INFO replication` and `INFO memory`.
   - **Stripe:** check Stripe Dashboard → Developers → Logs for elevated 4xx/5xx rate from their side.
4. If a deploy landed in the last hour, consider rollback: `cd /opt/<app> && bash scripts/promote.sh rollback`.

**Escalate if:** error rate stays >5% after rollback — page Tiffy + Alex.

---

#### Alert S1: Celery Queue Backup

**Severity:** P2 · **Trigger:** `celery_queue_depth{queue="celery"} > 50` for 10+ min.

**Why this matters:** Background work (Stripe webhooks, payout reconciliation, email sends) is queuing up. Customers won't see immediate failures, but workflows that depend on async (e.g., subscription confirmation emails, payout settlement) will be delayed.

**Triage (in order):**

1. Snapshot queue state: `docker compose -f /opt/<app>/docker-compose.yml exec redis redis-cli LLEN celery`.
2. Check worker health: `docker compose ... ps celery-worker celery-beat` — both should be `Up (healthy)`. If unhealthy, restart: `docker compose ... restart celery-worker`.
3. Check what's stuck: `docker compose ... exec celery-worker celery -A celery_app inspect active --json | jq` — look for tasks stuck >5min.
4. Check downstream rate limits:
   - **Stripe:** historical 5xx in their logs cause our retry backoff.
   - **SES (email):** sandbox cap is 200/24h.
5. If queue depth >500 OR growing despite healthy worker: scale a 2nd worker temporarily — `docker compose ... up -d --scale celery-worker=2`. Revert when queue drains.

**Threshold tuning (OQ7 in spec):** 50 is a launch-time guess. After the first 4 weeks of prod traffic, tune to `p95(queue_depth) + 2*stddev` over a 7-day window.

---

#### Alert S1C: Critical Async Queue Backup

**Severity:** P1 · **Trigger:** staging `celery_queue_depth{queue="critical"} > 5` for 5+ min.

**Redirect:** This is part of the staging async migration. Use [`async-task-staging-rollout.md`](../runbooks/async-task-staging-rollout.md) for rollout posture, rollback, and seven-day gates.

**Triage:** Check the critical worker first, then money-adjacent tasks: payout, Stripe reconciliation, subscription reconciliation. If the critical queue is growing, stop any staging SQS primary/shadow experiment, leave `TASK_BACKEND=celery`, and restart only the affected staging worker after collecting `celery inspect active` and worker logs.

---

#### Alert S1R: Routed Async Queue Backup

**Severity:** P2 · **Trigger:** staging `celery_queue_depth{queue=~"default|bulk|media"} > 100` for 15+ min.

**Redirect:** This is part of the staging async migration. Use [`async-task-staging-rollout.md`](../runbooks/async-task-staging-rollout.md) for split-worker rollout and rollback steps.

**Triage:** Identify `queue` from the alert, inspect the matching staging worker logs, and compare `task_publish_total` with `task_execution_total` by queue/backend. If only one routed queue is backed up, scale or restart that worker class instead of restarting every worker.

---

#### Alert S1M: Media Transcode Backlog

**Severity:** P2 · **Trigger:** `celery_queue_depth{queue="media"} > 30` for 5+ min (prod-inclusive, per-host).

**What it means:** The on-box Celery `media` queue is core audio transcode (ffmpeg 2-pass EBU-R128 loudnorm + `libmp3lame`). At the single-worker floor it drains ~2-3 uploads/min, so a sustained depth >30 is ~10+ minutes of backlog and growing. Artist **time-to-playable** is degrading and host review queues fill with tracks that aren't playable yet. This is deliberately tighter than S1R (>100/15m) so it gives lead time.

**NOT a money incident.** Under prod auto-capture (`submission_manual_capture` OFF), money is captured at `confirmPayment` **before** transcode. A media backlog delays playability/review, not capture. Do not escalate to payments.

**Triage / remediation:**
1. Confirm it's a real backlog, not a scrape blip: `celery_queue_depth{queue="media"}` trend over 15m, and `celery-worker` health (`docker ps`, `/health`).
2. Check whether one slow/stuck job is head-of-lining: worker logs for the `transcode[...]: TIMINGS` lines; a single wedged encode blocks the lane.
3. If it's genuine load (launch/drop burst) and depth is still climbing: **arm the pre-staged C-class resize** (the launch burst valve). A bigger box is the intended lever, not shipping new infra under load; the C resize also relieves the 2-vCPU contention the single-box floor cannot. The specific resize target and steps are pre-staged and rehearsed ahead of launch — see the media-scale launch-readiness plan (task #8); until that lands, escalate to the operator rather than improvising a resize under load.
4. Do NOT throw workers at it blindly on the 2-vCPU box: more media concurrency on a saturated box starves the live tier (hosts/audience). Prefer the resize.

**Note:** The raw signals (`celery_queue_depth{queue="media"}`, and transcode duration via `celery_task_duration_seconds_total{...,queue="media"}` ÷ `celery_task_events_total`) already ship from app `/metrics`; this alert only adds the tuned, prod-inclusive threshold. Activation requires importing the rule via the Grafana-managed Alerting provisioning API (see `grafana/alerts/rules.yml` header).

---

#### Alert ATE1: Async Task Publish Errors

**Severity:** P1 · **Trigger:** staging `task_publish_total{result="error"}` for 5+ min.

**Redirect:** Use [`async-task-staging-rollout.md`](../runbooks/async-task-staging-rollout.md), especially the preflight, SQS smoke, and rollback sections.

**Triage:** Use alert labels `backend` and `queue`. For `backend="celery"`, verify Redis and Celery broker settings. For `backend="sqs"`, verify queue URL, IAM send permission, and AWS region. Keep primary task execution on Celery unless this is an explicitly approved staging cutover.

---

#### Alert ATE1S: Async Task Shadow Publish Errors

**Severity:** P2 · **Trigger:** staging `task_publish_shadow_total{result="error"}` for 5+ min.

**Redirect:** Use [`async-task-staging-rollout.md`](../runbooks/async-task-staging-rollout.md) for shadow queue topology. Shadow queues must stay separate from primary queues.

**Triage:** Disable `TASK_PUBLISH_SHADOW` if errors are noisy, then verify `TASK_SHADOW_BACKEND=sqs`, `TASK_SHADOW_QUEUE_<CLASS>_URL`, and the dedicated shadow DLQ wiring. This is telemetry loss, not primary task loss, but it invalidates migration readiness.

---

#### Alert ATE2: Async Task Execution Errors

**Severity:** P2 · **Trigger:** staging `task_execution_total{result=~"error|failure"}` for 5+ min.

**Triage:** Use the async migration dashboard to break down by task/backend/queue, then trace the failing task in Sentry and worker logs using the correlation id. If `backend="sqs"`, stop the consumer before replaying messages and inspect the matching DLQ first.

---

#### Alert ATE3: Celery Worker Task Failures

**Severity:** P2 · **Trigger:** staging `celery_task_events_total{result="failure"}` for 5+ min.

**Triage:** Inspect the queue label, then check the staging worker logs and Sentry for the failing task name. This alert is queue-aggregated to avoid cardinality blowups; use dashboard task labels for drill-down, not alert fanout.

---

#### Alert ATE6: Async Metrics Collector Down

**Severity:** P2 · **Trigger:** staging `metrics_collector_up{collector="redis_async"} == 0` for 5+ min.

**Triage:** Treat migration dashboards as incomplete. Check app `/metrics`, Redis connectivity, and app logs for `metrics_export_failures_total` operations. Fix telemetry before using async metrics as a readiness gate.

---

#### Alert ATE7: Async Metrics Export Failures

**Severity:** P2 · **Trigger:** staging `metrics_export_failures_total{operation!="none"}` increased in 10 min.

**Triage:** Use labels `collector` and `operation`, then inspect app logs around the scrape. This catches partial metric loss while `/metrics` still returns 200; do not tune rollout gates from a partial scrape.

---

#### Alert ATE8: Tasking Metric Failures

**Severity:** P2 · **Trigger:** staging `tasking_metric_failures_total{operation!="none"}` increased in 10 min.

**Triage:** Check whether publish/execution counters failed to persist or render. SQS smoke readiness depends on these counters, so rerun the smoke only after the metric failure is fixed.

---

#### Alert ATE9: Celery Metric Write Failures

**Severity:** P2 · **Trigger:** staging Loki log line `celery metric write failed kind=` appears.

**Triage:** Search Loki or worker logs for the stable `kind=` value. Worker execution may be healthy while lifecycle, revocation, beat-depth, or execution metrics are missing; fix the write path before trusting Celery dashboards.

---

#### Alert ATE4: SQS Oldest Message Age

**Severity:** P2 · **Trigger:** staging primary SQS oldest message age >5 min for 10+ min.

**Redirect:** Use [`async-task-staging-rollout.md`](../runbooks/async-task-staging-rollout.md) for SQS smoke, delay limits, and rollback.

**Triage:** Confirm whether the consumer is intentionally disabled. If not, check consumer logs, IAM receive/delete permissions, visibility timeout, and whether one poison message is repeatedly failing. Do not promote any primary SQS traffic while this alert is active.

---

#### Alert ATE5: SQS DLQ Messages

**Severity:** P2 · **Trigger:** staging primary or shadow async DLQ visible messages >0 for 5+ min.

**Redirect:** Use [`async-task-staging-rollout.md`](../runbooks/async-task-staging-rollout.md#dlq-inspection-and-replay) for the exact inspect/replay commands and Terraform-derived DLQ env setup.

**Triage:** Inspect first, replay only after the failure mode is understood, and add `--delete` only after the replay target accepted the message. Never replay shadow DLQ messages into primary queues.

---

#### Alert S3: Celery Revocation Rate High

**Severity:** P2 · **Trigger:** staging `rate(celery_task_revoked_total[15m]) > 0.2` for 30+ min.

**Triage:** Find the dominant `task` in `celery_task_revoked_total`. A sustained revocation rate means tasks are expiring before workers can execute them, even if queue depth looks normal. Reduce the producing beat frequency, raise worker capacity, or pause the offending staging fan-out before it hides migration results.

---

#### Alert S4: Celery Beat Publish Depth Spike

**Severity:** P2 · **Trigger:** staging `celery_beat_depth_at_publish > 200` for 15+ min.

**Triage:** Use the `task` label to find which beat publish observed backlog. This is the leading indicator for S3 revocations; inspect beat logs, worker active tasks, and routed queue depth before changing schedules.

---

#### Alert S5: Celery Beat Schedule Missed

**Severity:** P2 · **Trigger:** staging beat task age exceeds 3x its expected interval for 5+ min.

**Triage:** Check `celery-beat` logs, then inspect worker scheduled and active tasks. If the task ran but did not update its Redis heartbeat, investigate task failure or heartbeat write failure before assuming the scheduler is dead.

---

#### Alert S2: DB Pool Near Exhaustion

**Severity:** P1 · **Trigger:** `db_pool_checked_out / db_pool_size > 0.85` for 5+ min.

**Why this matters:** SQLAlchemy connection pool is approaching saturation. Once full + overflow exhausted, requests will queue waiting for a connection, then 500 with `TimeoutError`. P1 because customer-facing requests start failing fast.

**Triage (in order):**

1. Identify long-running queries — prod DB is dedicated RDS (`standard-prod-db`) reached through pgbouncer, so run psql from the app container with its owner `DATABASE_URL` (see [Database Access](#access-information)): `docker compose -f /opt/<app>/docker-compose.yml -f /opt/<app>/docker-compose.production.yml exec app sh -lc 'psql "$DATABASE_URL" -c "SELECT pid, age(clock_timestamp(), query_start) AS age, state, left(query, 100) AS query FROM pg_stat_activity WHERE state != \$\$idle\$\$ ORDER BY age DESC LIMIT 10;"'` (`$$idle$$` is Postgres dollar-quoting — avoids nested single-quote escaping)
2. Identify long-running transactions (held connection blocking pool return): same query but filter `WHERE state IN ('idle in transaction', 'idle in transaction (aborted)')`. Anything older than 60s is suspicious.
3. Kill a stuck query if confirmed safe to abort: `SELECT pg_cancel_backend(<pid>);`. Avoid `pg_terminate_backend` unless cancel fails.
4. Check pgbouncer side: `docker compose ... exec pgbouncer psql -h localhost -p 6432 -U postgres pgbouncer -c "SHOW POOLS;" -c "SHOW CLIENTS;"`.
5. If no specific stuck query but pool is genuinely under load, scale: bump `SQLALCHEMY_POOL_SIZE` (default 20) + restart app. Document the bump in `tasks/todo.md` and revisit query patterns.

**Escalate if:** pool stays >85% after killing all stuck queries — page Alex.

---

#### Alert A1-ext: External Reachability Failed

**Severity:** P1 · **Trigger:** Grafana Synthetic Monitoring probe `probe_success{job="standard-iowire-health"}` reports failure from at least one region for 5+ min.

**Why this matters:** Customers cannot reach the platform from outside our VPC. Independent of internal metrics — this rule is the canary for Caddy/AWS/DNS-level outages that look fine to internal monitoring (because internal probes never leave the box).

**Triage (in order):**

1. Identify the failing region: alert label `{{ $labels.probe }}` (Atlanta or Frankfurt).
2. Independently verify from your machine: `curl -v -m 10 https://standard.iowire.com/health` — expect `200 OK` with `{"status":"ok",...}` body.
3. If both regions fail (alert label shows multiple) → outage is upstream of the probe vendors:
   - SSH and check Caddy: `docker compose -f /opt/<app>/docker-compose.yml logs caddy --tail=100`
   - Check DNS: `dig +short standard.iowire.com` should resolve to AWS EIP `<DB-EIP>`.
   - Check EC2 status: AWS Console → EC2 → instance health.
4. If single region fails (e.g., only Frankfurt) → likely transient regional routing or trans-Atlantic blip. Wait 5 more min. If sustained 30+ min from a single region, file a ticket with the probe vendor; not a STANDARD outage.
5. While debugging, check `https://standard.iowire.com/health` from a 4G phone hotspot — confirms it's not a corp-network DNS quirk.

**Escalate if:** all regions fail >10 min — page Alex AND Tiffy. Treat as full outage; declare P0 in `#alerts-prod`.

---

#### Sentry: Dispute commit failures (stuck at transient state)

**Severity:** P1 · **Trigger:** Any of these Sentry messages fires:
`DISPUTE_RESOLVED_BUT_COMMIT_FAILED`, `APPEAL_RESOLVED_BUT_COMMIT_FAILED`,
`SLA_AUTO_REFUND_COMMIT_FAILED`, `DISPUTE_PARTIAL_TRANSFER_LEG1_COMMIT_FAILED`,
`DISPUTE_HEAL_COMMIT_FAILED`, `DISPUTE_RECOVERY_REVERT_COMMIT_FAILED`.

**Why this matters:** Atomic resolution chain failed at Phase 3 (commit). Stripe-side money may have already moved, but DB state + audit row did not write — dispute is stuck at `resolving` / `resolution_failed` / `partially_resolved` / `appeal_resolving`. Refund-path cases self-heal via `charge.refunded` webhook within 1–60s; split-path and recovery-script failures require operator action.

**Triage:** Full procedure in [`dispute-atomic-recovery.md`](./dispute-atomic-recovery.md). TL;DR: read Sentry extras for `dispute_id` + Stripe id, verify Stripe dashboard, wait 5 min for webhook self-heal, then run `scripts/recover_dispute_revert.py --dry-run` before applying recovery.

**Escalate if:** dispute has both `stripe_transfer_provider_id` and `stripe_transfer_coord_id` partially set (split-path stuck) — page payments on-call, do not target-state `resolved`.

**SLA:** Investigate within 1h, resolve within 4h. SOC 2 CC7.2.

---

#### Alert M1: Alloy Scrape Down

**Severity:** P1 · **Trigger:** `absent_over_time(up{job="prometheus.scrape.flask"}[5m]) > 0` for 5+ min. `noDataState=OK` is intentional. PromQL semantics: `absent_over_time` returns an **empty vector** (→ NoData) when the input series HAS samples — i.e., during healthy Alloy operation. Flipping `noDataState=Alerting` would false-fire every evaluation cycle while Alloy is healthy. When samples ARE absent (Alloy down), the query returns `1` and the `> 0` threshold fires normally. See `<vault>/Lessons/grafana-cloud-mimir-ruler-grafana-am-wiring.md` §gotchas (vault is Obsidian-only on Alex/Tiffy laptops; no public URL).

**Why this matters:** Alloy is the only metric shipper. If it stops scraping `app:5000/metrics`, every other alert in this runbook silently transitions to `NoData=OK` — including the P1 alerts. The platform could be on fire and no Grafana rule would notify. **This alert is the dead-man's switch for the entire alerting system.**

**Triage (in order):**

1. SSH prod: `ssh -i ~/code/tb_code/cert/standard-poc-key.pem ubuntu@standard.iowire.com`.
2. Check Alloy container: `cd /opt/<app> && docker compose -f docker-compose.yml ps alloy` — expect `Up (healthy)`.
3. Tail logs for the most recent failure mode: `docker compose -f docker-compose.yml logs alloy --tail=200 | grep -iE "error|fail|panic"`. Three common causes:
   - **OOMkilled** — container memory limit hit. `docker inspect $(docker compose ps -q alloy) | jq '.[0].State.OOMKilled'`. Bump memory limit in compose, restart.
   - **`prometheus.remote_write` 401/403** — `GRAFANA_PROMETHEUS_TOKEN` rotated and SSM not refreshed. Verify the key is present without leaking its value: `grep -c '^GRAFANA_PROMETHEUS_TOKEN=' /opt/<app>/.env.ssm` (POSIX grep, no ripgrep dep; returns `1` if set). If `0`, re-pull from SSM and restart Alloy.
   - **Flask `/metrics` unreachable** — verify Flask is alive: `docker compose exec app curl -s http://localhost:5000/metrics | head -5`. If Flask is down, this is actually an A2/S2-class incident — fix Flask first.
4. Restart Alloy if root cause is transient: `docker compose -f docker-compose.yml restart alloy`. Wait 60s, then re-query `up{job="prometheus.scrape.flask"}` in Grafana Explore — expect value 1.
5. **CRITICAL OUT-OF-BAND CHECK** while Alloy is down: every other dashboard is lying. Spot-check critical systems manually:
   - Disk: `df -h /` (compare against DiskSpaceLow's 15% threshold).
   - Postgres: `docker exec <db-container> psql -U postgres -d standard_iowire -c "SELECT count(*) FROM pg_stat_activity"`.
   - Celery: `docker exec <redis-container> redis-cli LLEN celery`.

**Escalate if:** Alloy restart loop after 3 attempts — page Alex. Investigate container memory limit + OOM history in the host's `journalctl -k`.

---

#### Alert E1: Stripe Recon Stale

**Severity:** P2 · **Trigger:** `stripe_recon_last_run_age_seconds > 90000` (25h) for 1h.

**Why this matters:** The nightly Stripe reconciliation Beat task is our only out-of-band check that internal ledger state matches Stripe's source of truth. If it hasn't run, drift accumulates silently and the next invoice run could send wrong totals to customers.

**Current state (2026-05-20):** This alert is **dormant in `NoData=OK`**. The metric `stripe_recon_last_run_age_seconds` is not yet emitted from `boom-queue/metrics_endpoint.py` — the gauge ships in a future sprint (tracked in the W2 backlog at `<vault>/Compliance/Evidence/sprint-107-w1-observability-close-out.md`). The alert is wired ahead of the metric so the runbook URL resolves; flipping `noDataState=Alerting` is the cutover signal once the gauge ships.

**Triage (in order, once the gauge is live):**

1. Is the Beat task scheduled? `docker compose -f /opt/<app>/docker-compose.yml exec celery-worker celery -A celery_app inspect scheduled | grep -A2 stripe-recon` (target the worker, not beat — `inspect scheduled` queries worker task reservations).
2. Is the worker consuming? `docker compose -f docker-compose.yml logs celery-worker --tail=200 | grep stripe-recon`.
3. Check the Redis heartbeat key the gauge reads from: `docker compose -f docker-compose.yml exec redis redis-cli GET std:stripe_recon:last_run`. Convert the unix timestamp; if >25h ago, the task ran but failed before updating the key — check worker logs for the actual exception.

**Resolution paths:**

- **Beat not scheduling** → `docker compose -f docker-compose.yml restart celery-beat`. Schedule lives in `boom-queue/celery_app.py`; if removed accidentally, restore.
- **Worker not running** → `docker compose -f docker-compose.yml ps celery-worker`; restart if missing.
- **Task running but failing** → fix the underlying error. Until fixed, trigger a synchronous manual recon before the next invoice run: `docker compose -f docker-compose.yml exec celery-worker python -c "from tasks.stripe_reconciliation import run_daily_recon; print(run_daily_recon.apply().get())"`. This runs in the worker container's Python process (bypassing the broker queue) and prints the recon ledger; blocks until completion. The task lives at `boom-queue/tasks/stripe_reconciliation.py:182`; confirm the Redis heartbeat key `std:stripe_recon:last_run` is updated afterward. (Do NOT use `celery ... call ... run_daily_recon` — that only enqueues and returns a UUID without waiting; operator won't see the ledger.)

**Escalate if:** recon has been stale >48h AND an invoice run is scheduled in the next 24h — that's revenue-at-risk; halt the invoice cron and page Alex.

---

#### Alert M2: Active Series Approaching Cap

**Severity:** P3 · **Trigger:** `scalar(count({__name__!=""})) > 7000` for 1h (70% of the 10K Mimir free-tier cap).

**Why this matters:** This is the cost re-evaluation trigger per `<vault>/Decisions/2026-05-20-observability-stack-90d.md` §Re-evaluation Triggers. Hitting the 10K cap silently drops series — alerts stop firing, dashboards go blank. The 30% buffer exists so we can audit and act before that.

**Triage (in order):**

1. Top metrics by cardinality (paste into Grafana Explore → Mimir datasource):
   ```promql
   topk(20, count by (__name__) ({__name__!=""}))
   ```
   Anything new in the top-20 vs. last week is the suspect.
2. Recent commits adding or modifying metrics:
   ```bash
   git log --oneline --since="7 days ago" -- 'boom-queue/metrics_endpoint.py' 'alloy/config.alloy'
   ```
3. Per-metric label fan-out — find the cardinality bomb:
   ```promql
   topk(20, count by (__name__, job)({__name__!=""}))
   ```
   Look for any metric with `path=`, `user_id=`, or other unbounded labels.

**Resolution paths:**

- **Cardinality bomb from a new metric** → revert the offending commit OR rewrite the label scheme: bucket high-cardinality labels into `_high|_normal|_low`, drop user IDs, switch user-scoped counters to histograms.
- **Organic growth (no single bomb)** → revenue floor check. Per the Decision doc, GC Pro is $19/mo for 200K series. If MRR > $50/mo → upgrade tier; else → rescope the worst-offender metric.
- **Stale series from removed deploys** → Mimir auto-evicts after 5h staleness; if this fires during a deploy window, wait 6h and re-check before acting.

**CRITICAL:** Do NOT add more metrics or labels while this alert is firing. Every new series accelerates the cap hit and may silently drop existing alerts mid-investigation.

**Escalate if:** active series > 9,000 → 90% of cap; series eviction may have already started. Page Alex.

---

## Troubleshooting

### Common Issues

#### Issue 1: Site Not Responding (502 Bad Gateway)

**Symptoms**:
- Curl returns 502 error
- Browser shows "Bad Gateway"
- Users cannot access site

**Diagnosis**:
```bash
cd /opt/<app>

# Check if backend service is running
docker compose -f docker-compose.yml ps

# Check backend logs (the single Flask app is the `app` service)
docker compose -f docker-compose.yml logs --tail=50 app

# Check if service is listening on port
docker compose -f docker-compose.yml exec app netstat -tlnp | grep 5000
```

**Solution**:
```bash
# Restart the affected service
docker compose -f docker-compose.yml restart app

# If restart doesn't work, check for crashes
docker compose -f docker-compose.yml logs app | grep -i "error\|crash\|exception"

# Check for resource exhaustion
docker stats --no-stream

# If all else fails, restart all services (use rollback path if a recent deploy is suspect)
docker compose -f docker-compose.yml restart
# OR
bash /opt/<app>/scripts/promote.sh rollback prod
```

#### Issue 2: Database Connection Errors

**Symptoms**:
- Applications show "connection refused"
- Logs show "could not connect to server"
- 500 errors on all sites

**Diagnosis**:
```bash
# Check if database is running
docker compose -f docker-compose.yml ps db

# Check database logs
docker compose -f docker-compose.yml logs db --tail=50

# Test database connection
docker exec -it <db-container> psql -U postgres -d standard_iowire -c "SELECT 1;"

# Check connection count
docker exec -it <db-container> psql -U postgres -d standard_iowire -c "SELECT count(*) FROM pg_stat_activity;"
```

**Solution**:
```bash
cd /opt/<app>

# Restart database
docker compose -f docker-compose.yml restart db

# Wait for healthy status
watch docker compose -f docker-compose.yml ps db

# Restart dependent services
docker compose -f docker-compose.yml restart app celery-worker celery-beat

# If database won't start, check disk space
df -h

# Check for corrupted data files
docker compose -f docker-compose.yml logs db | grep -i "corrupt\|error"
```

#### Issue 3: SSL Certificate Issues

**Symptoms**:
- Browser shows "Your connection is not private"
- Curl shows SSL certificate errors
- Certificate expired warning

**Diagnosis**:
```bash
# Check certificate expiration
echo | openssl s_client -servername standard.iowire.com -connect standard.iowire.com:443 2>/dev/null | openssl x509 -noout -dates

# Check Caddy logs
docker compose -f docker-compose.yml logs caddy --tail=100 | grep -i "certificate\|acme\|tls"
```

**Solution**:
```bash
cd /opt/<app>

# Caddy auto-renews, but if it fails:

# Restart Caddy
docker compose -f docker-compose.yml restart caddy

# Check Caddy can reach Let's Encrypt
docker compose -f docker-compose.yml exec caddy curl -I https://acme-v02.api.letsencrypt.org/directory

# Check ports 80 and 443 are open
sudo netstat -tlnp | grep -E ":80|:443"

# If renewal fails, check rate limits (5 per week)
# Wait and try again, or use staging environment

# Force renewal (reload Caddy from its mounted Caddyfile)
docker compose -f docker-compose.yml exec caddy caddy reload --adapter caddyfile --config /etc/caddy/Caddyfile
```

#### Issue 4: High CPU Usage

**Symptoms**:
- Server sluggish
- `top` shows high CPU usage
- Applications slow to respond

**Diagnosis**:
```bash
# Check which container is using CPU
docker stats --no-stream

# Check top processes
top -bn1 | head -20

# Check for runaway queries
docker exec -it <db-container> psql -U postgres -d standard_iowire -c "SELECT pid, query, state FROM pg_stat_activity WHERE state = 'active';"
```

**Solution**:
```bash
cd /opt/<app>

# If a specific service is the culprit
docker compose -f docker-compose.yml restart <service_name>

# Kill long-running queries (prefer pg_cancel_backend over pg_terminate_backend; see Alert S2)
docker exec -it <db-container> psql -U postgres -d standard_iowire -c "SELECT pg_cancel_backend(pid) FROM pg_stat_activity WHERE query_start < NOW() - INTERVAL '5 minutes' AND state = 'active';"

# Check for DOS attack
docker compose -f docker-compose.yml logs caddy | tail -1000 | cut -d' ' -f1 | sort | uniq -c | sort -rn | head

# Consider scaling up instance if sustained high usage (R4 t3.large upgrade still pending — see Sprint 107 R4)
```

#### Issue 5: Out of Disk Space

**Symptoms**:
- Database writes fail
- Docker builds fail
- Applications crash with "no space left on device"

**Diagnosis**:
```bash
# Check disk usage
df -h

# Find large files
du -sh /* | sort -rh | head -10

# Check Docker usage
docker system df
```

**Solution**:
```bash
# Clean up Docker
docker system prune -a --volumes

# Clean up old logs
sudo find /var/log -type f -name "*.log" -mtime +30 -delete

# Clean up old backups (canonical local path; S3 retention handled by bucket lifecycle)
find /opt/<app>/backups -type f -mtime +7 -delete

# Resize EBS volume if needed (AWS console)
# Then resize filesystem
sudo growpart /dev/xvda 1
sudo resize2fs /dev/xvda1
```

#### Issue 6: Authentication Not Working

**Symptoms**:
- Users cannot login
- "Invalid credentials" for correct passwords
- Sessions expire immediately

**Diagnosis**:
```bash
# Check database connection
docker exec -it <db-container> psql -U postgres -d standard_iowire -c "SELECT COUNT(*) FROM users;"

# Check Redis connection
docker compose -f docker-compose.yml exec redis redis-cli -a ${REDIS_PASSWORD} ping

# Check audit logs
docker exec -it <db-container> psql -U postgres -d standard_iowire -c "SELECT * FROM audit_logs WHERE event_type LIKE '%login%' ORDER BY created_at DESC LIMIT 10;"

# Check application logs
docker compose -f docker-compose.yml logs app | grep -i "login\|auth"
```

**Solution**:
```bash
cd /opt/<app>

# Restart Redis (sessions)
docker compose -f docker-compose.yml restart redis

# Clear sessions
docker compose -f docker-compose.yml exec redis redis-cli -a ${REDIS_PASSWORD} FLUSHDB

# Restart the app (only one Flask service today — no separate portal/metrics/admin)
docker compose -f docker-compose.yml restart app

# Check SECRET_KEY environment variable
docker compose -f docker-compose.yml exec app env | grep SECRET_KEY

# Verify password hashes in database
docker exec -it <db-container> psql -U postgres -d standard_iowire -c "SELECT email, substring(password_hash, 1, 20) FROM users LIMIT 5;"
```

#### Issue 7: Sentry DSN Missing in Production

If the app refuses to boot with `RuntimeError: SENTRY_DSN required in production`:

**Root cause:** SENTRY_DSN env var is unset or empty. The assertion lives at `boom-queue/app.py:305-347` and is intentional — see `Sessions/2026-04-22-r6-sentry-dsn-assertion.md` for context.

**Normal fix:** Restore SENTRY_DSN from SSM:
```bash
aws ssm get-parameter --name /<env>/SENTRY_DSN --with-decryption --query Parameter.Value
# Paste into .env or .env.prod
```

**Emergency escape hatch (use ONLY if SSM is down):**
```bash
export ALLOW_MISSING_SENTRY_DSN=true
# Container boots but observability is OFF. File an incident ticket immediately.
```

**DO NOT:** set `ALLOW_MISSING_SENTRY_DSN=true` permanently. There is no TTL. A forgotten override silently disables Sentry for the entire deployment lifetime. Remove the flag as soon as SSM access is restored.

---

## Backup & Recovery

### Backup Procedures

#### Daily Automated Backup

**Canonical script**: `scripts/backup_db.sh` (lives in this repo, deployed to `/opt/<app>/` on prod).

**Format**: pg_dump custom format (`-Fc`, zlib-compressed) — NOT plain SQL. Restores require `pg_restore -Fc`, not `psql <`.

**Destination**: `s3://standard-iowire-backups/daily/standard_iowire_YYYYMMDD-HHMMSS.dump`.

**Cron schedule**: daily on prod host. Retains 7 days locally at `/opt/<app>/backups/`; S3 retention is whatever the bucket lifecycle policy dictates.

**Failure alerting**: the script traps `ERR` and publishes to SNS topic `arn:aws:sns:us-east-1:443215160956:standard-iowire-alerts`. See `scripts/backup_db.sh:19-23`.

**Freshness check**: `scripts/check_backup_freshness.sh` runs as a scheduled agent and alerts if the latest S3 object is older than 25 hours.

**Invoke manually** (on prod host):
```bash
sudo -u ubuntu /opt/<app>/scripts/backup_db.sh
```

#### Manual ad-hoc backup

When you need a one-off dump outside the daily schedule (e.g., before a risky migration):

```bash
# Full database, custom format — matches production dump format
docker compose -f docker-compose.yml exec -T db \
  pg_dump -U postgres -d standard_iowire -Fc --no-owner --no-acl \
  > backup-manual-$(date +%Y%m%d-%H%M%S).dump

# Single table, plain SQL (human-readable, for quick inspection)
docker compose -f docker-compose.yml exec -T db \
  pg_dump -U postgres -d standard_iowire -t users --no-owner --no-acl \
  > users-$(date +%Y%m%d).sql
```

Do NOT use `pg_dump | gzip` — the canonical format is `-Fc` so the restore path stays uniform.

### Recovery Procedures

#### Restore from Backup

**⚠️ WARNING: This will overwrite current data!**

Backups produced by `scripts/backup_db.sh` are **pg_dump custom format** (`-Fc`,
zlib-compressed). They are NOT gzipped SQL — do not use `gunzip` or pipe to
`psql`. Use `pg_restore -Fc`.

```bash
cd /opt/<app>

# 1. Stop all app/worker services except database
docker compose -f docker-compose.yml stop app celery-worker celery-beat

# 2. Fetch the backup from S3 (replace TIMESTAMP with the desired dump)
aws s3 cp s3://standard-iowire-backups/daily/standard_iowire_YYYYMMDD-HHMMSS.dump \
  /tmp/restore.dump

# 3. Copy into the db container and restore with pg_restore -Fc
docker cp /tmp/restore.dump <db-container>:/tmp/restore.dump
docker exec -i <db-container> pg_restore -Fc \
  -U postgres -d standard_iowire \
  --no-owner --no-acl --clean --if-exists \
  /tmp/restore.dump

# 4. Restart services
docker compose -f docker-compose.yml up -d

# 5. Verify
docker compose -f docker-compose.yml ps
curl -f http://localhost:5000/health
```

#### Point-in-Time Recovery

WAL archiving is not configured — PITR to an arbitrary second is not available.
The recovery granularity is the daily S3 snapshot. To restore to a specific
historical point, pick the S3 object whose `LastModified` is closest to the
desired time and run the **Restore from Backup** procedure above.

#### Restore Drill (integrity test)

To exercise the restore pipeline without touching prod, run the scripted drill.
It downloads the latest S3 dump, restores it into an ephemeral Docker Postgres,
verifies canonical-table row counts, and tears the container down:

```bash
AWS_PROFILE=standard python3 scripts/backup/restore_drill.py --latest --verbose
```

Exit 0 = pass, 1 = restore/verification failed, 2 = setup/teardown failed.
Runs locally, zero infra cost. See also `scripts/check_backup_freshness.sh`
for the daily scheduled-agent freshness check (alerts if latest S3 object >25h).

#### Disaster Recovery

**Complete server failure:**

1. **Launch new EC2 instance** (from AWS console)
2. **Attach Elastic IP** (`<DB-EIP>` — verify with `dig +short standard.iowire.com`)
3. **Clone repository**
4. **Restore backup** from S3 or local copy
5. **Deploy services** using docker-compose
6. **Verify DNS** points to new IP

**Estimated Recovery Time**: 30-60 minutes

---

## Incident Response

### Severity Levels

| Level | Description | Response Time | Example |
|-------|-------------|---------------|---------|
| **P0 - Critical** | Complete outage | < 15 minutes | All sites down |
| **P1 - High** | Major feature broken | < 1 hour | Login broken |
| **P2 - Medium** | Minor feature broken | < 4 hours | Metrics not updating |
| **P3 - Low** | Cosmetic issue | < 24 hours | Typo in UI |

### P0 - Critical Incident Response

**All sites down or database unavailable**

**Immediate Actions** (first 5 minutes):
1. **Acknowledge incident** - Update status page
2. **Check basics**:
   ```bash
   # Can you SSH?
   ssh -i ~/code/tb_code/cert/standard-poc-key.pem ubuntu@standard.iowire.com

   # Are services running?
   cd /opt/<app>
   docker compose -f docker-compose.yml ps

   # Is disk full?
   df -h
   ```
3. **Quick fixes** (in order of preference):
   ```bash
   cd /opt/<app>

   # If a recent deploy is the suspected cause — roll back to the previous
   # image (canonical recovery path, faster than restarting in place):
   bash scripts/promote.sh rollback prod

   # Otherwise, restart all services
   docker compose -f docker-compose.yml restart

   # If that doesn't work
   docker compose -f docker-compose.yml down && docker compose -f docker-compose.yml up -d
   ```

**Investigation** (minutes 5-15):
1. **Check logs**:
   ```bash
   cd /opt/<app>

   # Look for errors in last 30 minutes
   docker compose -f docker-compose.yml logs --since 30m 2>&1 | grep -i "error\|fatal\|exception"

   # Sentry first — application errors land there with stack traces
   (source .env.ssm && python3 scripts/sentry_issues.py)

   # Check system logs
   sudo journalctl -xe --since "30 minutes ago"
   ```

2. **Check AWS**:
   - Instance status (AWS console)
   - Security group rules
   - EBS volume status

3. **Check external**:
   ```bash
   # DNS resolution
   dig standard.iowire.com
   
   # Network connectivity
   ping 8.8.8.8
   ```

**Resolution**:
- Document findings
- Apply fix
- Verify all sites healthy
- Update stakeholders
- Write postmortem

### P1 - High Severity Response

**Major feature broken (e.g., login, payment processing)**

**Response** (within 1 hour):
1. **Isolate issue**:
   - Which service/feature?
   - Which users affected?
   - Since when?

2. **Workaround**:
   - Can users access via alternate method?
   - Can we disable feature temporarily?

3. **Fix**:
   - Deploy hotfix if available
   - Or rollback to previous version

4. **Communicate**:
   - Notify affected users
   - Update status page

### Dispute Evidence Response

When a Stripe dispute is opened (notification via Sentry or email):

1. **Timeline awareness:** Stripe gives ~7 days to respond. Some dispute reasons (e.g. `fraudulent` without card-present data) have shorter windows. Check dispute detail in `https://dashboard.stripe.com/disputes/<id>` for the exact deadline.
2. **Triage via admin view:** Log into `/admin/disputes` for an overview. The current S3 view is read-only — evidence upload is not yet in-app.
3. **Submit evidence in Stripe Dashboard:** https://dashboard.stripe.com/disputes → select dispute → "Submit Evidence". Include:
   - Customer communication logs (export from our admin user detail view)
   - Service/product documentation (platform terms, receipt screenshot)
   - Shipping / delivery proof (for submissions: file upload timestamps, review session logs)
   - Customer signature / agreement if available
4. **Log the response in vault:** Create a `Comms/_dispute-log.md` entry with dispute ID, reason code, evidence submitted, outcome when resolved.

**In-app evidence upload is planned for Sprint 108** (deferred from S3 in Sprint 107 W2). <!-- TODO(verify): confirm the Sprint 108 backlog ticket / Architecture doc that tracks the in-app evidence upload — the cohost-split-per-plan link previously here was unrelated. --> Until then, Stripe Dashboard is the authoritative path.

---

## Maintenance Procedures

### Cross-references

- **Service Level Objectives** — see [`docs/operations/SLO.md`](./SLO.md) for tier definitions, availability + latency targets, and error budget policy. Ratified 2026-05-06.
- **Stripe live-mode cutover** — see [`docs/operations/stripe-cutover.md`](./stripe-cutover.md) for atomic-cutover invocation, staging rehearsal, rollback, and webhook signing-key rotation.
- **Rate-limit secret rotation** — `scripts/rotate_rate_limit_secret.sh --env both` (rotates SSM staging+prod + GH Actions secret atomically; documented evidence cadence at `<vault>/Compliance/Evidence/rotation-log.md`).
- **DB backup + restore drill** — `scripts/backup_db.sh` (cron) + `.github/workflows/restore-drill.yml` (quarterly; SOC2 CC7.5+A1.2 evidence).

### Planned Downtime

**When needed**:
- Major version upgrades
- Infrastructure changes
- Database migrations

**Procedure**:

1. **Schedule** (at least 48 hours notice):
   - Choose low-traffic time (typically 2-4 AM EST)
   - Notify users via email/banner

2. **Prepare**:
   ```bash
   # Full backup (canonical pg_dump -Fc to S3)
   sudo -u ubuntu /opt/<app>/scripts/backup_db.sh

   # Test changes on staging (celtics)
   # Prepare rollback plan — confirm `:previous` tag exists in local docker images
   ```

3. **Execute**:
   ```bash
   cd /opt/<app>

   # Stop the app + workers (keep db/redis/caddy up for static maintenance page)
   docker compose -f docker-compose.yml stop app celery-worker celery-beat

   # Perform maintenance
   # ...

   # Bring services back up
   docker compose -f docker-compose.yml up -d

   # Verify health
   curl -f http://localhost:5000/health
   curl -f https://standard.iowire.com/health
   ```

4. **Communicate**:
   - Send "all clear" message
   - Monitor for issues

### Updating Applications

Production is sign-off gated and ships ONLY an official release (a snapshot of a
`main` revision already validated on staging). Do NOT `git pull` +
`docker compose build` on the prod host: that bypasses sign-off, the version
stamp, and `:previous` rollback tagging, and `promote.sh prod` now REQUIRES a
release version. Full process: `docs/deployment/RELEASE_PROCESS.md`.

```bash
# 1. Merge to main (auto-validates on staging.iowire.com).

# 2. Cut a release from the staging-validated SHA (creates tag vYYYY.MM.DD):
./scripts/promote.sh cut-release          # or the "Cut Release" GitHub workflow

# 3. Deploy the release to prod (type the version to confirm):
./scripts/promote.sh prod v2026.06.13     # or the "Deploy to Production" workflow

# Rollback to the previous image (restores the :previous tag, health-gated):
./scripts/promote.sh rollback prod
```

**Emergency only** (audited, bypasses the release cut, requires explicit confirm):

```bash
scripts/break_glass_ship.sh --reason "<incident text>" --confirm
```

For full deploy/rollback procedures see `docs/deployment/ROLLBACK_RUNBOOK.md`.

### Scaling Up

**When to scale**:
- CPU > 70% sustained
- Memory > 80% sustained
- Response time > 500ms (95th percentile)

**Vertical scaling** (current approach):
```bash
# 1. Create snapshot (AWS console)
# 2. Stop instance
# 3. Change instance type (t3.medium → t3.large)
# 4. Start instance
# 5. Verify services start correctly
```

**Horizontal scaling** (future):
- Multiple EC2 instances
- Load balancer
- Shared database
- Redis cluster

### Gunicorn Worker Count

**Current value:** `-w 1 -k gevent --worker-connections 1000` (see
`docker-compose.yml:86` — runtime authoritative source; `Dockerfile.prod`
CMD is aligned but shadowed at runtime by the compose `command:` field).

**Why 1 worker, not N:**

1. **Socket.IO concurrency model.** The app uses `async_mode='gevent'`
   (`boom-queue/socketio_events.py:143`). A single gevent worker uses
   cooperative multitasking — it serves up to 1000 simultaneous connections
   via greenlets on one OS process. Worker count does not equal concurrency
   when `-k gevent` is active; `--worker-connections` does.
2. **Socket.IO horizontal scaling requires extra infra.** Running multiple
   gunicorn workers with Socket.IO needs (a) sticky sessions at the reverse
   proxy and (b) a shared message queue (Redis pub/sub) so events fan out
   across workers. We have Redis but no `SocketIO(message_queue=...)` wiring,
   and Caddy is not configured for sticky sessions. Raising `-w` without
   those two pieces will silently drop events between peers connected to
   different workers.
3. **Memory headroom on t3.medium.** Celtics is t3.medium (2 vCPU, 4GB RAM)
   running Caddy, PostgreSQL 15, PgBouncer, Redis 7, Celery worker+beat,
   LiveKit, plus the Flask app — all in one host-networked stack. Each
   additional gunicorn worker is another ~200-400MB resident set (SQLAlchemy
   session state, loaded models, jinja cache). The app service already has
   `mem_limit: 1g` and `cpus: 1.0` in compose.
4. **The classic `(2 × CPU) + 1` rule is for sync workers.** It does not
   apply to async worker classes. The gunicorn docs explicitly recommend
   1-2 async workers per core and letting `--worker-connections` carry the
   concurrency.

**When to reconsider:**

- **Switch away from gevent** (e.g., to `sync` or `uvicorn` for ASGI). Sync
  workers are 1-request-at-a-time, so you'd need `-w 5` minimum on a 2-vCPU
  box. This is a significant change — pair with migration plan review.
- **Upgrade to t3.large or larger** (Sprint 107 R4 candidate). With 8GB RAM
  and 2 vCPUs you could experiment with `-w 2 -k gevent` **only after**
  wiring `SocketIO(message_queue='redis://...')` and enabling sticky
  sessions in Caddy. Load-test first with `tests/load/` before committing.
- **Split the Socket.IO traffic out** (future horizontal scaling). A
  dedicated Socket.IO process fleet would let the HTTP-only fleet run
  multiple sync workers safely.

**Anti-patterns observed historically:**

- `docs/audits/2026-02-21-tech-stack-audit.md` and pre-Sprint-107 copies of
  `PROJECT_STATUS.md` and this runbook all claimed `-w 4`. That claim was
  never the deployed configuration — it was copied from the pre-Sprint-100
  Dockerfile.prod CMD default, which is overridden by compose. Sprint 107
  R5 reconciled all three sources to the actual runtime value.

---

## Emergency Contacts

### Internal Team

| Role | Name | Email | Phone |
|------|------|-------|-------|
| Primary On-Call | Alex Lannon | plzthx@users.noreply.github.com | on-file |
| Secondary On-Call | Tiffy Bomb | tiffy@standard.live | on-file |
| Database Admin | Alex Lannon | plzthx@users.noreply.github.com | on-file |
| DevOps Lead | Alex Lannon | plzthx@users.noreply.github.com | on-file |

Phone numbers intentionally not published in this repo — coordinate via the
Slack/Comms channels below. For major incidents requiring immediate voice
contact, phone numbers are held on-file with Primary and Secondary.

### External Services

| Service | Contact | Purpose |
|---------|---------|---------|
| AWS Support | support.aws.com | Infrastructure issues |
| Caddy Community | forum.caddyserver.com | SSL/proxy issues |
| PostgreSQL | postgresql.org/support | Database issues |

### Escalation Path

1. **Primary on-call** - First responder (< 15 min)
2. **Secondary on-call** - If no response (< 30 min)
3. **DevOps Lead** - Major incidents (< 1 hour)
4. **Management** - Customer-impacting outages

---

## Quick Reference

### Essential Commands

```bash
# SSH to server
ssh -i ~/code/tb_code/cert/standard-poc-key.pem ubuntu@standard.iowire.com

# All compose commands assume cwd == /opt/<app>
cd /opt/<app>

# Check all services
docker compose -f docker-compose.yml ps

# Restart all services
docker compose -f docker-compose.yml restart

# View logs
docker compose -f docker-compose.yml logs -f --tail=100

# Database backup (canonical pg_dump -Fc to S3)
sudo -u ubuntu /opt/<app>/scripts/backup_db.sh

# Health check (external + internal)
curl -f https://standard.iowire.com/health
curl -f http://localhost:5000/health

# Sentry triage (preferred over log greps for app errors)
(source /opt/<app>/.env.ssm && python3 /opt/<app>/scripts/sentry_issues.py)

# Database console
docker exec -it <db-container> psql -U postgres -d standard_iowire

# Deploy / rollback
bash /opt/<app>/scripts/promote.sh prod
bash /opt/<app>/scripts/promote.sh rollback prod
```

### Important File Locations

All prod paths are rooted at `/opt/<app>/` (owned by `ubuntu`).

| File | Location | Purpose |
|------|----------|---------|
| Docker Compose | `/opt/<app>/docker-compose.yml` | Service orchestration (canonical compose file) |
| Caddy Config | `/opt/<app>/Caddyfile` | Reverse proxy + TLS + CSP |
| Compose-interp env | `/opt/<app>/.env` | Non-secret compose interpolation vars only (11 keys). Regenerated by `scripts/load_ssm_secrets.py`. |
| Runtime env | `/opt/<app>/.env.ssm` | 79 runtime params loaded from AWS SSM `/<env>/*`. Never hand-edited. |
| Backups | `/opt/<app>/backups/` (local) + `s3://standard-iowire-backups/daily/` | Database dumps (pg_dump `-Fc`) |
| Deploy script | `/opt/<app>/scripts/promote.sh` | Unified deploy + health-gated auto-rollback (`promote.sh prod`, `promote.sh staging`, `promote.sh rollback <env>`) |
| Logs | `docker compose -f docker-compose.yml logs` | Application logs (note: `docker compose`, not `docker-compose`) |

---

**Version**: 2.2
**Last Updated**: 2026-05-04
**Next Review**: 2026-06-04

---

## Tax Handling

**Last Updated**: 2026-04-22 | **Decision doc**: `<vault>/Decisions/2026-04-22-us-only-tax-launch.md`

### Launch scope

US-only. Six states have active marketplace facilitator laws requiring the platform to collect and remit sales tax on behalf of hosts: **CA, WA, CO, TX, PA, MN**. Economic nexus thresholds apply (typically $100K revenue or 200 transactions per state in the prior 12 months). Stripe Tax monitors thresholds and activates collection automatically when crossed.

International customers (non-US) are blocked at signup — see Non-US customers below.

### What Stripe handles

Stripe Tax is enabled on subscription products. At US checkout, Stripe auto-calculates state sales tax based on the customer's billing address and adds a tax line to the generated receipt/invoice. No manual tax calculation is required in application code.

Verify configuration: Stripe Dashboard > Tax > Overview. Confirm tax is enabled and product tax codes are assigned (software/SaaS category).

### What we track

Stripe tax collection reporting lives in: **Stripe Dashboard > Tax > Reports**

- **Monthly**: Review the tax liability summary. Flag any state with unexpected $0 collections against known active customers in that state.
- **Annually**: Export full-year report for accountant. Required for state filing (Stripe does not file on our behalf — it calculates and collects only).

### Economic nexus monitor

Automated nexus threshold monitoring is **deferred to Sprint 108** (backlog Dec.4). Until that tooling ships:

- **Quarterly manual check**: Review Stripe Dashboard > Tax > Registrations. Confirm no new thresholds have been crossed in states not yet registered.
- Alert threshold: if a single state shows >$50K collected in a quarter, escalate to accountant before the quarter closes.

### Non-US customers

Non-US signups are blocked at the platform boundary. Confirm the active mechanism before closing this section:

- **Option A (IP geolocation)**: Signup route checks IP country code, rejects non-US at route level.
- **Option B (billing country)**: Stripe Checkout billing-country dropdown is restricted to US addresses only.

If neither mechanism is confirmed live, open a P1 issue — accepting international payments without VAT registration is a compliance violation.

### Escalation

Tax questions beyond standard Stripe Tax configuration:
- First: review `<vault>/Decisions/2026-04-22-us-only-tax-launch.md` for scope boundaries
- Escalate to: **Anrok** or **TaxJar** (no contract yet — obtain quotes before Sprint 109 if nexus complexity grows)
- Never attempt manual multi-state tax calculation in application code

### Phase 2 Scope (deferred, Sprint 108+)

Phase 1 (live today) is intentionally US-only to sidestep VAT/GST complexity while we validate
launch economics. Phase 2 is the scope required to accept international billing and automate
nexus management. Do not scope-creep Phase 1 code toward Phase 2 — tax logic is a compliance
surface, not a feature branch.

**In scope for Phase 2:**

1. **International billing (VAT/GST collection).** Relax the US-only gates in
   `boom-queue/routes/subscriptions.py:91` (subscription create) and
   `boom-queue/routes/payments/intents.py:426` (PI create) — both carry
   `TODO(Sprint 108 Phase 2)` markers referencing this section. Required work:
   - Provision an international tax engine (Stripe Tax covers 50+ jurisdictions but Stripe is
     not the system of record for filing — evaluate Anrok vs. TaxJar for registration +
     filing coverage).
   - Register for VAT in the EU (single One-Stop-Shop registration acceptable for B2C SaaS),
     UK VAT, Australian GST, Canadian GST/HST, etc. Threshold-based registrations deferred
     until Stripe Tax reports show ≥$10k in a jurisdiction.
   - Update the Stripe Customer creation path to pass `tax: {valid_location: true}` and accept
     per-customer VAT IDs (B2B reverse-charge flow). Test coverage required on
     `test_subscription_tax_*` and `test_payment_intent_international_*`.
   - Update the pricing-page disclaimer and marketing copy — current strings say "US-only at
     this time" in `templates/pricing.html` and error copy at `subscriptions.py:98`.

2. **Economic-nexus automation** (backlog item **Dec.4**, tracked in
   `tasks/todo.md`). Today's procedure is a **quarterly manual check** against
   Stripe Dashboard > Tax > Reports. Phase 2 automates it:
   - New Celery beat task: `boom-queue/tasks/economic_nexus_monitor.py` runs daily,
     `SELECT SUM(Transaction.amount_cents), COUNT(*) FROM transactions WHERE
     created_at >= NOW() - INTERVAL '365 days' GROUP BY billing_state`.
   - Alert threshold: **50% of the lowest state threshold** ($100k revenue OR 200
     transactions in a rolling 365-day window) → Slack alert + email to Alex.
   - Full cross-threshold → P1 incident, register-before-next-quarter-close.
   - Data source: `Transaction` rows (not Stripe Reports) so we can alert before
     Stripe's monthly report lag.

3. **Non-US gate relaxation contract.** Before removing US-only checks:
   - Economic-nexus monitor must be live and green for ≥30 days.
   - International tax engine contract signed and webhook handlers tested against
     test-mode international customers.
   - Legal sign-off on updated privacy/terms for non-US residents (GDPR, UK-GDPR,
     CCPA comparable already covered; Quebec Law 25 requires explicit consent language).
   - Regression test pass on the 10 state-nexus tests in
     `tests/boom_queue/test_subscription_tax.py` (create if absent).

4. **Migration of legacy US customers.** Customers created under Phase 1 have
   `Stripe Customer.address.country = 'US'` stamped with `customer_update={'address': 'never'}`
   (see `subscriptions.py:116-123`). Phase 2 must either:
   - Unlock `address` editing for existing customers via `stripe.Customer.modify` batch job, OR
   - Accept that Phase 1 customers remain locked US and only new signups get international —
     cleaner blast radius, preferred default.

**Explicitly out of scope for Phase 2:**
- Multi-currency pricing (separate initiative — requires `Plan.stripe_price_id` fan-out per
  currency). Phase 2 ships USD-only international.
- Tax-inclusive vs. tax-exclusive pricing toggle (tax always added on top in Phase 2 — matches
  current Stripe Tax default).
- Marketplace-facilitator remittance for US states that don't accept Stripe Tax as the
  filer-of-record (CA, WA, CO, TX, PA, MN). Already flagged in
  `Decisions/2026-04-22-us-only-tax-launch.md` as a launch-day residual risk.

**Triggers that force Phase 2 sooner:**
- Any US state economic-nexus threshold hit (automatic, alerted by Dec.4 monitor).
- Legal or accountant escalation flagging accrued silent liability.
- Commercial pressure: top-10 requested feature on `host_waitlist` is non-US signup.

---

*For deploy/rollback procedures, see: `docs/deployment/ROLLBACK_RUNBOOK.md` (uses unified `scripts/promote.sh`).*
*For database migrations, see: Alembic docs + `boom-queue/migrations/`. Migration gate lives in `.github/workflows/alembic-drift.yml` (allowlist-backed).*
