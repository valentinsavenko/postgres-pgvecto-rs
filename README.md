# postgres-pgvecto-rs

Basic postgres chart that I build for usage with the [Immich Chart](https://github.com/immich-app/immich-charts/blob/main/README.md).
Which requires pgvecto-rs/pgvector.
Most charts fail to use the official docker image form tensorchord out of the box.

The data AND backups are all stored in a single PV for now, so make sure it's a backuped folder!

```bash
# install
CHART=https://github.com/valentinsavenko/postgres-pgvecto-rs/raw/refs/heads/main/postgres-pgvecto-rs-0.2.0.tgz
RELEASE_NAME=psql-ps


helm install ${RELEASE_NAME} $ ${CHART} -f values.yaml
```

# database backups

The chart creates a cronjob that runs weekly and creates 10 weeks of backups.
Backups use `pg_basebackup` for file-level consistency with verification before rotation.

```bash
# trigger backup NOW
kubectl create job --from=cronjob/${RELEASE_NAME}-postgres-backup postgres-backup-test

# render restore job template with your conf
helm template ${RELEASE_NAME}  ${CHART} --set restore.enabled=true -f values.yaml --show-only templates/job.yaml >> db-job-restore-backup.yaml

# restore most recent bkp
kubectl apply -f db-job-restore-backup.yaml
```

## Backup format

Each backup is a full, self-contained file-level backup:

```
/backups/<release>_backup_2026-05-23_02-00-00/
├── base.tar.gz      # Full database files (compressed)
├── pg_wal.tar.gz    # WAL generated during backup
└── backup_label     # Backup metadata
```

## Backup process

1. `pg_start_backup` - forces a checkpoint to ensure data consistency
2. `pg_basebackup` - creates a compressed tar backup of the data directory
3. `pg_stop_backup` - marks the end of the backup
4. Verification - checks that backup files exist, are non-empty, and tar integrity passes
5. Rotation - old backups are only deleted after successful verification

## Restore process

The restore job extracts the most recent backup, verifies it, and replaces the PostgreSQL data directory.

```bash
# 1. Render and apply the restore job
helm template ${RELEASE_NAME} ${CHART} --set restore.enabled=true -f values.yaml --show-only templates/job.yaml | kubectl apply -f -

# 2 Wait for the restore job to complete
kubectl wait --for=condition=complete job/${RELEASE_NAME}-postgres-restore --timeout=300s

```

## Configuration

```yaml
backup:
  schedule: "0 2 * * 0"  # Cron schedule (default: weekly Sunday 2AM)
  maxBackups: 10          # Number of backups to keep
  verifyEnabled: true     # Verify backup integrity before rotation
```
