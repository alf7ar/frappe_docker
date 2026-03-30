# Backup & Restore Procedures

Alfhar ERPNext multi-tenant platform — operational runbook.

## Backup Strategy

| Tier | Frequency | Retention | Location |
|------|-----------|-----------|----------|
| Frequent | Every 6 hours | 7 days | Local + Backblaze B2 |
| Daily | 02:00 | 30 days | Local + Backblaze B2 |

Each backup includes:
1. **Frappe database** — `bench backup` produces a compressed SQL dump
2. **Site files** — private/public file attachments (included in `bench backup --with-files`)

---

## Running a Manual Backup

```bash
# Back up all sites in the running bench
docker exec frappe_docker-backend-1 \
  bench --site all backup --with-files

# Back up a specific site
docker exec frappe_docker-backend-1 \
  bench --site frontend backup --with-files

# Find the backup files
docker exec frappe_docker-backend-1 \
  find /home/frappe/frappe-bench/sites -name "*.sql.gz" -newer /tmp -ls
```

---

## Automated Backups (compose.backup-cron.yaml)

The `compose.backup-cron.yaml` overlay adds Ofelia cron scheduling on top of the existing `scheduler` service.

### Start backup cron

```bash
docker compose \
  -f frappe_docker/pwd.yml \
  -f frappe_docker/overrides/compose.backup-cron.yaml \
  up -d

# Verify the cron labels are applied
docker inspect frappe_docker-scheduler-1 \
  --format '{{json .Config.Labels}}' | python3 -m json.tool | grep ofelia
```

### View backup logs

```bash
# Ofelia job execution log
docker logs frappe_docker-cron-1 --tail 100 -f

# Frappe bench backup log
docker logs frappe_docker-scheduler-1 --tail 50
```

---

## Off-site Backup with Backblaze B2 (H-2)

Add the following to your `.env` file before starting the backup cron:

```env
B2_APPLICATION_KEY_ID=<your-key-id>
B2_APPLICATION_KEY=<your-application-key>
B2_BUCKET_NAME=erphost-backups
BACKUP_CRONSTRING=@every 6h
```

Install and configure `rclone` inside the scheduler container for B2 uploads:

```bash
# One-time rclone setup (run inside the container, or bake into custom image)
docker exec frappe_docker-scheduler-1 bash -c "
  curl -sf https://rclone.org/install.sh | bash
  rclone config create b2 b2 \
    account \"\$B2_APPLICATION_KEY_ID\" \
    key \"\$B2_APPLICATION_KEY\"
"

# Test upload
docker exec frappe_docker-scheduler-1 bash -c "
  LATEST=\$(find /home/frappe/frappe-bench/sites -name '*.sql.gz' | sort | tail -1)
  rclone copy \"\$LATEST\" b2:\$B2_BUCKET_NAME/frontend/
  echo 'Uploaded: '\$LATEST
"
```

---

## Restore Procedure

### 1. Identify the backup

```bash
# List bench backups (local)
docker exec frappe_docker-backend-1 \
  find /home/frappe/frappe-bench/sites/frontend/private/backups \
  -name "*.sql.gz" | sort

# List remote backups (B2)
docker exec frappe_docker-backend-1 \
  rclone ls b2:erphost-backups/frontend/ 2>/dev/null || \
  echo "rclone not configured in this container"
```

Backup file naming: `<YYYYMMDD>_<HHMMSS>-<site>-database.sql.gz`

### 2. Copy backup into container (if restoring from host or B2)

```bash
# From host filesystem
docker cp /path/to/backup.sql.gz \
  frappe_docker-backend-1:/home/frappe/frappe-bench/sites/frontend/private/backups/

# From B2 (run inside container)
docker exec frappe_docker-backend-1 bash -c "
  rclone copy b2:erphost-backups/frontend/<backup-file>.sql.gz \
    /home/frappe/frappe-bench/sites/frontend/private/backups/
"
```

### 3. Stop workers to prevent write conflicts

```bash
docker compose -f frappe_docker/pwd.yml \
  stop scheduler queue-short queue-long websocket
```

### 4. Restore the database

```bash
docker exec frappe_docker-backend-1 \
  bench --site frontend restore \
  /home/frappe/frappe-bench/sites/frontend/private/backups/<backup-file>.sql.gz
```

For a full restore with files:

```bash
docker exec frappe_docker-backend-1 \
  bench --site frontend restore \
  /home/frappe/frappe-bench/sites/frontend/private/backups/<backup-file>.sql.gz \
  --with-public-files /path/to/public-files.tar \
  --with-private-files /path/to/private-files.tar
```

### 5. Run post-restore migration

```bash
docker exec frappe_docker-backend-1 \
  bench --site frontend migrate

docker exec frappe_docker-backend-1 \
  bench --site frontend clear-cache
```

### 6. Restart all services

```bash
docker compose -f frappe_docker/pwd.yml \
  start scheduler queue-short queue-long websocket
```

### 7. Verify

```bash
# API health check
curl -sf http://localhost:8080/api/method/frappe.auth.get_logged_user
# Expected: HTTP 403 (server running, unauthenticated)

# Verify alfhar_erp is still importable
docker exec frappe_docker-backend-1 \
  python3 -c "import alfhar_erp; print('alfhar_erp', alfhar_erp.__version__)"
```

---

## Disaster Recovery (Full Server Loss)

1. **Provision a new server** with Docker Engine installed
2. **Clone the repository** and copy `.env` from secure storage
3. **Start the stack**:
   ```bash
   docker compose -f frappe_docker/pwd.yml up -d
   ```
4. **Restore each site** using steps 2–7 above
5. **Verify DNS** — update A records if server IP changed
6. **Re-issue SSL certificates** if using Traefik:
   ```bash
   docker compose \
     -f frappe_docker/pwd.yml \
     -f frappe_docker/overrides/compose.traefik-ssl.yaml \
     restart traefik
   ```

Expected RTO: ~1 hour for a single site.

---

## See Also

- `overrides/compose.backup-cron.yaml` — automated Ofelia cron container
- `overrides/compose.resource-limits.yaml` — memory/CPU limits (ALF-17 C-2)
- `overrides/compose.logging.yaml` — log rotation (ALF-17 C-3)
- `overrides/compose.healthchecks.yaml` — container health checks (ALF-17 H-1)
