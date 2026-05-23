# Plan: Improve Helm Chart Backup with pg_basebackup

## Context

The current backup implementation uses `pg_dump` (logical SQL dump) which:
- Is inconsistent when writes happen during backup
- Provides no point-in-time recovery
- Is slow and produces large backups

User reports backup is broken when used on a running instance.

## Requirements

1. Use `pg_basebackup` for file-level backups (physical backup)
2. Each backup is **full and self-restorable** (no incremental/WAL replay required for basic restore)
3. **Verify backups** before rotating old ones
4. **Rotate old backups only after positive verification**

---

## Todos

- [ ] **Update `templates/cronjob.yaml`**:
  - Replace `pg_dump` with `pg_basebackup`
  - Add `pg_start_backup` / `pg_stop_backup` wrapping
  - Add backup verification step (extract tar and run `pg_verifybackup` or check integrity)
  - Only rotate old backups after verification passes

- [ ] **Update `templates/job.yaml` (restore job)**:
  - Update to restore from `pg_basebackup` format (tar extraction)
  - Ensure restore works with the new backup format

- [ ] **Update `values.yaml`**:
  - Add `backup.verifyEnabled: true` option (default true)
  - Document new backup format in comments

- [ ] **Update `README.md`**:
  - Document new backup mechanism
  - Update restore instructions for new format

---

## Technical Design

### Backup CronJob Flow

```
1. pg_start_backup('backup', true)
   └── Forces checkpoint, marks backup start in WAL

2. pg_basebackup -Ft -z -P
   └── Creates tar-formatted, compressed base backup
   └── -F t  : output tar format (not plain)
   └── -z    : gzip compression
   └── -P    : show progress

3. pg_stop_backup()
   └── Marks backup end, switches WAL segment

4. Verify backup integrity
   └── Extract to temp location
   └── pg_verifybackup (PostgreSQL 13+) OR
   └── Check tar integrity with tar -tzf

5. Rotate old backups (only if verification passed)
   └── ls -drt | head -n -maxBackups | xargs rm -rf
```

### Backup Directory Structure

```
/backups/
├── <release>_backup_2026-05-23_02-00-00/
│   ├── base.tar.gz          # Full database files
│   ├── pg_wal.tar.gz       # WAL needed for PITR (optional for full restore)
│   └── backup_label         # Backup metadata
└── <release>_backup_2026-05-16_02-00-00/
    └── ...
```

### Restore Job Flow

```
1. Find most recent backup directory
2. Extract base.tar.gz to PGDATA location
3. Copy backup_label to PGDATA
4. Set up recovery.conf (or recovery signal in PG13+)
5. Start PostgreSQL - it will auto-recover
```

---

## Assumptions

1. **PostgreSQL version**: pgvecto-rs pg16-v0.2.1 supports `pg_basebackup` and `pg_verifybackup`
2. **Storage**: Same PVC is used (not ideal but acceptable for this scope)
3. **No incremental**: Each backup is a full base backup, self-contained
4. **Verification**: `pg_verifybackup` is available in the Docker image (PostgreSQL 13+). If not, fallback to `tar -tzf` integrity check
5. **Restore is manual**: User triggers restore job manually (existing behavior)

---

## Success Criteria

1. Backup cronjob completes without errors on a running PostgreSQL instance
2. Each backup is a complete, standalone restorable copy
3. Old backups are only deleted AFTER verification of new backup succeeds
4. Restore job successfully restores from the new backup format
5. Backup verification catches corrupted/incomplete backups before rotation

---

## Files to Modify

| File | Changes |
|------|---------|
| `templates/cronjob.yaml` | Complete rewrite of backup command |
| `templates/job.yaml` | Update restore to use tar extraction |
| `values.yaml` | Add `backup.verifyEnabled` option |
| `README.md` | Document new backup/restore process |