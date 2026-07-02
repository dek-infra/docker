# ClickHouse Zero-Downtime Automated Backup Architecture

This repository contains a complete, production-ready setup for automating ClickHouse database backups to an S3-compatible storage (MinIO) using **n8n** as an orchestration engine.

## 🚀 Key Features
- **Strictly Zero Downtime**: Utilizes ClickHouse's native `BACKUP TO S3` functionality. No pod restarts, sidecars, or maintenance windows required.
- **Smart Incremental Backups**: Performs a Full Backup every Sunday and Incremental Backups (fetching only changed data) from Monday to Saturday, drastically reducing storage costs for large (1TB+) databases.
- **Automated MS Teams Alerts**: Integrated webhooks alert the team upon success, or immediately trigger upon failure with detailed error logs.
- **Easy Disaster Recovery**: Simple, one-line SQL commands to restore data from any incremental point-in-time.

---

## 🏗️ Architecture Stack
1. **ClickHouse**: The core analytical database.
2. **MinIO**: S3-compatible object storage acting as the backup destination.
3. **n8n**: The workflow automation tool managing the cron schedule, conditional logic, API requests, and MS Teams alerts.

---

## 🛠️ Setup Instructions

### 1. Start the Infrastructure
Run the provided Docker Compose file to spin up ClickHouse, MinIO, and n8n:
```bash
docker-compose up -d
```

### 2. Configure n8n Workflow
1. Access n8n at `http://localhost:5678`.
2. Import the `_workflows/n8n-smart-incremental-workflow.json` file.
3. **Credentials**: Create an HTTP Basic Auth credential in n8n for ClickHouse (`system_admin` / `qwer1234`).
4. **Alerts**: Double-click the MS Teams nodes and insert your MS Teams Webhook URL. Ensure the `Ignore SSL Issues` option is checked if necessary.
5. Activate the workflow!

### 3. Configure MinIO Lifecycle Policy
To prevent your storage from growing indefinitely, configure a retention policy:
1. Access MinIO UI at `http://localhost:9001` (`minioadmin` / `minioadmin`).
2. Go to **Buckets** -> `clickhouse-backups` -> **Lifecycle**.
3. Add a rule to expire/delete folders older than **14 Days** (or your desired retention period).

---

## 💾 Native Backup & Restore Commands

*(Note: These are handled automatically by n8n, but provided here for manual testing and disaster recovery).*

### Full Backup (e.g., Sundays)
```sql
BACKUP ALL EXCEPT DATABASE information_schema, INFORMATION_SCHEMA, system 
TO S3('http://minio:9000/clickhouse-backups/backup-YYYYMMDD/', 'system_admin', 'qwer1234') 
ASYNC;
```

### Incremental Backup (e.g., Mon - Sat)
```sql
BACKUP ALL EXCEPT DATABASE information_schema, INFORMATION_SCHEMA, system 
TO S3('http://minio:9000/clickhouse-backups/backup-YYYYMMDD/', 'system_admin', 'qwer1234') 
SETTINGS base_backup = S3('http://minio:9000/clickhouse-backups/backup-<PREVIOUS_DAY>/', 'system_admin', 'qwer1234') 
ASYNC;
```
> **Important:** We must exclude virtual databases like `information_schema` and `system` using `EXCEPT DATABASE`.

---

## 🚨 Disaster Recovery (How to Restore)

If a database is corrupted or accidentally dropped, you can restore it directly from the incremental backup folder. ClickHouse is smart enough to automatically fetch the base data linked to that incremental point!

1. **Clean Slate** (Drop the corrupted database if it still exists):
```sql
DROP DATABASE your_database;
```

2. **Restore** from the specific backup date:
```sql
RESTORE ALL 
EXCEPT DATABASE information_schema, INFORMATION_SCHEMA, system 
FROM S3('http://minio:9000/clickhouse-backups/backup-YYYYMMDD/', 'system_admin', 'qwer1234')
SETTINGS allow_non_empty_tables=true;
```
> **Tip:** Use `allow_non_empty_tables=true` if you are restoring into an existing database structure that already has some data to prevent `CANNOT_RESTORE_TABLE` errors.