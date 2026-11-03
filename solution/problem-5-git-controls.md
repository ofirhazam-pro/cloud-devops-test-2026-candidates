# Problem 5 — Git controls

## Cause and risk

Direct pushes skip review/checks; deploying every push can ship an unreviewed defect. Lightweight Azure Repos and pipeline controls address this without another CI/CD system.

## Proposed workflow

1. Developers use short-lived `feature/*` or `fix/*` branches and PR into protected `main`; disallow routine direct and force pushes.
2. Set Azure Repos **Build validation**, **minimum one reviewer** (not author), and **comment resolution** policies. Reset votes on new changes; restrict bypass/direct-push permissions to audited maintainers.
3. PR CI runs lint, tests and migration checks; require success before merge. Main builds/publishes once; production deploy requires named maintainer approval and uses the recorded image digest.
4. **Urgent fix:** branch from the latest `main` (for example, `fix/urgent-api-error`), make the smallest safe change, and mark the PR urgent. Run the same required build validation and migration checks, get at least one reviewer, resolve comments, and merge through the PR. Then obtain the normal production approval, deploy the recorded image digest, and verify service health. Speed this up by paging the reviewer and approver; do not skip tests or deploy directly from the fix branch.
5. **Exceptional Azure DevOps outage:** do not treat urgency alone as a reason to bypass policy. If the platform outage itself blocks the normal process and the incident commander authorizes an emergency release, limit the bypass to named maintainers, record the reason/change/approver, and create a PR as soon as the platform returns. Run the required checks and review that PR to validate and audit the emergency change.

## Verification

- Confirm direct push is blocked and a PR with failing checks or unresolved comments cannot merge. Confirm only authorized approvers can release and production uses the validated digest.
- **Urgent-fix tabletop (no production change):** walk through a simulated critical API bug. The developer branches from `main` and opens an urgent PR; the team identifies the required build/migration checks, reviewer, comment resolution, production approver, and digest to deploy. Confirm the release is blocked if a required check fails, then describe how the team would verify health after an approved deployment. Separately simulate Azure DevOps being unavailable and confirm who may authorize a documented emergency bypass and who owns the follow-up PR/checks.

These policies and tests are proposed; no Azure DevOps Server project was configured for this submission.
