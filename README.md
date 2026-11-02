# GCP Cloud DevOps Engineer exercise

A practical assignment: improve a small existing GCP service and explain how to operate it.

- [Candidate assignment](assignment/ASSIGNMENT.md) — scenario, five problems and submission requirements.
- [Problem 1: Service availability](solution/problem-1-availability.md) — safer Kubernetes releases and zone-outage resilience, with an API manifest example.
- [Problem 2: Backup and recovery](solution/problem-2-backup-recovery.md) — Cloud SQL point-in-time recovery, Cloud Storage deleted-object recovery, and a recovery runbook with proposed verification drills.
- [Problem 3: Linux pipeline agent disk pressure](solution/problem-3-agent-disk.md) — diagnostic commands, safe space recovery, and prevention for the self-hosted build agent.
- [Problem 4: Failed release](solution/problem-4-release.md) — Azure DevOps Server pipeline flow, immutable image tracking, approval, and migration-aware recovery.
- [Problem 5: Git controls](solution/problem-5-git-controls.md) — a lightweight PR, branch-policy, and production authorization workflow.
- [Corrected architecture](solution/architecture.md) — traffic/data paths, regional availability, and Azure DevOps agent connectivity.

## Assumptions and submission notes

- The existing VPN, HTTPS load balancer, private GKE API connectivity, workload identity, attached agent service account, and Secret Manager access continue to work as stated in the assignment. No development environment is added.
- The availability proposal targets a single-zone outage, not a regional outage. Placing one API replica in each of three zones requires eligible nodes and capacity in all three; resource sizes and recovery durations must be validated against actual workload and data volumes.
- Regional GKE capacity, Cloud SQL regional HA, PITR, and seven-day Cloud Storage soft delete improve availability/recovery but add cost and do not replace tested backups or incident drills.
- **Validation:** documentation and snippets were reviewed locally; `git diff --check` passed. No cloud resources, live pipeline, or recovery drill were run. All proposed cloud/pipeline tests are labeled as such.
- **Time spent:** Candidate to fill in actual time before submission.
- **AI disclosure:** Substantial AI assistance was used to draft and edit this solution. The candidate should review every recommendation, verify environment-specific details, and be prepared to explain the design and tradeoffs.
- No secrets or real customer data are included.

Seven calendar days, with a suggested effort of 6–8 hours. No GCP account, deployment or paid resources required. All systems, incidents and targets are fictional.
