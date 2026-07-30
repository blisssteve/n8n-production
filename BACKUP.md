# n8n Backup and Restore Strategy

This stack uses layered Dokploy backups. PostgreSQL is authoritative for workflows,
credentials, users, projects, and execution metadata. Named-volume archives preserve
n8n local state, filesystem binary data, and custom/community nodes.

## Objectives

- RPO: 24 hours for the standard schedule; take an additional logical dump immediately
  before every n8n upgrade or schema-affecting change.
- RTO: 4 hours for a normal database-and-volume restore, subject to destination and
  image availability.
- Destination: Dokploy destination `Google Drive via rclone S3`, bucket
  `dokploy-backups`.
- Retention: latest 14 restore points per backup record.
- Schedule timezone: Dokploy cron runs in UTC; the schedules below correspond to
  02:25-03:00 Australia/Brisbane.

## Active Dokploy backups

| Layer | Method | UTC schedule | Retention | Consistency |
|---|---|---:|---:|---|
| PostgreSQL `n8n` | Logical Compose database backup (`pg_dump`, gzip) | `25 16 * * *` | 14 | Authoritative online database backup |
| `n8n_data` | Dokploy named-volume archive | `40 16 * * *` | 14 | Online supplementary snapshot |
| `n8n_binary_data` | Dokploy named-volume archive | `50 16 * * *` | 14 | Online supplementary snapshot |
| `n8n_custom_nodes` | Dokploy named-volume archive | `0 17 * * *` | 14 | Online supplementary snapshot |

The three active volume backups use `turnOff=false`. This avoids Dokploy's unsafe
failure mode where a failed stop-based archive can leave the application stopped.
Because the files are archived online and separately, they are not an atomic
consistency group with PostgreSQL. Select the closest same-night set during restore.

The physical `n8n_postgres_data` volume-backup record remains disabled. A live tar of
PostgreSQL's data directory is not a supported database-consistent backup, while a
stop-based native Dokploy backup lacks an independent restart watchdog. The logical
database backup is the supported recovery source.

## Pre-upgrade gate

Before changing either the n8n or task-runners image:

1. Resolve the latest non-draft, non-prerelease n8n release.
2. Verify registry manifests exist for both `docker.n8n.io/n8nio/n8n:<version>` and
   `n8nio/runners:<version>`.
3. Trigger the logical PostgreSQL backup manually.
4. Confirm a new, non-zero remote object exists under
   `n8n-production-xw4mkx_postgres/n8n/logical-postgres/`.
5. Keep the old n8n and task-runners tags as the image rollback point.

Do not start an older n8n image against a database already migrated by a newer image.
Rollback after a failed migration requires restoring the matching pre-upgrade database
backup first, then redeploying both old image tags.

## Restore order

1. Stop n8n and task-runners through Dokploy; keep PostgreSQL available for the
   supported restore operation.
2. Restore the selected logical PostgreSQL dump through Dokploy.
3. Restore the matching `n8n_data`, `n8n_binary_data`, and `n8n_custom_nodes` archives
   into disposable volumes first when performing a drill, or into the production
   volumes only during an approved incident restore.
4. Deploy the n8n and task-runners image versions recorded for that restore point.
5. Verify `/healthz`, login, workflow and credential counts, representative binary
   data, installed community nodes, webhooks, and task-runner connectivity.

Never overwrite production volumes merely to test a backup.

## Verification and drills

Evidence levels must be reported separately:

1. Configured: enabled record, schedule, destination, and retention are correct.
2. Executed: a new non-zero remote object exists.
3. Validated: an independent full readback passes SHA-256 plus gzip/tar or SQL-dump
   integrity checks.
4. Restore-proven: a disposable database/volume restore passes application-level tests.

Run a disposable restore drill quarterly and after material backup or storage changes.
Alert on backup failure, missing daily objects, application restart/health failure,
destination reachability, and a restore drill older than 120 days.

The internal rclone S3 endpoint is reachable from the Dokploy manager only. Current
manual verification proves configuration, execution, remote object existence, and
continued application health; independent readback and disposable restore remain
separate required gates.