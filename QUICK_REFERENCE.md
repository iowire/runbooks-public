# Quick Reference - iowire.com

**Emergency Access**: `ssh -i ~/code/tb_code/cert/standard-poc-key.pem ubuntu@standard.iowire.com`
(that host is **PRODUCTION** — confirm with `dig +short standard.iowire.com` before
running anything. EC2 Name tags are misleading; see
`.claude-shared/rules/production-safety.md`.)

> **This page is prod-framed but is NOT fully current — parts of it predate the
> dedicated-RDS cutover.** Two standing corrections that apply to every block below:
>
> 1. **Prod's database is the RDS `<prod-db>`, not a container.** Any
>    `docker exec ... psql` against a compose `db` container on prod queries an
>    orphaned database prod does not read — including the read-only queries on
>    this page. Treat their output as untrustworthy. See **Backup & Restore**.
> 2. **Services named `standard`, `portal`, `metrics`, `admin` do not exist.**
>    The real services are `caddy, app, app-next, celery-worker, celery-beat,
>    pgbouncer, db, redis, tusd, livekit, discord-bot, livekit-ingress,
>    livekit-egress, alloy`. Commands naming the first four are **no-ops** —
>    they will not restart what you think they restarted.
>
> Known-stale blocks are tracked in `tasks/runbook-drift-2026-07-15.md` §4.
> For anything load-bearing, prefer `OPERATIONAL_RUNBOOK.md` (alert-driven,
> maintained) over this page.

---

## 🚨 Emergency Commands

```bash
# Check if everything is running
docker-compose ps

# Restart everything
docker-compose restart

# View recent errors
docker-compose logs --since 1h 2>&1 | grep -i error

# Check health
curl http://localhost:8080/health
```

---

## Health Checks

| Service | Command | Expected |
|---------|---------|----------|
| **App (internal)** | `docker exec <app-container> python -c "import urllib.request; print(urllib.request.urlopen('http://localhost:5000/health').status)"` | 200 |
| **App (external)** | `curl -s https://standard.iowire.com/health` | JSON |
| **Database** | `docker exec <db-container> psql -U postgres -d standard_iowire -c "SELECT 1;"` | 1 row |
| **Redis** | `docker exec <redis-container> redis-cli ping` | PONG |
| **PgBouncer** | `docker exec <pgbouncer-container> sh -c 'echo "SHOW POOLS;" \| psql -p 6432 -U postgres pgbouncer'` | pool list |
| **Prometheus** | `docker exec <app-container> python -c "import urllib.request; print(urllib.request.urlopen('http://localhost:5000/metrics').read().decode()[:100])"` | metrics text |
| **Uptime Monitor** | `tail -5 /var/log/uptime_monitor.log` | recent OK |
| **Metrics JSONL** | `tail -1 /var/log/standard-metrics.jsonl \| python3 -m json.tool` | JSON entry |

---

## 📊 Monitoring

```bash
# Resource usage
docker stats --no-stream

# Disk space
df -h

# Active users
docker exec -it <db-container> psql -U postgres -d standard_iowire -c \
  "SELECT COUNT(*) FROM user_sessions WHERE expires_at > NOW();"

# Failed logins (last hour)
docker-compose logs --since 1h | grep -i "authentication failed" | wc -l
```

---

## 🔧 Common Fixes

### Site Down (502 Error)
```bash
docker-compose restart <service_name>
# Wait 30 seconds
curl -I https://<subdomain>.iowire.com
```

### Database Connection Error
```bash
docker-compose restart db
# Wait for healthy status
docker-compose ps db
docker-compose restart standard portal metrics admin
```

### High CPU
```bash
# Find culprit
docker stats --no-stream | sort -k3 -rh

# Restart it
docker-compose restart <service_name>
```

### Out of Disk Space

> **NEVER run `docker system prune --volumes` (or `-a`) on prod.** `--volumes`
> destroys every volume not attached to a *running* container — including
> `caddy_data` (TLS certs), `postgres_data`, and `user_uploads` (**user audio**).
> Any stopped service (a restart, a `down`, a prune during triage) makes its
> volumes eligible. `-a` additionally deletes every unused image, forcing a
> full rebuild on a box that builds the app from source.

```bash
# Reclaim safely: stopped containers + dangling images only. Never --volumes, never -a.
docker container prune -f
docker image prune -f

# Rotate logs (usually the real consumer)
sudo logrotate -f /etc/logrotate.conf

# Clean old backups (canonical local path; S3 retention is bucket lifecycle)
find /opt/<app>/backups -type f -mtime +7 -delete
```

Full triage with per-consumer diagnosis: `OPERATIONAL_RUNBOOK.md` → Alert S3: Disk Space Low.

---

## 💾 Backup & Restore

> **Prod's database is the dedicated RDS `<prod-db>`**, not a container.
> The base compose file still defines a local `db` service, and the production
> override repoints pgbouncer at the RDS. So **any `docker exec ... psql` /
> `pg_dump` against a compose `db` container on prod talks to an orphaned
> database that prod does not read.** It will appear to succeed and change
> nothing. See `docker-compose.override.yml` (pgbouncer `DATABASE_URL`) and
> `scripts/deploy_box.sh` (`DATA_TIER_SERVICES=""` on prod — "do NOT (re)start
> the orphan local `db` container").

### Snapshot before anything risky (verified, safe)
```bash
# Point-in-time RDS snapshot. This is the prod-correct pre-change safety net.
AWS_PROFILE=standard aws rds create-db-snapshot \
  --db-instance-identifier <prod-db> \
  --db-snapshot-identifier pre-change-$(date +%Y%m%d-%H%M) --region us-east-1
```
Source: `docs/deployment/ROLLBACK_RUNBOOK.md` → Pre-Deploy Checklist.

### Restore from Backup

> ### ⛔ GAP — there is no verified prod restore procedure. Do not improvise one.
>
> The step-by-step restore that used to live here was **removed on 2026-07-15**:
> it stopped four services that do not exist, restored into a container that is
> not prod's database, and finished with a single-`-f` `up -d` that repoints
> prod's pgbouncer away from the RDS. Every one of those is silent — the
> operator sees success.
>
> **This repo does not contain a drilled RDS restore procedure**, and the
> provenance of the daily dumps is **unverified**: `scripts/backup_db.sh` dumps
> the compose `db` service (the orphan local container), not the RDS. Until that
> is resolved, do **not** assume `s3://standard-iowire-backups/daily/` holds
> prod data.
>
> **If you need to restore prod: STOP and escalate to Alex.** A prod restore is
> an RDS operation (snapshot restore / PITR), it is sign-off gated, and it must
> not be assembled from this page under incident pressure. Related:
> `docs/deployment/ROLLBACK_RUNBOOK.md` (rollback ≠ restore).
>
> Restoring into a **local or staging** container is a different, documented
> operation: `OPERATIONAL_RUNBOOK.md` → Recovery Procedures. Do not point it at prod.

---

## 🔄 Deployments

### Deploy Code Update
Production is sign-off gated. Never `git pull` + build on the prod host by hand.
Full process: `docs/deployment/RELEASE_PROCESS.md`.
```bash
./scripts/promote.sh cut-release          # cut vYYYY.MM.DD from staging-validated main
./scripts/promote.sh prod v2026.06.13     # deploy the release (type the version to confirm)
```

### Rollback
```bash
./scripts/promote.sh rollback prod        # restore the previous image (health-gated)
```

---

## 👥 User Management

> **Use the admin UI. Do not mutate users with `psql` on prod.**
>
> The hand-rolled `INSERT`/`UPDATE`/`DELETE` recipes that used to live here were
> **removed on 2026-07-15**. They bypassed the audited admin path (no
> `admin_action` row, no attribution — violates the standing "prod mutations use
> the audited path" rule), and every one of them was also broken:
> they targeted a `db` container that is not prod's database (see
> **Backup & Restore** above), referenced a `users.roles` column that no longer
> exists (ABAC bundles replaced it), called a `scripts/hash_password.py` that is
> not in the repo, and wrote to a `user_sessions` table that exists only under
> `.archive/`. A "successful" password reset here changes nothing on the account
> the user is locked out of.

| Task | Audited path |
|---|---|
| Find / inspect a user | `/admin/users` → `boom-queue/routes/admin/users.py:86` (`admin_users`), detail at `:280` |
| Change user fields | `/api/admin/users/<id>` `PUT` → `boom-queue/routes/admin/users.py:144` (`api_admin_update_user`) |
| Password reset | Self-serve flow — `boom-queue/routes/auth.py:1255` (`forgot_password`) issues the token; `:1324` (`reset_password`) consumes it |
| Unlock a locked-out user | `/admin/users/<id>/unlock` `POST` → `boom-queue/routes/admin/lockout.py:52` (`admin_unlock_user`) |
| Change a subscription | `/admin/users/<id>/subscription` `POST` → `boom-queue/routes/admin/users.py:357` |

**Force-logout has no audited admin route today** (the old recipe targeted a dead
table). If you need to invalidate a session on prod, escalate to Alex rather than
improvising SQL.

If break-glass SQL is genuinely unavoidable, it runs against the **RDS** (see
**Backup & Restore** above for why, and `OPERATIONAL_RUNBOOK.md` →
Database Access for the owner-role connection), with an explicit audit-log entry
and sign-off. It is never a `docker exec` against a compose `db` container.

---

## 📝 Logs

```bash
# Last 100 lines, all services
docker-compose logs --tail=100

# Follow logs live
docker-compose logs -f

# Specific service
docker-compose logs -f standard

# Last hour, errors only
docker-compose logs --since 1h 2>&1 | grep -i error

# Export logs for analysis
docker-compose logs --since 24h > logs-$(date +%Y%m%d).txt
```

---

## 🔐 Security

### Check Failed Logins
```bash
docker exec -it <db-container> psql -U postgres -d standard_iowire -c \
  "SELECT COUNT(*), event_data->>'email' as email 
   FROM audit_logs 
   WHERE success = false AND event_type LIKE '%login%' 
   AND created_at > NOW() - INTERVAL '1 hour'
   GROUP BY email 
   ORDER BY count DESC;"
```

### Block IP (via Caddy)
```bash
# Edit Caddyfile to add:
# @blocked remote_ip 1.2.3.4
# abort @blocked

nano ~/deployment/Caddyfile
docker-compose restart caddy
```

### Review Recent Admin Actions
```bash
docker exec -it <db-container> psql -U postgres -d standard_iowire -c \
  "SELECT created_at, event_type, event_data
   FROM audit_logs
   WHERE site = 'admin'
   ORDER BY created_at DESC
   LIMIT 10;"
```

### Rate Limit Whitelist

Production IPs exempt from rate limiting:
- `127.0.0.1` / `::1` - Localhost (for internal health checks)
- Production server IP — retired; see `.claude-shared/rules/production-safety.md` for the current value
- `172.31.15.80` - Dev server (celtics.iowire.com)

To add IPs: Set `RATE_LIMIT_WHITELIST_IPS` env var with comma-separated list.

**Security Note**: X-Forwarded-For headers are trusted because ProxyFix middleware ensures only the Caddy reverse proxy can set these headers. External clients cannot bypass rate limiting via header injection.

---

## 🌐 DNS & SSL

### Check DNS
```bash
dig standard.iowire.com

# Should return: the current prod IP — see .claude-shared/rules/production-safety.md (this doc's old value is retired)
```

### Check SSL Certificate
```bash
echo | openssl s_client -servername standard.iowire.com \
  -connect standard.iowire.com:443 2>/dev/null | openssl x509 -noout -dates

# Check all subdomains
for domain in iowire.com standard.iowire.com portal.iowire.com metrics.iowire.com admin.iowire.com; do
  echo "$domain:"
  echo | openssl s_client -servername $domain -connect $domain:443 2>/dev/null | \
    openssl x509 -noout -enddate
done
```

### Force SSL Renewal
```bash
docker-compose restart caddy
# Caddy will auto-renew if < 30 days until expiry
```

---

## 📞 When Things Go Wrong

1. **Don't panic** - Take a breath
2. **Check basics** - SSH, docker-compose ps, disk space
3. **Check logs** - Look for errors in last 30 minutes
4. **Try restart** - docker-compose restart
5. **Check monitoring** - ~/monitor.sh
6. **Review runbook** - OPERATIONAL_RUNBOOK.md
7. **Ask for help** - Escalate if needed

---

## 📚 Documentation

| Document | Purpose |
|----------|---------|
| **QUICK_REFERENCE.md** | This file - Quick commands |
| **OPERATIONAL_RUNBOOK.md** | Detailed procedures |
| **DEPLOYMENT_GUIDE.md** | Deployment steps |
| **DATABASE_MIGRATION_GUIDE.md** | PostgreSQL migration |
| **TROUBLESHOOTING.md** | Common issues |

---

## 🔗 Important URLs

| Service | URL |
|---------|-----|
| **Main Site** | https://iowire.com |
| **STANDARD** | https://standard.iowire.com |
| **Admin Panel** | https://standard.iowire.com/admin |
| **API Docs** | https://standard.iowire.com/api/docs |
| **Health Check** | https://standard.iowire.com/health |
| **Metrics** | https://standard.iowire.com/metrics (Prometheus) |

---

## 📊 Test Credentials

> ⚠️ **LOCAL-DEV-ONLY.** This table is stale and duplicates a source of truth that has since diverged. Canonical: `docs/TEST_USERS.md` — its LOCAL-DEV-ONLY banner explains why the published password is not valid on any deployed target. Use that file, not this section.

---

## 💡 Tips

- Use `tmux` or `screen` for persistent sessions
- Bookmark this file in your browser
- Keep SSH key handy
- Monitor disk space weekly
- Test backups monthly
- Document everything you do

---

**Last Updated**: February 6, 2026  
**Version**: 2.0
