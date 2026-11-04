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
- The availability proposal targets one zone outage, not a regional outage. Three-zone placement requires eligible nodes and capacity in all three zones; size resources and recovery capacity using actual workload/data.
- Regional GKE capacity, Cloud SQL regional HA, PITR, and seven-day Cloud Storage soft delete improve availability/recovery but add cost and do not replace tested backups or incident drills.
- **Validation:** reviewed docs/snippets and checked links/whitespace locally. No cloud resources, live pipeline, or recovery drill were run; proposed tests are labeled.
- **Time spent:** Candidate to fill in actual time before submission.
- **AI disclosure:** AI tools, including Copilot, were used to assist with researching technical concepts, reviewing the proposed architecture, and improving the clarity of the documentation. The final solution was reviewed and validated by me.
- No secrets or real customer data are included.

## Official references

- [GKE regional clusters](https://cloud.google.com/kubernetes-engine/docs/concepts/regional-clusters) and [Cloud SQL high availability](https://cloud.google.com/sql/docs/postgres/high-availability)
- [Cloud SQL point-in-time recovery](https://cloud.google.com/sql/docs/postgres/backup-recovery/pitr) and [Cloud Storage soft delete](https://cloud.google.com/storage/docs/soft-delete)
- [Azure DevOps Server 2022 branch policies](https://learn.microsoft.com/en-us/azure/devops/repos/git/branch-policies?view=azure-devops-server) and [pipeline approvals](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals?view=azure-devops-server)
