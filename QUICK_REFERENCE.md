# Quick Reference - iowire.com

**Emergency Access**: `ssh -i ~/code/tb_code/cert/standard-poc-key.pem ubuntu@standard.iowire.com`

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
| **PgBouncer** | `docker exec standard-iowire-pgbouncer-1 sh -c 'echo "SHOW POOLS;" \| psql -p 6432 -U postgres pgbouncer'` | pool list |
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
```bash
# Clean Docker
docker system prune -a --volumes

# Clean old backups
find ~/postgres_backups -mtime +7 -delete
```

---

## 💾 Backup & Restore

### Backup Now
```bash
~/backup_postgres.sh
```

### Restore from Backup
```bash
# ⚠️ This overwrites current data!
docker-compose stop standard portal metrics admin
gunzip -c ~/postgres_backups/backup-YYYYMMDD-HHMMSS.sql.gz | \
  docker exec -i <db-container> psql -U postgres standard_iowire
docker-compose up -d
```

---

## 🔄 Deployments

### Deploy Code Update
```bash
cd ~/code
git pull origin main
docker-compose build
docker-compose up -d --no-deps <service_name>
# Verify
curl http://localhost:<port>/health
```

### Rollback
```bash
cd ~/code
git revert HEAD
docker-compose build
docker-compose up -d
```

---

## 👥 User Management

### Create User (via psql)
```bash
docker exec -it <db-container> psql -U postgres -d standard_iowire << 'EOF'
INSERT INTO users (email, password_hash, roles, created_at, updated_at)
VALUES (
  'user@example.com',
  '$argon2id$...',  -- Use scripts/hash_password.py
  '["user"]'::jsonb,
  NOW(),
  NOW()
);
EOF
```

### Reset Password
```bash
# Generate hash
cd ~/code
python3 scripts/hash_password.py "new_password"

# Update in database
docker exec -it <db-container> psql -U postgres -d standard_iowire -c \
  "UPDATE users SET password_hash = '<hash>' WHERE email = 'user@example.com';"
```

### Delete User Sessions (Force Logout)
```bash
docker exec -it <db-container> psql -U postgres -d standard_iowire -c \
  "DELETE FROM user_sessions WHERE user_id = (SELECT id FROM users WHERE email = 'user@example.com');"
```

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
- `98.87.212.123` - Production server (standard.iowire.com)
- `172.31.15.80` - Dev server (celtics.iowire.com)

To add IPs: Set `RATE_LIMIT_WHITELIST_IPS` env var with comma-separated list.

**Security Note**: X-Forwarded-For headers are trusted because ProxyFix middleware ensures only the Caddy reverse proxy can set these headers. External clients cannot bypass rate limiting via header injection.

---

## 🌐 DNS & SSL

### Check DNS
```bash
dig standard.iowire.com

# Should return: 98.87.212.123
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

| Role | Email | Password |
|------|-------|----------|
| Admin | admin@iowire.com | TestPass123! |
| Team | marcus@iowire.com | TestPass123! |
| Client | luna@client.com | TestPass123! |
| Reviewer | tiffy@standard.live | TestPass123! |
| Artist | troy@test.com | TestPass123! |

**⚠️ Change these in production!**

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
