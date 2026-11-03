# Problem 2 — Backup and recovery

## Risk and preparation

Daily backups can leave nearly 24 hours of database changes at risk, exceeding the 15-minute RPO. PITR is disabled, deleted bucket objects have no configured recovery, and no restore has been timed.

- Enable Cloud SQL PITR and retain logs for at least seven days; keep automated backups with at least seven-day retention. Limit who can change recovery settings.
- Enable Cloud Storage soft delete with seven-day retention. Add object versioning if overwritten files must also be recovered; manage resulting storage costs.
- Retain Cloud SQL and Cloud Storage audit logs sufficient to identify the destructive transaction and affected object generations. Restrict log/retention administration.
- Document how to pause all writers, restore a private Cloud SQL instance in-region, recover selected objects, and update the database endpoint. Pre-check IAM, quotas, connectivity, and operator access.

PITR and soft delete address separate failure modes; neither alone ensures database/file consistency or protects against every regional outage or compromised administrator.

## Recovery runbook — deleted at 10:20, discovered at 10:35

Count the two-hour RTO from incident start: service should be restored by 12:20, leaving 1 hour 45 minutes after discovery.

1. **Contain:** declare the incident, enable maintenance/read-only mode, stop API/worker/scheduled writers, and preserve the source database and bucket.
2. **Identify:** use audit logs to find the first destructive transaction, affected rows and object generations. Select a PITR time just before it; preserve current state for valid later writes.
3. **Restore DB:** PITR-restore Cloud SQL to a **new isolated same-region instance** with compatible settings/private connectivity. Confirm the deleted rows exist.
4. **Restore files:** recover only affected soft-deleted objects. Use object versioning for overwrites if enabled; avoid blanket restores.
5. **Reconcile:** compare each restored DB reference to readable object names/generations and checksums. Replay only verified legitimate later writes. If any required file is missing or mismatched, remain in maintenance mode.
6. **Reconnect/test:** update the app's controlled DB endpoint, restart API/workers, keep writes paused, then run integrity and lookup/report/download smoke tests. Verify connectivity and that no pod uses the old DB.
7. **Reopen:** resume writers, remove maintenance mode, monitor consistency/errors, and retain source/evidence through review.

## Limits and target verification

Cloud SQL PITR does not roll Cloud Storage back. Restored rows may refer to files deleted at 10:20; restore those generations before reopening, but do not roll back the whole bucket because it may contain valid newer files.

The current setup cannot meet the targets: daily backups exceed the RPO and deleted files have no configured recovery. Even with PITR and soft delete, restore time depends on database size, object volume, permissions, quotas and operator steps. A two-hour RTO must be demonstrated, not assumed.

Run an isolated drill with synthetic rows/files: capture transaction IDs and checksums, simulate deletion, restore and reconnect, and time until validated service returns. Verify no more than 15 minutes of DB changes are lost, recovery completes within two hours, and deleted files are recoverable throughout seven days. Test missing/mismatched objects and failed endpoint switching; keep maintenance active on failure. Record timings and repeat after material changes.

These are proposed controls and drills; no cloud restore was performed.
