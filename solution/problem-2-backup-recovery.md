# Problem 2 — Backup and recovery

## Cause and risk

The current daily Cloud SQL backup with seven-day retention does not provide a 15-minute recovery point: between backups, nearly a day's database changes could be lost. With point-in-time recovery (PITR) disabled, there is no transaction-log recovery to a time just before the deletion. The bucket has no deleted-object recovery. Even if a backup exists, an untested restore may not meet the two-hour recovery time, and restoring the database alone could leave records pointing to missing files.

## Changes to make before an incident

| Area | Change |
|---|---|
| Cloud SQL | Enable PITR and retain transaction logs for at least seven days. Keep automated backups with at least seven-day retention. PITR provides fine-grained recovery; backups provide a separate restore source. Protect backup configuration and permissions from ordinary application/operator roles. |
| Cloud Storage | Enable soft delete with a seven-day retention period on the private bucket. This is the primary protection for accidental object deletion. Consider object versioning as well if overwrites must be recoverable; set lifecycle rules to control the cost of retained generations. |
| Auditability | Enable and retain Cloud Storage Data Access logs for object writes/deletes and Cloud SQL PostgreSQL audit logging sufficient to identify the destructive transaction and affected records. Restrict log access and keep logs outside the normal application write path. |
| Recovery readiness | Keep the restore target in the same region with private connectivity, compatible database settings, and enough quota/capacity. Ensure operators can restore Cloud SQL and undelete objects without using application credentials. Document how to pause writers and change the application's database endpoint through its existing configuration/Secret Manager path. |

PITR and soft delete are recovery controls, not protection against every regional outage or compromised privileged identity. Restrict who can change retention and backup settings, monitor changes, and use separate administrative access.

## Recovery runbook: deletion at 10:20, discovered at 10:35

The objective is a database state immediately before the first destructive transaction, and restoration of the corresponding deleted object generations. For the RTO, start the clock at the incident time: target service restoration by 12:20. The 10:35 discovery leaves 1 hour 45 minutes for recovery.

1. **Declare the incident and stop changes.** At 10:35, put the service in maintenance/read-only mode. Stop API writes, workers, scheduled jobs, and other bucket/database writers. Record incident times. Preserve the affected production database and bucket; do not overwrite, delete, or restore over production data during investigation.
2. **Establish the recovery point and impact.** Use database audit records and Cloud Storage Data Access logs to identify the first deletion time, affected rows, object names, and deleted generations. Choose a PITR timestamp immediately before that database transaction. Preserve a consistent snapshot/export of the post-incident production state for investigation and possible reconciliation of legitimate changes made after the target time.
3. **Restore Cloud SQL to a new instance.** PITR-restore to a separate, isolated instance in the same region, with compatible PostgreSQL version, flags, private IP/connectivity, and capacity. Do not restore over the source instance. Wait for the restore to finish and verify that the deleted survey rows exist at the chosen point and the application database is reachable privately.
4. **Recover only the affected bucket objects.** Use the bucket's soft-delete retention to restore the specific object generations identified in step 2 to their original names. If an object was overwritten rather than deleted, recover the required prior generation from versioning if enabled. Avoid a blanket restore that could overwrite newer valid objects. Keep the service in maintenance mode while doing this.
5. **Reconcile database and files.** Compare every restored row's object reference with the bucket. Verify object name, generation where recorded, size, and checksum/metadata against the database or known upload manifest. Confirm that each referenced object is readable and that no required objects are missing. Identify orphaned objects, but do not delete them during recovery. If valid user writes occurred after the PITR timestamp, reconcile and replay only verified non-destructive changes from the preserved source/audit trail; never replay the deletion.
6. **Reconnect and validate before reopening.** Point the application to the restored database using its controlled configuration/Secret Manager mechanism, then restart or roll out API and worker pods so new connections use the restored instance. Keep background writes paused. Run database integrity checks and application smoke tests for survey lookup, report generation, and file download; check application errors, database connectivity, and object access. Confirm the application is not still connected to the damaged source.
7. **Reopen gradually.** After checks pass, enable writes and workers, then remove maintenance mode. Monitor errors and verify new uploads and records are consistent. Retain the original source and recovery evidence until incident review and reconciliation are complete.

If the database and files cannot be reconciled, keep writes disabled and escalate rather than reopening on a known-inconsistent state.

## Consistency, limits, and proving the targets

The database PITR timestamp and bucket recovery are separate operations; a Cloud SQL restore does not roll Cloud Storage back. A database restore to just before 10:20 may contain references to objects deleted at 10:20, which is why deleted generations must be restored before reopening. Conversely, do not roll back or overwrite the entire bucket: valid objects written after 10:20 may not belong to the restored database state. Audit logs and a row-to-object reconciliation are needed to identify the correct objects and any valid post-target writes.

The present setup cannot meet the stated targets: daily backups alone can lose almost 24 hours of database changes, and there is no configured recovery for deleted objects. Even after enabling PITR and soft delete, a two-hour RTO is not guaranteed by configuration alone. Cloud SQL restore duration, object count/size, quota, permissions, and operator execution time all need rehearsal. Soft delete covers the requested seven-day deletion window only when the object remains within that retention period; it does not by itself recover overwritten content.

Prove the targets with a scheduled restore drill in an isolated environment using synthetic survey rows and files:

- Record a known-good database transaction and upload objects; capture their identifiers and checksums. Then simulate accidental row and object deletion.
- Restore Cloud SQL to a point immediately before the deletion and recover the corresponding bucket generations. Measure from incident start until the application is serving validated traffic; require **under two hours**.
- Compare the latest required pre-incident database transaction and required file set with the recovered state. Confirm no more than **15 minutes** of database changes are lost, and that deleted test files are recoverable throughout the **seven-day** window.
- Test missing-object, checksum-mismatch, and endpoint-switch failure paths; verify the service remains in maintenance mode when validation fails.
- Record timings, restore timestamps, checksums, and any manual steps. Repeat after material changes to database size, bucket volume, network, or recovery tooling.

These settings and drills are proposed; no cloud restore or live recovery test was performed for this submission.
